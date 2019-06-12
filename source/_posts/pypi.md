---
title: 代码库打包上传到PyPI服务器
tags:
 - python
 - 打包
 - PyPI 
abstract: 本文介绍如何将python代码打包发布到PyPI仓库，简单陈述了打包及上传的过程。
posturl: python-pypi
date: 2018-10-02 10:39:36
---
`说明:本文写于2016年，当时没有搭建博客站点，也没有发布，写这个文档主要用于公司内部传阅。水平有限，如有错误，欢迎批评指正！`

## 热身知识
### 初识PyPI
PyPI这四个字母很熟悉，但是在实践发布代码包之前，我并不知道它的全称及含义，所以提及一下。引用[官网](https://pypi.python.org/pypi "PyPI官网")的解释：PyPI - the Python Package Index. The Python Package Index is a repository of software for the Python programming language.

PyPI其实是一个python语言的仓库,不仅仅是可以上传Package, 也可以上传Module。不过Package和Module在用起来是有区别的，也就是说上传的是Package，安装之后，import时就是Package。 Module亦然。下一节说说Package和Module的区别。

PyPI跟gitlab差不多，有官方的，有民间的，有私有的。一般情况下，公共抽象模块跟业务没有关系的代码可以上传到共有PyPI服务器，无私奉献的工程师是可敬的。如果跟业务绑得比较紧，有保密的必要，或者要提高安装和上传的速度等情况，可以考虑搭建PyPI私有源服务器，搭建过程也很简单，并且有很多轻巧的开源解决方案，比如[pypiserver](https://pypi.python.org/pypi/pypiserver "pypiserver源")。平时，我们说的修改安装源，就是修改PyPI的服务器地址。当然，即使是使用PyPI私有源服务器上传自己的代码，私有服务器上也只有一些内部代码，更多的第三方库需要在公共服务器上下载安装，这种情况可以不修改PyPI的地址，而是在使用pip安装特定的包的时候通过参数`-i`指定当前安装源地址!命令如下： 

```bash
pip install -i https://testpypi.python.org/pypi my_pack
```

### Package 和 Module
在python中，一个py文件是一个Module，文件名字就是Module的名字(__name__)。使用时,直接`import module_name` 。

Package是一个文件夹，里面包含一个`__init__.py`文件,以及包需要的Module文件，也可以包含SubPackage的文件夹。`__init__.py`文件中最好`import`一些Module，方便使用。比如Package名字是my_pack，对应目录下的`__init__.py`内容如下：
```python
import module_a
import module_b
import module_c
#  __all__ 限制了from my_pack import * 时，将不会import module_c
__all__ = ['module_a', 'module_b']
```
引用Package中的Module方法如下
```python
#  方法一
from my_pack import module_a
module_a.fun()

#  方法二
#  该是需要在__init__中import对应的module
import my_pack
my_pack.module_a.fun()
```
这里写个`__init__.py`的示例及两种方法是因为该文件的写法对module的引用方法不同。也是为后面PyPI打包的`setup.py`文件中选择以Package的方式还是Module的方式，以及如何修改Package的`__init__.py`作参考。

## 准备实践
考虑到一般没有必要搭建PyPI私有源服务器，这里只讲[PyPI官方服务器](https://pypi.python.org/pypi)上传代码的方法与流程。其实无论什么PyPI，核心方法是一样的，只是注册流程及服务器地址不同。
### 注册PyPI账号
先到[官网](https://pypi.python.org/pypi?%3Aaction=register_form)注册一个账号，然后邮箱验证一下。
### 配置用户文件
在用户目录下创建`.pypirc`文件，写入用户信息,如下
```bash
[distutils]
index-servers=
    pypi
    testpypi

[pypi]
repository = https://pypi.python.org/pypi
username = username
password = password

[testpypi]
repository = https://testpypi.python.org/pypi
username = username
password = password
```
这个文件里可以写入多个PyPI服务器的地址、用户名、密码，如上文件写入了两个服务器地址，其中一个是官方提供的测试服。上传代码的时候再指定上传到哪个服务器。
### 整理代码
这一步就是根据的需要，将需要上传的Package的`__init__.py`文件写好，方便安装后的使用。
### 写`setup.py`文件
`setup.py`文件是上传代码包、或者本地直接安装都可以用的，也是比较重要的文件。
下面通过例子来解释。
```python
# -*- coding:utf-8 -*-

from setuptools import setup, find_packages

setup(
      name='feiwu',
      version='0.0.2',
      description="some useful module on flask developer",
      keywords='web flask',
      author='chen',
      author_email='chenyehai737@qq.com',
      license = 'MIT',
      url='',
      packages=['feiwu'],
      package_data={
        'feiwu': ['ipcool', 'provinces.json'],
      },
      zip_safe=False,
      classifiers=[
        'Programming Language :: Python :: 2.6',
        'Programming Language :: Python :: 2.7',
      ],
      install_requires=[
        'Flask==0.10.1',
        'requests==2.7.0',
        'ujson==1.33',
        'qiniu==7.0.4',
      ],
      entry_points={
      },
)

```
如果缺少setuptools，可以自行安装。
下面说明一下`setup.py`文件中重要参数的作用。

1. name是上传的代码库名字，不能与服务器中既有名字重复，也是后续使用`pip install name`进行安装时的名字。
2. version 是代码库的版本，每次上传的版本要不一样。当使用0.1上传了一次，代码修改之后，version的值也要改变，再上传。
3. url是该代码维护的地址，可以为空。
4. packages参数是需要打包的Package，可以通过列表显式列出Package名字，逗号分隔，也可以使用setuptools中的find_packages()函数，找出符合条件的package。不过，我一般只打包一个Package，其他Package作为它的SubPackage，这样安装以后引用方便。
5. package_data是一个字典，列出各个Packages需要捆绑打包的其他数据文件(默认只会打包Package中的py文件)，可以通过正则表达式来写,比如`*.txt`。这个玩法很灵活，不一一列举。[文档中](https://setuptools.readthedocs.io/en/latest/setuptools.html#including-data-files)可以看到其他用法。
6. zip_safe参数接受bool类型，默认值为False。如果文件非常大，赋值True，安装时可以从zip文件直接运行。一般可以忽略这个参数，或者设置为False。具体看[文档](https://setuptools.readthedocs.io/en/latest/setuptools.html#setting-the-zip-safe-flag)。
7. classifiers用于说明代码包可以在什么环境下使用，应该只是说明性的，对整个打包过程没有影响，也可以不写。一般会写属于的语言、框架、操作系统等。
8. install_requires是一个列表，里面填写依赖的库，在安装这个包之前，pip会先安装列表中指明的依赖库，如果缺少依赖库，可能会导致安装之后，还需要手动安装依赖库。
9. entry_points好像很高端，我没看明白，所以例子中也没有写。希望高人指点一下！
10. 要上传单独Module的话，也是可以的，使用py_modules参数。


### 测试
目录结构可能是这样的。
```
├── setup.py
├── feiwu
│   ├── cfgtools.py
│   ├── flask_helper.py
│   ├── flask_logging.py
│   ├── __init__.py
│   ├── ipcool
│   ├── json_utils.py
│   ├── provinces.json
│   ├── readip.py
│   ├── README.txt
│   ├── utils.py
```
feiwu是一个Package，里面包含了它的Module和数据文件ipcool。
到了这一步，一般先尝试本地直接安装，也就是先不上传，不通过pip安装。安装命令为`python setup.py install`。
执行完毕之后，在python命令行中`import feiwu`测试一下。如果没有报错，再`dir(feiwu)`看看是否包含了你想要的Package和Module，如果没有，可能是`setup.py`的packages参数中没有填对，也可能是Package中的`__init__.py`文件没有写对。总的来说，这一步测试的结果跟发布到PyPI服务器上再pip安装的结果是一样的。
需要注意的是，测试的时候需要切换到其他目录。因为python首先会从当前目录去找需要import的文件，即使没有正确安装你的代码库或者安装出错，import的都是当前目录下的，不能保证已经成功安装。

### 准备上传
测试完毕之后，就可以准备上传了。在这一步中，可能会因为之前的配置或者网络等因素而报错，根据提示解决就可以了。
1. 先检查用户信息配置是否正确、`setup.py`文件是否正确、库的名字在服务器中是否重复了(自己维护的，版本更新不算重复)等。执行命令`python setup.py register -r pypi`（命令中-r参数后面带的是`~/.pypirc`文件中的服务器名，也可以直接写网址）看输出信息。至少要看到回复200，其他的警告信息，看看是否需要解决吧。此时会生成名为`name.egg-info`的文件夹,示例中是`feiwu.egg-info`。文件夹中是该代码库的一些信息文件，有兴趣的可以查阅。此时目录结构如下
```
.
├── feiwu
│   ├── cfgtools.py
│   ├── cfgtools.pyc
│   ├── flask_helper.py
│   ├── flask_helper.pyc
│   ├── flask_logging.py
│   ├── flask_logging.pyc
│   ├── __init__.py
│   ├── __init__.pyc
│   ├── ipcool
│   ├── json_utils.py
│   ├── json_utils.pyc
│   ├── provinces.json
│   ├── readip.py
│   ├── readip.pyc
│   ├── README.txt
│   ├── utils.py
│   └── utils.pyc
├── feiwu.egg-info
│   ├── dependency_links.txt
│   ├── not-zip-safe
│   ├── PKG-INFO
│   ├── requires.txt
│   ├── SOURCES.txt
│   └── top_level.txt
└── setup.py
```
2. 创建代码库的文件，并压缩。使用命令是`python setup.py sdist`。这个过程中会创建一个文件夹`name-version`,示例中是`feiwu-0.0.2`。然后将需要的py文件及数据文件拷贝进去，然后压缩移动到dist文件夹，并移除文件夹`name-version`。此命令的执行具体过程可以通过输出信息查看。此时目录结构如下:
```
├── dist
│   └── feiwu-0.0.2.tar.gz
├── feiwu
│   ├── cfgtools.py
│   ├── cfgtools.pyc
│   ├── flask_helper.py
│   ├── flask_helper.pyc
│   ├── flask_logging.py
│   ├── flask_logging.pyc
│   ├── __init__.py
│   ├── __init__.pyc
│   ├── ipcool
│   ├── json_utils.py
│   ├── json_utils.pyc
│   ├── provinces.json
│   ├── readip.py
│   ├── readip.pyc
│   ├── README.txt
│   ├── utils.py
│   └── utils.pyc
├── feiwu.egg-info
│   ├── dependency_links.txt
│   ├── not-zip-safe
│   ├── PKG-INFO
│   ├── requires.txt
│   ├── SOURCES.txt
│   └── top_level.txt
└── setup.py
```
3. 上传代码库。执行命令`twine upload dist/* -r pypi`进行上传。如果没有twine，可以通过pip安装。上传完毕，没有报错就可以换个虚拟环境安装试试了。
4. 以上几个命令可以写进shell文件里面。
```shell
python setup.py register -r pypi
python setup.py sdist
twine upload dist/* -r pypi
```

## 小结
总的来说，过程还是很简单，主要是`__init__.py`文件写好，然后`setup.py`写好就可以了。不过`setup.py`还有很多高端玩法，欢迎大家探讨、交流。