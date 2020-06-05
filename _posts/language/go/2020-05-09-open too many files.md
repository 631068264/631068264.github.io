---
layout:     post
rewards: false
title:      记一次 go 程序的Too many open files
categories:
    - go
---

# 背景

程序打开的**文件**/**socket连接数量**超过系统设定值。


# 用到的命令  

`ulimit -a` 查看每个用户最大允许打开文件数量

```

core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 20
file size               (blocks, -f) unlimited
pending signals                 (-i) 16382
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) unlimited
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

其中 open files (-n) 1024 表示每个用户最大允许打开的文件数量是1024




`lsof -p pid | wc -l` 某一进程的打开文件数量



`lsof -p pid` 某一进程的打开文件/socket


# TCP 状态解释

![](https://tva1.sinaimg.cn/large/006tKfTcgy1g1kl4m77guj31860meq3z.jpg)
![](https://tva3.sinaimg.cn/large/006tKfTcgy1g1kl4x2guzj314s0mwdh6.jpg)

对于已经建立的连接，网络双方要进行四次握手才能成功断开连接，如果缺少了其中某个步骤，
将会使连接处于假死状态，连接本身占用的资源不 会被释放。
网络服务器程序要同时管理大量连接，所以很有必要保证无用连接完全断开，否则大量僵死的连接会浪费许多服务器资源。

排错时注意端口ip定位对应服务


# 程序问题

主要是client问题

## grpc client

数据量一大rpc并发多之前写的时候没有注意，参照[**官网列子**](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_client/main.go)用的是rpc短连接

由于没有复用connection导致每次都有频繁读去证书文件出现

```go
func NewClient() (client *Client, err error) {
	conf := config.Get()

	client = &Client{}
	conn, err := getTLSConn(conf.RPCClient.RocksDB.Cert, conf.RPCClient.RocksDB.Address)
	if err != nil {
		return
	}
	c := protodef.NewxxxxClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)

	client.conn = conn
	client.client = c
	client.ctx = ctx
	client.cancel = cancel

	return
}


func (client *Client) xxx(dbName, key string) (data []byte, err error) {
	reply, err := client.client.xxx(client.ctx, &protodef.xxxRequest{
	    。。。。。
	})
	defer client.ConnClose()

    。。。。。

	return
}


func (client *Client) ConnClose() {
	client.cancel()
	client.conn.Close()
}

func getTLSConn(certPath string, rpcAddress string) (conn *grpc.ClientConn, err error) {
	cert, err := ioutil.ReadFile(certPath)
	if err != nil {
		return
	}
	cp := x509.NewCertPool()
	if !cp.AppendCertsFromPEM(cert) {
		return nil, fmt.Errorf("credentials fail")
	}

	creds := credentials.NewTLS(&tls.Config{RootCAs: cp, InsecureSkipVerify: true})
	conn, err = grpc.Dial(rpcAddress, grpc.WithTransportCredentials(creds))
	if err != nil {
		return
	}

	return
}
```

之前只是以为是证书的关系

```go
var certContent []byte

func getTLSConn(certPath string, rpcAddress string) (conn *grpc.ClientConn, err error) {
	if len(certContent) == 0 {
		certContent, err = ioutil.ReadFile(certPath)
		if err != nil {
			return
		}
	}
	cp := x509.NewCertPool()
	if !cp.AppendCertsFromPEM(certContent) {
		return nil, fmt.Errorf("credentials fail")
	}

	creds := credentials.NewTLS(&tls.Config{RootCAs: cp, InsecureSkipVerify: true})
	conn, err = grpc.Dial(rpcAddress, grpc.WithTransportCredentials(creds))
	if err != nil {
		return
	}

	return
}
```

然后就转为报 **grpc transport: Error while dialing dial tcp 127.0.0.1:9101: socket: too many open files**

- [simply maintain a single connection and attach multiple client stubs to it dynamically](https://github.com/grpc/grpc-go/issues/682)

看到有人说到这个才发现要复用**connection**,让**connection**创建多个client, 让程序初始化时先调用**NewConn**创建**connection**

```go

func NewConn() (conn *grpc.ClientConn, err error) {
	conf := config.Get()

	conn, err = getTLSConn(conf.RPCClient.RocksDB.Cert, conf.RPCClient.RocksDB.Address)
	if err != nil {
		return
	}
	return
}

func NewClient() (client *Client, err error) {
	c := protodef.NewYiZhiSecTrafficClient(RPCConn)

	client = &Client{}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	client.client = c
	client.ctx = ctx
	client.cancel = cancel

	return
}

func (client *Client) ConnClose() {
	client.cancel()
}

```

## http client

因为有较多grpc连接所以改好后，以为已经解决问题，运行一段时间后看下log，又有一大堆**Create error open : xxxx : too many open files**


因为是上传文件的rpc接口，看报错以为是打开了文件没有关，然后打log发现文件确实关了。这就奇怪了。通过`lsof -p pid`发现报错的时候出现大量TCP连接


最终发现该接口还会去调另一个HTTP api，**然后http client是需要close的**

```go

client := http.Client{}
_, err = client.Post(xx, "application/json", bytes.NewReader(data))
return err
```

改成

```go

result, err := http.Post(xxx, "application/json", bytes.NewReader(data))
if err != nil {
    return
}
defer result.Body.Close()


```