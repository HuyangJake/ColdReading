---
title: 在iOS中运行Python
date: 2018-09-03 12:27:12
tags: [iOS]
---

在iOS设备中运行python脚本？那不就意味着可以在手机上跑爬虫，可以使用各种牛逼哄哄的python库了吗。

这个标题对我很有吸引力，曾经就有见到过在iOS平台上的python编译器(很多iOS上python的IDE，如[Python3IDE](https://itunes.apple.com/cn/app/python3ide/id1357215444?mt=8))，可以执行输入的python语和本地的python文件。

当然我想要的不是像[这篇文章](https://segmentfault.com/a/1190000004945692)说的用python编写一整个iOS程序，而只是在iOS应用中嵌入python文件执行非UI的逻辑，也就是说只需要在项目中嵌入一个python的编译环境。

面对市场上这么多的iOS版python编译器，首先可以确定的是，针对iOS端的python编译库是存在的。我关心的问题是，能否支持python项目化的编译，能否导入丰富的三方库。那就动手一试咯

<!--more-->

---

### 小目标
七牛的iOS平台SDK有这样一个特点，上传文件的时候需要生成token，但是生成token的逻辑在客户端的SDK中不存在，只能通过调用服务端的SDK才能获得。
于是我们的小目标就诞生了：在iOS端上调用七牛python服务端SDK来生成token给客户端的七牛SDK使用。

---

### iOS的Python解释器
针对iOS、MacOS平台，[pybee](https://pybee.org/)开源了python支持库[Python-Apple-support](https://github.com/pybee/Python-Apple-support)，这个是老版本的库[Python-iOS-support](https://github.com/pybee/Python-iOS-support).



#### 准备编译环境
我这次使用的是老库中的[Python3.4.2-b5](https://github.com/pybee/Python-iOS-support/releases/download/3.4.2-b5/Python-3.4.2-iOS-support.b5.tar.gz)版本，下载下来有两个framework，分别是`OpenSSL`和`Python`。将这个两个framework拖入项目中，添加必要的lib库如图：
![](http://ojam5z7vg.bkt.clouddn.com/15359053297417.jpg)

在项目中创建`PythonEnvironment.bundle`将`Python.framework`中的`Restources`文件夹内容复制进去，在初始化Python环境之前将bundle中的文件复制到指定目录作为Home路径

#### 设置Home路径、初始化

设置上面准备home路径，并初始化编译环境。

``` objectivec
const char * frameworkPath = [[NSString stringWithFormat:@"%@/Resources",[self p_pythonFrameworkPath]] UTF8String];
wchar_t  *pythonHome = _Py_char2wchar(frameworkPath, NULL);
Py_SetPythonHome(pythonHome);

Py_Initialize();
PyEval_InitThreads();

//在释放的时候调用
Py_Finalize();
```

#### 执行Python代码、文件
编译环境设置好之后，使用`PyRun_SimpleString(python_code)`便可以简单执行Python代码

``` objectivec
PyRun_SimpleString("print('hello world')");
```
便可以输出`hello world`

##### 执行Python文件

``` objectivec
NSString *scriptPath = [[NSBundle mainBundle] pathForResource:@"test" ofType:@"py"];
    
FILE *mainFile = fopen([scriptPath UTF8String], "r");
   
PyRun_SimpleFile(mainFile, (char *)[[scriptPath lastPathComponent] UTF8String]);
```

上面是执行main bundle中的Python文件方式，这种方式暂时没有找到如何调用文件中的某各类具体方法和传参。

另外一种方式可以做到上面描述的需求，将在下一节中说明

---

### 准备七牛Python库
下载好的七牛SDK文件源码解压，在Xcode中创建一个bundle加入项目中，bundle中放七牛SDK的核心文件，如图：
![](http://ojam5z7vg.bkt.clouddn.com/15359397140219.jpg)

在需要使用七牛SDK之前，将此bundle中的文件拷贝到Python运行环境的home目录下

#### 编写token生成Python文件
查看七牛的文档了解到生成token需要用到`auth.py`这个文件中的`Auth`类， 我们需要想办法创建一个`Auth`实例并传入需要的参数，再将生成的token导出来。

首先自己创建一个`iostoken.py`文件，Python的文件名和方法名需要小写，类名需要大写。在`iostoken.py`中创建`TokenForiOS`类

``` python
import json
from qiniu import Auth

class TokenForiOS(object):
    
    def create_token(jsonParams):
        print(str(jsonParams))
        values = json.loads(jsonParams)
        access_key = values.get('access_key')
        secret_key = values.get('secret_key')
        #要上传的空间
        bucket_name = values.get('bucket_name')
        #上传到七牛后保存的文件名
        file_name = values.get('file_name')
        #构建鉴权对象
        q = Auth(access_key, secret_key)
        #生成上传 Token，可以指定过期时间等
        token = q.upload_token(bucket_name, file_name, 3600)
        return token
```

上面是我使用七牛SDK中的Auth生成token的代码，类名为`TokenForiOS`方法名为`create_token`，现在需要找到合适的地方调用。

不过在想要使用`TokenForiOS`类之前，需要将其加入`qiniu`模块的初始化`__init__.py`中：

``` python
from .iostoken import TokenForiOS
```

接下来就可以愉快地调用了

`Python.framework`中有一套宏可以导入Python模块，生成实例，传参调用方法，具体使用例子见下代码块


``` objectivec

PyObject *pModule = PyImport_ImportModule([@"qiniu.iostoken" UTF8String]);//导入模块
    
PyObject *pyClass = PyObject_GetAttrString(pModule, [@"TokenForiOS" UTF8String]);//获取类
    
PyObject *pyInstance = PyInstanceMethod_New(pyClass); //创建实例
    
NSMutableDictionary *params = [NSMutableDictionary new];
[params setObject:@"123" forKey:@"access_key"];
[params setObject:@"456" forKey:@"secret_key"];
[params setObject:@"jake" forKey:@"bucket_name"];
[params setObject:@"pic" forKey:@"file_name"];
    
NSError *error = nil;
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:params options:NSJSONWritingPrettyPrinted error:&error];
NSString *paramterJsonString = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
    
PyObject *result = NULL;
result = PyObject_CallMethod(pyInstance, [@"create_token" UTF8String], "(s)", [paramterJsonString UTF8String] );

char * resultCString = NULL;
PyArg_Parse(result, "s", &resultCString); //将python类型的返回值转换为c
    
NSLog(@"%s", resultCString);
```

至此，我的小目标就完成啦。后面还有很多事情可以做，比如跑个爬虫试试，毕竟还没有尝试过网络请求。当然也还有很多坑要踩。

---


### 总结

使用过程中有下面几点体会
* `Python-Apple-support`库的缺少文档，在设置Home目录导入模块的过程中踩了很多坑
* 编译器对Python语法提示并不支持，难以排查写错的地方
* framework的体积是在过大，对项目总体积影响大

---

PS:在查询过程中发现了[PyObjc](https://pythonhosted.org/pyobjc/index.html)这个项目，能够使用python通过bridge调用OC的方法从而使用Cocoa框架，实现使用python编写Cocoa GUI应用。
>PyObjC is a bridge between Python and Objective-C. It allows Python scripts to use and extend existing Objective-C class libraries; most importantly the Cocoa libraries by Apple.

>This document describes how to use Objective-C class libraries from Python scripts and how to interpret the documentation of those libraries from the point of view of a Python programmer.

---
### Reference

[在iOS app中运行Python文件（Swift+Objective C+Python) 带Demo](https://blog.csdn.net/haojinming/article/details/77816403) 

[iOS 工程中调用Python方法（带Demo）](https://www.jianshu.com/p/80b5be51fb1d)

[Embedding Python in an iPhone app](https://stackoverflow.com/questions/3691655/embedding-python-in-an-iphone-app)

[PyObjc](https://pythonhosted.org/pyobjc/index.html)