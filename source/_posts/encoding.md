---
title: 论pyqt 编码的蹊跷——QTextStream、QString、string、unicode相关 
date: 2017-01-14 14:00:00 
categories:
tags:
     - 编码
---
#### 环境：Pyqt4.8 32位，python2.7.3 32位
你是不是每次看到什么字符编码、文件编码和字符串类型，都会有些懵逼呢？反正我有点，以前情况不复杂，这次遇到个坑，特此记录下。
* 背景：
从**qrc**文件中读取某文本文件，然后解析成json，并显示在Qt控件上。该文件以**utf-8**编码，并保存有**中文**，对，就是这个中文的引出的话题,不是中文也就不复杂了……  
* 分析：
	1. 从rcc编译的qrc文件中读取文件，也就意味着无法使用python的标准代码：
	```python
	with open(file_path) as f:
		content = json.load(f)
		print content
	```
	别无选择，只能使用QFile。
	2. 说到QFile，自然要用到QTextStream了。
	3. 再使用python unicode()函数将str对象解码。
	4. 最后使用json库loads()方法，解析成json对象。
	基本代码是这样的：  
	```python
	def read_file(path):
		try:
			f = QtCore.QFile(path)
			if not f.open(QtCore.QFile.ReadOnly | QtCore.QFile.Text):
				return ""
			ts = QtCore.QTextStream(f)
			tsData = ts.readAll()
			content = unicode(tsData, "utf-8", "ignore")
			return json.loads(content)
		except:
			import traceback
			traceback.print_exc()
		finally:
			f.close()
	```
	最后发现报错了……
	ValueError: Invalid control character at: line 14 column
	5. 尝试将tsData先转换为utf-8编码，结果还是报错……
	还尝试着直接使用str()等等方法，包括网上的一些技巧，比如：json.loads(content, strict=False)，都失败了……

* 解决及总结：
	1. QTextStream在读取文本文件时，会默认使用Local的字符编码，如果不指定编码，会使后续的处理寸步难行……
	后续有个官方链接说明了
	For Python v2 the following conversions are done by default.
	If Qt expects a char *, signed char * or an unsigned char * (or a const version) then PyQt4 will accept a unicode or QString that contains only ASCII characters, a str, a QByteArray, or a Python object that implements the buffer protocol.
	If Qt expects a char, signed char or an unsigned char (or a const version) then PyQt4 will accept the same types as for char *, signed char * and unsigned char * and also require that a single character is provided.
	If Qt expects a QString then PyQt4 will accept a unicode, a str that contains only ASCII characters, a QChar or a QByteArray.
	If Qt expects a QByteArray then PyQt4 will accept a unicode that contains only Latin-1 characters, or a str
	2. Unicode()在不指定encoding参数的情况下，有两种操作。如果字符串是str对象，则会调用str()，也就是使用python默认的ascci编码来解码。如果已经是Unicode对象则不会任何附加操作。
	If no optional parameters are given, unicode() will mimic the behaviour of str() except that it returns Unicode strings instead of 8-bit strings. More precisely, if object is a Unicode string or subclass it will return that Unicode string without any additional decoding applied.
	所以在这里，我们需要指定utf-8的编码格式，才能转化为unicode对象。
	最后代码如下：
	```python
	@contextmanager
	def read_file(path):
		try:
			f = QtCore.QFile(path)
			if f.open(QtCore.QFile.ReadOnly | QtCore.QFile.Text):
				ts = QtCore.QTextStream(f)
				ts.setCodec("utf-8")
				tsData = ts.readAll()
				content = unicode(tsData.toUtf8(), "utf-8", "ignore")
				yield json.loads(content)
			else:
				yield ""
		except:
			import traceback
			traceback.print_exc()
			yield ""
		finally:
			f.close()
	```

有个需要注意的地方，如果要gui控件能正常显示中文，`content = unicode(tsData.toUtf8(), "utf-8", "ignore")`中的**toUtf8()**是必不可少，不然会显示为乱码。 

* 总结：
一句话总结：**区分什么是编码，什么是对象，Unicode是中转对象，str->unicode是解码，unicode->str是编码。**
不知在谁的blog上看到的了，很形象，很深刻……谢谢这样仁兄！

***
相关链接：
[PyQt 4.12 Reference Guide](http://pyqt.sourceforge.net/Docs/PyQt4/gotchas.html)
