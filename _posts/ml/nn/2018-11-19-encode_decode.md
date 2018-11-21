---
layout:     post
rewards: false
title:      AE VAE GAE
categories:
    - ml
---

# AutoEncoder
AE

- **encode** input x, and encodes it using a transformation h to encoded signal y

$$y = h(x)$$

- **decode**

$$r = f(y) = f(h(x))$$

weights 可以share, encode decode的weights 可以简单地反过来用
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fxd77qop3fj30t80hkdfu.jpg)
损失函数的不一样，hidden layer不一样 => 不同的AE



# Variational Autoencoder
变分自编码器(VAE)

根据很多个样本，学会生成新的样本

学会数据 x 的分布 P (x)，这样，根据数据的分布就能轻松地产生新样本但数据分布的估计不是件容易的事情，尤其是当数据量不足的时候。

VAE对于任意d维随机变量，不管他们实际上服从什么分布，我总可以用d个服从标准高斯分布的随机变量通过一个足够复杂的函数去逼近它。

- AE 是还原x
- VAE是产生新的x
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fxfofnw3maj30wu0k8dgd.jpg)
训练完后VAE，使用decode来生成新的X


>KL 散度(Kullback–Leibler Divergence) 来衡量两个分布的差异，或者说距离

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fxd82llvovj31j80dcweq.jpg)
[cross-entry 与 KL散度等效 对于损失函数](https://stackoverflow.com/questions/41863814/kl-divergence-in-tensorflow)

```python
import numpy as np

def KL(P,Q):
""" Epsilon is used here to avoid conditional code for
checking that neither P nor Q is equal to 0. """
     epsilon = 0.00001

     # You may want to instead make copies to avoid changing the np arrays.
     P = P+epsilon
     Q = Q+epsilon

     divergence = np.sum(P*np.log(P/Q))
     return divergence

```

# Generative Adversarial Network
GAN 生成式对抗网络

同时训练两个神经网络
- 生成器(Generator):记作 G，通过对大量样本的学习，输入一些随机噪声能够生成一些以假乱真的样本
- 判别器(Discriminator):记作D，接受真实样本和G生成的样本，并进行判别和区分
- G 和 D 相互博弈，通过学习，G 的生成能力和 D 的判别能力都逐渐 增强并收敛,交替地训练，避免一方过强

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fxdgtf0tnvj30f406hjri.jpg)

核心代码
```python
with tf.variable_scope('gan'):
    G = generator(noise)
    real, r_logits = discriminator(X, reuse=False)
    # 注意共享变量
    fake, f_logits = discriminator(G, reuse=True)
```

核心Loss
```python

# 计算logits, labels距离
def sigmod_cost(logits, labels):
    return tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=logits, labels=labels))

with tf.variable_scope('loss'):
    """
    接近1为real 接近0为fake
    目标：
        G 制造以假乱真的
        D 轻易区分真假
    就是 G 和 D 之间的对抗
    """
    g_loss = sigmod_cost(logits=f_logits, labels=tf.ones_like(f_logits))
    tf.summary.scalar('g_loss', g_loss)

    r_loss = sigmod_cost(logits=r_logits, labels=tf.ones_like(r_logits))
    f_loss = sigmod_cost(logits=f_logits, labels=tf.zeros_like(f_logits))
    d_loss = r_loss + f_loss
    tf.summary.scalar('d_loss', d_loss)
```

Train 同时训练
```python
tvar = tf.trainable_variables()
dvar = [var for var in tvar if 'discriminator' in var.name]
gvar = [var for var in tvar if 'generator' in var.name]

d_opt = tf.train.AdamOptimizer(learning_rate=learning_rate, beta1=beta1).minimize(d_loss, var_list=dvar)
g_opt = tf.train.AdamOptimizer(learning_rate=learning_rate, beta1=beta1).minimize(g_loss, var_list=gvar)
```

使用[CNN](https://www.oreilly.com/learning/generative-adversarial-networks-for-beginners?imm_mid=0f3eba&cmp=em-data-na-na-newsltr_20170628)的话没有GPU加速就别搞了，用CPU慢,超级慢。要非常注意`shape`

用普通FC训练 G生成出来的结果
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fxfimk5r0jg30hy0hukjp.gif)
```python
def discriminator_fc(X, reuse=None):
    """判别器"""
    with tf.variable_scope('discriminator', reuse=reuse):
        l1 = tf.layers.dense(X, 128, activation=tf.nn.relu, )
        logits = tf.layers.dense(l1, 1, )
        prob = tf.nn.sigmoid(logits)
        return prob, logits

def generator_fc(X):
    """生成器"""
    with tf.variable_scope('generator'):
        l1 = tf.layers.dense(X, 128, activation=tf.nn.relu, )
        prob = tf.layers.dense(l1, 784, activation=tf.nn.sigmoid)
        return prob

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    writer = tf.summary.FileWriter('gan_mnist/{}_{}'.format(mode, now), sess.graph)

    for i in range(epoch):
        batch_X, _ = mnist.train.next_batch(batch_size)
        batch_noise = np.random.uniform(-1., 1., [batch_size, z_dim])

        _, d_loss_print = sess.run([d_opt, d_loss], feed_dict={X: batch_X, noise: batch_noise, is_train: True})
        _, g_loss_print = sess.run([g_opt, g_loss], feed_dict={X: batch_X, noise: batch_noise, is_train: True})

        if i % 100 == 0:
            summary = sess.run(merged_summary_op, feed_dict={X: batch_X, noise: batch_noise, is_train: True})
            writer.add_summary(summary, i)
            print('epoch:%d g_loss:%f d_loss:%f' % (i, g_loss_print, d_loss_print))

        if i % 500 == 0:
            batch_test_noise = np.random.uniform(-1., 1., [batch_size, 100])
            samples = sess.run(G, feed_dict={noise: batch_test_noise, is_train: False})
            fig = plot(samples)
            plt.savefig('{}/{}.png'.format(output_path, str(num_img).zfill(3)), bbox_inches='tight')
            num_img += 1
            plt.close(fig)

    to_gif(output_path, output_gif='{}_{}output'.format(mode, now))
```