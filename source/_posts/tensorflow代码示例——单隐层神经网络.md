---
title: tensorflow代码示例——单隐层神经网络
date: 2019-07-13 12:03:10
tags:
    - tensorflow
---

```
#coding: utf-8
import tensorflow as tf
from numpy.random import RandomState

#每轮训练输入的数据量
batch_size = 8

'''
含有一个隐藏层的神经网络
输入层                           隐藏层                              输出层
                                 A1
                           
                  

(X1)
          
            
                                 A2                                   Y


(X2)

                                 A3
'''

#w1是输入层和隐藏层之前的全连接权重, 相应的w2是隐藏层和输出层之间的连接权重
#这里采用tensorflow内置的随机正态分布对连接权重进行初始化
w1 = tf.Variable(tf.random_normal([2, 3], stddev=1, seed=1))
w2 = tf.Variable(tf.random_normal([3, 1], stddev=1, seed=1))

#定义输入数据的占位
x = tf.placeholder(tf.float32, shape=(None, 2), name="x-input")
y_= tf.placeholder(tf.float32, shape=(None, 1), name="y-input")

#定义神经网络的前向传播过程
a = tf.matmul(x, w1)
y = tf.matmul(a, w2)

#采用sigmoid激活函数
y = tf.sigmoid(y)

#采用交叉墒损失函数(y 和 y_ 之前的差距，其中y是神经网络的预测值，y_是实际值)
#clip_by_value对变量的值做上下限处理，将其限制在某个范围内
val = y_ * tf.log(tf.clip_by_value(y, 1e-10, 1.0)) + (1 - y_) * tf.log(tf.clip_by_value(1-y, 1e-10, 1.0))
cross_entropy = -tf.reduce_mean(val)

#反向传播优化方法，除了AdamOptimizer之外，常用的还有GradientDescentOptimizer MomentumOptimizer
#0.001是学习率，学习的目标就是最小化损失函数
train_step = tf.train.AdamOptimizer(0.001).minimize(cross_entropy)

#接下来我们造一点训练数据
rdm = RandomState(1)
dataset_size = 128 #128 组训练数据
X = rdm.rand(dataset_size, 2) #随机出来的训练数据输入
Y = [[int(x1+x2<1)] for (x1,x2) in X] #训练数据期望的输出

#运行整个训练过程
with tf.Session() as sess:
    sess.run(tf.global_variables_initializer()) #初始化所有的变量

    #变量初始化之后，看下w1和w2的当前值
    print sess.run(w1)
    print sess.run(w2)

    #迭代5000次
    for i in range(5000):
        #每次采用batch_size组数据进行训练
        start = (i*batch_size) % dataset_size
        end = min(start + batch_size, dataset_size)

        sess.run(train_step, feed_dict={x: X[start:end], y_:Y[start:end]})

        #每100轮输出一下当前的训练结果：W1和W2和交叉墒
        if i % 100 == 0:
            total_entropy = sess.run(cross_entropy, feed_dict={x:X, y_:Y})
            print("after %d training step, cross entropy is %g" % (i, total_entropy))

    print sess.run(w1)
    print sess.run(w2)






```