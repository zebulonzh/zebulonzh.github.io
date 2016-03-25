---
layout: post
title: '文件名格式检查'
date: 2015-06-15 15:39
categories: Python
excerpt:
---

这个脚本现在用于上个项目中，负责对所有打包文件的命名检查，该脚本的主要特点如下：

* 接受两个命令行参数用于指定搜索路径和日志保存路径，可以方便的嵌入QA的脚本链中。
* 如果检查出指定目录（及其递归子目录）中有非法文件，会保存在以当前时间命名的log文件中，全部合格则不会产生该文件
* 代码比较硬，两段注释起来的部分是用来对指定后缀的文件进行检查，之前是这样写的，不过QA说直接检查所有文件就行，于是就雪藏了

``` python
# -*-coding: utf-8 -*-
#param[1]:the absolute path of the root to search
#param[2]:the absolute path of the log file to saved

import os, re, sys, time

def checkName(name):
	pattern = re.compile("\W")	#only numbers, charactors and underline
	match = pattern.search(name)
	if not match:
		return False;
	else:
		return True

assert len(sys.argv) >= 2, "PARAM ERROR: Need a root path"
assert len(sys.argv) >= 3, "PARAM ERROR: Need a output path"

print "\n================Scanning================\n"
logTime = time.strftime("%m-%d-%H-%M-%S", tiem.localtime(time.time()))

fileName = ""
if sys.argv[2][-1]!=os.sep:
	fileName = sys.argv[2] + os.sep + logTime + ".txt"
else:
	fileName = sys.argv[2] + logTime + ".txt"

logFile = open(fileName, "w")
if not logFile:
	print "Log File can't create!"
	os.system("pause")

num = 0
totalNum = 0

'''
extList = [".lua", ".ccbi", ".png", ".plist"]
'''
for dirPath, dirNames, fileNames in os.walk (sys.argv[1]):
	for f in fileNames:
		ext = os.path.splitext(f)
		totalNum += 1
		if checkName(f[:len(ext[0])]):
			num += 1
			logFile.write(os.path.join(dirPath, f) + "\n")
			print os.path.join(dirPath, f)
		'''
		for suffix in extList:
			in ext[1] = suffix:
			totalNum += 1
			if checkName(f[:len(ext[0])]):
				num += 1
				logFile.write(os.path.join(dirPath,f) +"\n")
				print os.path.join(dirPath,f)
		'''
logFile.close()

print "\n================Finished================\n"
print "%d" % totalNum + "files checked"
print "%d" % num + "error(s)"
if num == 0:
	os.remove(fileName)
else:
	print "Logdata is saved in " + fileName
```