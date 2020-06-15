---
title: RNN-based GAN 基础模型及相关论文
tags:
  - Fall Detection
  - Machine Learning
  - Deep Learning
categories: Machine Learning
abbrlink: 28656
date: 2019-05-31 02:22:33
---

## GAN基础模型

![](/images/fig9.jpg)

替换成CGAN可以提升模型普适性


## 判别器/分类器基础模型

![](/images/fig7.png)

## 判别器残差结构

![](/images/fig10.png)
生成器是否需加入残差结构仍待考虑


## 生成器基础模型

![](/images/fig8.png)

生成器与判别器的正则化层、激活函数、以及训练的Loss函数均需细化讨论如何添加

## 相关论文
1.GAN开山之作：

>[Generative Adversarial Nets](http://papers.nips.cc/paper/5423-generative-adversarial-nets.pdf)

2.WGAN：引入Wasserstein距离，既解决了训练不稳定的问题，也提供了一个可靠的训练进程指标，而且该指标确实与生成样本的质量高度相关。

>[Wasserstein Generative Adversarial Networks](http://proceedings.mlr.press/v70/arjovsky17a/arjovsky17a.pdf)

>[Improved Training of Wasserstein GANs](http://papers.nips.cc/paper/7159-improved-training-of-wasserstein-gans.pdf) (WGAN-GP)

3.GAN优化、收敛相关：

>[Gradient descent GAN optimization is locally stable](https://arxiv.org/abs/1706.04156)

>[Which Training Methods for GANs do actually Converge?](https://www.semanticscholar.org/paper/Which-Training-Methods-for-GANs-do-actually-Mescheder-Geiger/1b741fd8c2ebff0200556cf2690dfc9fd79dcba5)

4.提出了一种叫做 “谱归一化”（spectral normalization）的新的权重归一化（weight normalization）技术，来稳定判别器的训练。

>[Spectral Normalization for Generative Adversarial Networks](https://openreview.net/pdf?id=B1QRgziT-)

5.展示了通过使用模拟数据对机器学习模型进行训练可以泛化得到原始数据：

>[Privacy-preserving generative deep neural networks support clinical data sharing](https://www.biorxiv.org/content/10.1101/159756v1)

6.使用RGAN、RCGAN生成时间序列医疗数据

>[Real-valued (Medical) Time Series Generation with Recurrent Conditional GANs](https://arxiv.org/pdf/1706.02633.pdf)

7.使用BiLSTM-CNN生成标记心电图数据
>[Electrocardiogram generation with a bidirectional LSTM-CNN generative adversarial network](https://www.nature.com/articles/s41598-019-42516-z.pdf)

8.使用GAN增强语音识别鲁棒性

>[Generative Adversarial Networks Based Data Augmentation for Noise Robust Speech Recognition](https://speechlab.sjtu.edu.cn/papers/hh112-hu-icassp18.pdf)

9.GAN数据增强
>[Data Augmentation Generative Adversarial Networks](https://arxiv.org/pdf/1711.04340.pdf)

>[一种基于生成式对抗神经网络的数据增强方法](http://www.joca.cn/CN/10.11772/j.Issn.1001-9081.2018051008)

>[BAGAN: Data Augmentation with Balancing GAN](https://arxiv.org/pdf/1803.09655.pdf)

>[DeLiGAN : Generative Adversarial Networks for Diverse and Limited Data](https://arxiv.org/pdf/1706.02071.pdf)

>[GAN-based Synthetic Medical Image Augmentation
for increased CNN Performance in Liver Lesion Classification](https://arxiv.org/pdf/1803.01229.pdf)

