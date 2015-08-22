---
layout: post
title: 'cocos ide踩坑记录'
date: 2015-07-17 10:33
categories: IDE
excerpt:
---

* content
{:toc}

## 0.写在前面

目前ide有两个版本，1.2.0的eclipse版和2.0BETA的IDEA版，前者虽然bug百出，性能很差，但是基本功能还算齐全，后者性能有很大提升，但是由于是BETA版，所以功能少很多，尤其是lua方面，代码提示和打包都没有，我们项目使用lua，所以只好选择前者，但愿IDEA能发展的好一点吧。

## 1.版本适配    

首先，项目需要对c++部分做修改，所以无法使用framework，这里我就不吐槽cocos的下载bug了。
其次，1.2release版的ide其实是和v3.4版的c2d-x engine适配的，这一点我没有看到任何官方的提示或者说明，只能自己一个版本一个版本测试。如果使用v3.5及以上的engine，ide的工程目录会找不到对应版本的api。

## 2.建立工程

由于使用的是engine，所以通过cocos new建立工程，之后在ide中导入工程，其工程文件是隐藏的，选中目录即可。

## 3.编译运行

* engine的空项目第一次编译运行时，会提示你没有模拟器，而framework是已经自带了的，我使用的是macbook，所以构建了mac的模拟器，但是运行的时候，出现error

	`
	cocos2d: fullPathForFilename: No file found at config.json. Possible missing file.
	`

这个问题是源码lua部分的模板有bug，在"cocos/cocos2d/Cocos2dConstants.lua"最下面，有个叫做cc.AsyncTackPool.TaskType的table,在这个表之前给他初始化一下或者整个注释起来都可以。

* 继续运行的时候时，会遇到这个error
    `Unable to load nib file: MainMenu`
    这个是模拟器的问题，之前构建的模拟器不能用，需要用Xcode打开其runtime，然后把mac版选为target，clean，build&run，跑起来之后，模拟器的文件夹会出现新的模拟器，这个模拟器是Xcode生成的。然后检查一下ide中对于模拟器的选择了路径是否正确，不出意外的话，这次应该可以在ide中跑起来了


