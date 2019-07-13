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



###
2019-07-13 12:05:25.725879: I tensorflow/core/platform/cpu_feature_guard.cc:141] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA

[[-0.8113182   1.4845988   0.06532937]
 [-2.4427042   0.0992484   0.5912243 ]]
[[-0.8113182 ]
 [ 1.4845988 ]
 [ 0.06532937]]
after 0 training step, cross entropy is 1.89805
after 100 training step, cross entropy is 1.62943
after 200 training step, cross entropy is 1.40099
after 300 training step, cross entropy is 1.19732
after 400 training step, cross entropy is 1.02375
after 500 training step, cross entropy is 0.887612
after 600 training step, cross entropy is 0.790222
after 700 training step, cross entropy is 0.727325
after 800 training step, cross entropy is 0.689437
after 900 training step, cross entropy is 0.667623
after 1000 training step, cross entropy is 0.655075
after 1100 training step, cross entropy is 0.647813
after 1200 training step, cross entropy is 0.643196
after 1300 training step, cross entropy is 0.639896
after 1400 training step, cross entropy is 0.637246
after 1500 training step, cross entropy is 0.635031
after 1600 training step, cross entropy is 0.633027
after 1700 training step, cross entropy is 0.631151
after 1800 training step, cross entropy is 0.629368
after 1900 training step, cross entropy is 0.627724
after 2000 training step, cross entropy is 0.626172
after 2100 training step, cross entropy is 0.624696
after 2200 training step, cross entropy is 0.623293
after 2300 training step, cross entropy is 0.622006
after 2400 training step, cross entropy is 0.620801
after 2500 training step, cross entropy is 0.619664
after 2600 training step, cross entropy is 0.618592
after 2700 training step, cross entropy is 0.617622
after 2800 training step, cross entropy is 0.616723
after 2900 training step, cross entropy is 0.615883
after 3000 training step, cross entropy is 0.615096
after 3100 training step, cross entropy is 0.614397
after 3200 training step, cross entropy is 0.613756
after 3300 training step, cross entropy is 0.61316
after 3400 training step, cross entropy is 0.612608
after 3500 training step, cross entropy is 0.612126
after 3600 training step, cross entropy is 0.611689
after 3700 training step, cross entropy is 0.611285
after 3800 training step, cross entropy is 0.610913
after 3900 training step, cross entropy is 0.610594
after 4000 training step, cross entropy is 0.610309
after 4100 training step, cross entropy is 0.610046
after 4200 training step, cross entropy is 0.609804
after 4300 training step, cross entropy is 0.609603
after 4400 training step, cross entropy is 0.609423
after 4500 training step, cross entropy is 0.609258
after 4600 training step, cross entropy is 0.609106
after 4700 training step, cross entropy is 0.608983
after 4800 training step, cross entropy is 0.608874
after 4900 training step, cross entropy is 0.608772
[[ 0.02476983  0.56948674  1.6921941 ]
 [-2.1977348  -0.23668921  1.1143895 ]]
[[-0.45544702]
 [ 0.49110925]
 [-0.98110336]]



```