---
layout:     post
rewards: false
title:      anti防盗链
categories:
    - spider
tags:
    - spider
---

# 图片防盗链

图片爬虫

## 加 meata
html head 加meta
`<meta name="referrer" content="never">`

## img
img['referrerPolicy'] = 'no-referrer'

```python
# 图片处理
    imgs = tag.find_all('img')
    if imgs:
        for img in imgs:
            if img['data-src']:
                # img['src'] = anti_referer(img['data-src'])
                img['src'] = img['data-src'].split('?')[0]
                img['referrerPolicy'] = 'no-referrer'
```

## base64

适合图不大 不多

```python
def anti_referer(url):
    b64_src = 'data:image/{content_type};base64,'
    content_type_re = re.search(r'mmbiz.qpic.cn/mmbiz_(?P<wx_fmt>.*?)/', url)
    try:
        content_type = content_type_re.group('wx_fmt')
        src = b64_src.format(content_type=content_type)
        resp = requests.get(url)
        content = resp.content
        b64 = base64.b64encode(content).decode()
        src += b64
        return src
    except:
        logger.get('error-log').error('[{}]\n'.format(url) + util.error_msg())
        return url

```

## 第三方代理

国外首选[第三方代理](https://images.weserv.nl/)<span class='heimu'>有几率被墙</span>
`https://images.weserv.nl/?url=`


## nginx

没试过 <span class='heimu'>觉得麻烦 上面的轻松</span>
[nginx 解决微信文章图片防盗链](https://www.jianshu.com/p/0511cda4e459)