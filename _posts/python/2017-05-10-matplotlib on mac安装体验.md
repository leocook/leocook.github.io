---
layout: post
comments: true
date: 2017-05-10
categories: python
tags: matplotlib
---

* content
{:toc}

Matplotlib是一个基于Python的2D绘图库，使用起来方便、快捷，图片质量较高，本文将简单的做一些介绍，并安装体验。




官方安装说明：http://matplotlib.org/users/installing.html
在Mac OSX上，可以使用Pip来安装。

## 关于Pip
Pip是python的一个包管理工具，下面说说它的安装。

- 下载安装脚本

```
curl -O https://bootstrap.pypa.io/get-pip.py
```

- 安装

```
#如果你使用Python 2.7,运行下面的命令：
python get-pip.py

#如果你使用Python 3,运行下面的命令：
python3 get-pip.py
```

安装是不是很简单？

## 安装matplotlib
直接命令安装：
```
pip install matplotlib
```

## Example

关于matplotlib的例子，在<code>http://matplotlib.org/examples/index.html</code>这里有很多，下面随机选了两个运行一下作为例子让大家看下。

- animate_decay

```
python animate_decay.py
```

![animate_decay](http://7xriy2.com1.z0.glb.clouddn.com/animate_decay.gif "animate_decay")

- dynamic_image

```
python dynamic_image.py
```

![dynamic_image](http://7xriy2.com1.z0.glb.clouddn.com/dynamic_image.gif "dynamic_image")


