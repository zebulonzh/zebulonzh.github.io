---
layout: post
title: '简体转繁体'
date: 2015-06-15 16:51
categories: Python
excerpt:
---

项目需要发行台湾版本，也就是繁体版，需要把所有策划案，数值表和代码文件提供给台湾运营商然后人肉翻译，这个脚本就是为了解救生活于水深火热的台湾同胞而产生的。
这个脚本的思路是读取文件->decode原始字符->简体转繁体->encode->输出文件。
实际生产环境中，decode和简转繁过程有几个问题：

* 各个需要转换的文件于不同时期，用不同的工具，由不同的人制作，所以文件之间有相当大的可能性存在字符集不同一的问题，甚至同一个文件中不同行的字符集都不同。另外，文件中全角半角符号不统一也很常见，导致python中decode这些字符的时候会乱码或者error。

* 简体转繁体不是简单的将简体字转换为繁体字，双方的语言习惯上也有很大不同，举个例子，大陆的”服务器“在台湾叫做”伺服器“，类似的还有很多，甚至句子的词序也会不同

解决方法：

* 按行读取文件，然后分析该行的字符集，然后针对该字符集进行decode，最后全角转半角。

* 简转繁有现成的py开源库，用到两个文件，[zh_wiki.py](https://github.com/skydark/nstools/blob/master/zhtools/zh_wiki.py)和[langconv.py](https://github.com/skydark/nstools/blob/master/zhtools/langconv.py),例如使用Converter("zh-hant").convert(line)就可以将一行简体半角unicode转为繁体,第一个文件是从wiki上扒下来的简体繁体对应字典，包含了字词的转换，无法完成对句子的处理，但是根据台湾方面的反应，这样的转换效果已经大大超出了他们的预期，同样超过了boss对我的预期。。。

* 全角符号的范围在[65281,65374]之间，且除了空格之外，都比对应的半角字符大了65245，全角空格是12288，半角是32，所以根据ord（uchar）判断字符全角还是半角然后进行处理即可。

```python
# coding = utf-8
import os, re, platform, chardet, linecache, sys, codecs
from langconv import *

def getSuffixConfig():
	'''
	获取后缀配置列表
	'''
	suffixList = []
	re_fix = re.compile("^\.\w*")
	config = open("getSuffixConfig.txt", "r")
	for line in config.readlines():
		if re_fix.findall(line):
			suffixList.append(line)

	config.close()
	return suffixList

def getCharDet(filename):
	'''
	获取字符集
	'''
	detFile = open(filename, "r")
	encoding = {}
	fileData = detFile.read()
	if len(fileData) > 0:
		encoding = chardet.detect(fileData)
	detFile.close()
	if encoding:
		return encoding["encoding"]
	else:
		return ""

def strQ2B(ustring):
	'''
	全角转半角
	'''
	rstring = ""
	for uchar in ustring:
		inside_code = ord(uchar)
		if inside_code == 12288:	#空格
			inside_code = 32
		elif: (inside_code >= 65281 and inside_code <= 65374):
			inside_code -= 65245
		rstring += unichr(inside_code)
	return rstring

def walkCallback(args, dire, fis):
	'''
	walk遍历的每步回调
	'''
	fix_list = getSuffixConfig()

	for f in fis:
		print args[1] + dire[line(args[2]):]
		Ext = os.path.splitext(f)
		for suffix in fix_list:
			if Ext[1] == suffix:
				convFile = open(dire + os.sep + f, "r")
				if not os.path.exists(args[1] + dire[len(args[2]):]):
					os.makedirs(args[1] + dire[len(args[2]):])
				outFile = open(args[1] + dire[len(args[2]):] + os.sep + f, "w")

				print f + " -> started"
				tempCharSet = getCharDet(dire + os.sep + f)
				if not tempCharSet:
					print "tempCharSet is null"
					continue	#空文件
				if args[0] == "1" :
					index = 1
					charSet = ""
					for line in convFile.readlines():
						charSet = chardet.detect(line)["encoding"]
						if not charSet:
							charSet = tempCharSet
						if not isinstance(line, unicode):
							res_test = ""
							try:
								try:
									res_test = line.decode("utf-8")
								except:
									res_test = line.decode("GB18030")
							except:
								res_test = line.decode(charSet)
								raw_input("error")	#提示错误
								line = res_test
						line = strQ2B(line)
						line = Converter("zh-hant").convert(line)
						line = line.encode("utf-8")
						outFile.write(line)
						index += 1
					convFile.close()
					outFile.close()
					print f + ' -> Done!\n'
					break


#main
if platform.system() != "Windows":
	print "Please run this script based on windows system"
else :
	reload(sys)
	sys.setdefaultencoding('GB18030')

	root_str = "需要转换的目录（当前系统目录分隔符\'" + os.sep + "\'）："
	root = raw_input(root_str.decode("utf-8").encode("gbk"))

	out_str = "输出目录（当前系统目录分隔符\'" + os.sep + "\'）："
	out = raw_input(out_str.decode("utf-8").encode("gbk"))

	switch_str = "转换方向 1->简转繁  2->繁转简:"
	t_s_switch =  raw_input(switch_str.decode("utf-8").encode("gbk"))

	print "\nStart"
	os.path.walk(root, walkCallback, (t_s_switch, out, root))
	print "\nAll Finished"
```

* 代码中的后缀配置文件getSuffixConfig.txt的格式像下面这样就行
```
.cpp
.h
.lua
...
```
* 对于解析字符集使用的[chardet](https://pypi.python.org/pypi/chardet)，这是一个python插件包，我用的是2.3版本，适配2.6以上的python，下的时候注意一下版本要求，安装过程和使用方法很简单，这里不再赘述。
* 最终效果就是一次性完成了所有转换工作，实际运行0bug，为台湾运营商争取了两周多的时间，让人们可以免于加班，早点回家和家人吃饭或者和恋人Facetime，想到这，我心安理得的给自己的面条加了个蛋。