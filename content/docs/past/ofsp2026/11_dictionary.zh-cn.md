---
uid: 20250916203141
title: 11_dictionary
date: 2025-09-16
update: 2026-02-04
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp
  - ofsp2026
  - OpenFOAM
excludeSearch: false
toc: true
weight: 12
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---
> [!important]
> 访问 https://aerosand.cn 以获取最近更新。


## 0. 前言

前面的讨论让我们了解了 OpenFOAM 写入写出和字典的本质。下面，我们看一看 OpenFOAM 为我们提供的写入写出方法。

OpenFOAM 的应用一般需要从 case 中读取字典，向 case 中输出计算结果等等。

OpenFOAM 是怎么实现从文件夹读取和写入的呢？OpenFOAM 的读取和写入更加高级，按关键词进行索引查找的方法直接封装在了相关的类中，直接使用方法即可。我们暂时不用深究到实现的代码层面。

本文主要讨论

- [ ] 了解 OpenFOAM 的不同数据格式的写入写出
- [ ] 了解 OpenFOAM 字典类提供的方法
- [ ] 编译运行 dictionary 项目

## 1. 项目准备

终端输入命令，建立本文项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_11_dictionary
code ofsp_11_dictionary
```

终端输入命令，为项目准备测试算例

```terminal {fileName="terminal"}
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
```

终端输入命令，测试初始求解器

```terminal {fileName="terminal"}
wmake
ofsp_11_dictionary -case debug_case
```

终端输出如下，

```terminal {fileName="terminal"}
Create time


ExecutionTime = 0 s  ClockTime = 0 s

End
```

上面的输出信息即说明初始求解器没有问题，可以在此基础上进行开发。

## 2. 说明文件

作为一个较为完整的 OpenFOAM 项目，我们为其提供说明文件

```markdown {fileName="/README.md"}
## About

这是一个用来理解OpenFOAM字典的项目。

## Bio

- Aerosand @ Aerosand

## Caution

需要使用 OpenFOAM v2406 及更新版本。

## Deploy

准备好环境和所有的文件。

在根目录下执行终端命令

清理并重新编译应用

1. wclean
2. wmake
   
清理并重新计算测试算例

1. ./caseclean
2. ./caserun

## Event

@ 202509010
- 增加清理脚本 

```

## 3. 脚本文件

脚本和之前讨论的项目类似，修改脚本内的求解器名称即可。

脚本 `caserun` 主要是负责应用编译成功后，调试算例的运行，暂时写入如下内容

```bash {fileName="/caserun",linenos=table,linenostart=1}
#!/bin/bash

blockMesh -case debug_case | tee debug_case/log.mesh
echo "Meshing done."

ofsp_11_dictionary -case debug_case | tee debug_case/log.run
```

脚本 `caseclean` 主要是负责清理应用到到编译前状态，如果应用要修改，那么测试算例也要还原到运行前的状态，所以暂时写入如下内容

```bash {fileName="/caseclean",linenos=table,linenostart=1}
#!/bin/bash

rm -rf debug_case/log.*
foamCleanTutorials debug_case
echo "Cleaning done."
```

终端输入命令，给脚本权限

```terminal {fileName="terminal"}
chmod +x caserun caseclean
```


## 4. 文件结构

文件结构如下

```terminal {fileName="terminal"}
.
├── caseclean
├── caserun
├── debug_case
│   ├── 0
│   │   ├── p
│   │   └── U
│   ├── constant
│   │   └── transportProperties
│   └── system
│       ├── blockMeshDict
│       ├── controlDict
│       ├── decomposeParDict
│       ├── fvSchemes
│       ├── fvSolution
│       └── PDRblockMeshDict
├── Make
│   ├── files
│   └── options
├── ofsp_11_dictionary.C
└── README.md
```

## 5. 主源码

主源码 `ofsp_11_dictionary.C` 内容如下

```cpp {fileName="/ofsp_11_dictionary.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"


    /* 
    * About dictionary customProperties
    */
    const word dictName("customProperties"); // 创建word类型对象，用以保存字典名称
    IOobject dictIO // 构造 IOobject 类型对象
    (
        dictName, // 字典名称
        runTime.constant(), // 字典位置
        mesh, // 和mesh相关
        IOobject::MUST_READ, // 必须从字典读取
        IOobject::NO_WRITE // 不向字典写入
    );

    if (!dictIO.typeHeaderOk<dictionary>(true))
    {
        FatalErrorIn(args.executable()) << "Cannot open specified dictionary"
            << dictName << exit(FatalError);
    }
    // 如果字典文件在文件头中指定的不是 dictionary 类型，则报错

    /* 
    * About dictionary myProperties
    */

    dictionary myDictionary;
    myDictionary = IOdictionary(dictIO);
    // 基于IOobject实体创建字典对象

    // 一般使用下面这种紧凑写法
    Info<< "Reading myProperties\n" << endl; 
    IOdictionary myProperties // 字典变量名和字典文件名取相同
    (
        IOobject
        (
            "myProperties",
            runTime.constant(),
            mesh,
            IOobject::MUST_READ,
            IOobject::NO_WRITE
        )
    );

    word solver; // 创建word类型变量
    myProperties.lookup("application") >> solver;
    // 从myProperties文件中查找到关键词，并取值赋给solver

    word format(myProperties.lookup("writeFormat"));
    // 或者写成更紧凑的形式

    scalar timeStep(myProperties.lookupOrDefault("deltaT", scalar(0.01)));
    // 也可以写成
    // scalar timeStep(myProperties.lookupOrDefault<scalar>("deltaT", 0.01));
    // 如果字典中没有提供这一关键词，则使用此句提供的默认值

    dimensionedScalar alpha("alpha",dimless,myProperties);
    // 常用这种语法读取字典中的参数
    
    dimensionedScalar beta(myProperties.lookup("beta"));
    // 这种写法在老代码中常见，但是在新一些的版本中，编译虽然能过，但是提醒语法过时


    bool ifPurgeWrite(myProperties.lookupOrDefault<Switch>("purgeWrite",0));
    // bool类型也可以按关键词查找并读取

    List<scalar> pointList(myProperties.lookup("point"));
    // 列表也可以读取

    HashTable<vector,word> sourceField(myProperties.lookup("source"));
    // 哈希表也可以读取

	vector myVec = vector(myProperties.subDict("subDict").lookup("myVec"));
	// 也可以在字典文件中再使用子字典
	// 注意后文字典文件中的子字典写法

	// 输出读取的内容
    Info<< nl
        << "application: " << solver << nl << nl
        << "writeFormat: " << format << nl << nl
        << "deltaT: " << timeStep << nl << nl
        << "alpha: " << alpha << nl << nl
        << "beta: " << beta << nl << nl
        << "purgeWrite: " << ifPurgeWrite << nl << nl
        << "point: " << pointList << nl << nl
        << "source: " << sourceField << nl << nl
	    << "myVec: " << myVec << nl << nl
        << endl;

    /* 
    * Write to files
    */
    fileName outputDir = runTime.path()/"processing"; // 创建outputDir变量并赋值路径
    mkDir(outputDir); // 创建上句路径的文件夹

    autoPtr<OFstream> outputFilePtr; // 输出文件流的指针
    outputFilePtr.reset(new OFstream(outputDir/"myOutPut.dat")); // 给指针定向

	// 通过指针给输出文件写入信息
    outputFilePtr() << "processing/myOutPut.dat" << endl;
    outputFilePtr() << "0 1 2 3 ..." << endl;
    sourceField.insert("U3", vector(1, 0.0, 0.0));
    outputFilePtr() << sourceField << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

## 6. 提供字典

提供字典文件 `debug_case/constant/customProperties`，该字典没有读取写入操作，所以只需要写上正确的文件头，内容留空处理。

```cpp {fileName="/debug_case/constant/customProperties",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "constant";
    object      customProperties;
}

// 留空

```

字典文件 `debug_case/constant/myProperties` ，内容如下

```cpp {fileName="/debug_case/constant/myProperties",linenos=table,linenostart=1}
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "constant";
    object      myProperties;
}

application     icoFoam;

writeFormat     ascii;

purgeWrite      1;

alpha           0.2;
beta            beta [0 0 0 0 0 0 0]  0.5;

point
(
    0
    1
    2
);

source
(
    U1 (0 0 0)
    U2 (1 0 0)
);

subDict
{
    myVec (0.0 0.0 1.0);
}

```

## 7. 编译运行

终端输入命令，编译运行

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

终端输出内容如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Reading myProperties


application: icoFoam

writeFormat: ascii

deltaT: 0.01

alpha: alpha [0 0 0 0 0 0 0] 0.2

beta: beta [0 0 0 0 0 0 0] 0.5

purgeWrite: 1

point: 3(0 1 2)

source: 
2
(
U1 (0 0 0)
U2 (1 0 0)
)

myVec: (0 0 1)



ExecutionTime = 0 s  ClockTime = 0 s

End

```

另外算例文件夹下有了一个新建文件夹 `debug_case/processing/`，路径下的 `myOutPut.dat` 内容如下

```cpp {fileName="/debug_case/processing/myOutPut.dat",linenos=table,linenostart=1}
processing/myOutPut.dat
0 1 2 3 ...

3
(
U1 (0 0 0)
U3 (1 0 0)
U2 (1 0 0)
)

```

## 8. 小结

本项目讨论了 OpenFOAM 设计的文件流写入写出方法，以后在实践中会不断地使用字典以及字典相关的方法。

本文完成讨论

- [x] 了解 OpenFOAM 的不同数据格式的写入写出
- [x] 了解 OpenFOAM 字典类提供的方法
- [x] 编译运行 dictionary 项目

## 支持我们

>[!tip]
>希望这里的分享可以对坚持、热爱又勇敢的您有所帮助。 
>
>如果这里的分享对您有帮助，您的评论、转发和赞助将对本系列以及后续其他系列的更新、勘误、迭代和完善都有很大的意义，这些行动也会为后来的新同学的学习有很大的助益。 
>
>赞助打赏时的信息和留言将用于展示和感谢。

{{< cards >}}
  {{< card link="/" title="支持我们" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="支付宝AliPay" >}}
{{< /cards >}}
