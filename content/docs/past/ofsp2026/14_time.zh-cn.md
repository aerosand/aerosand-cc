---
uid: 20251112144059
title: 14_time
date: 2025-11-12
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
weight: 15
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

本文简单介绍 OpenFOAM 中的 Time 类，在此基础上讨论 `createTime.H` 这个基础头文件。

本文主要讨论

- [ ] 测试新脚本
- [ ] 理解 createTime.H 头文件
- [ ] 了解 Time 类
- [ ] 理解时间相关的成员方法
- [ ] 编译运行 time 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_14_time
cd ofsp_14_time
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

终端输入命令，测试初始求解器

```terminal {fileName="terminal"}
wmake
ofsp_14_time -case debug_case
```

测试初始求解器没有问题，可以进一步开发。此步骤以后不再赘述。

### 1.1. 脚本和说明

读者可以发现，前面项目使用的脚本没有通用性，每次都需要重新修改，十分繁琐。

我们将逐渐介绍一些更方便的脚本写法。

终端输入命令，新建脚本

```terminal {fileName="terminal"}
code caserun caseclean
```

脚本 caserun 负责应用编译成功之后，调试算例的运行，内容如下

```bash {fileName="caserun",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)                     # Get application path
appName="${appPath##*/}"                            # Get application name

blockMesh -case debug_case | tee debug_case/log.mesh
echo "Meshing done."

$appName -case debug_case | tee debug_case/log.run

```

其中，`$0` 表示返回当前脚本的绝对路径 `%xxx` 表示去除 `xxx` 字符，`/*` 表示 `/` 及之后的所有字符。`|| exit 1` 表示如果 `cd` 命令执行失败（返回非零值），则退出脚本并返回错误码1。

>[!tip]
>不用担心脚本语言，暂时只需要知道使用上述脚本即可，更深入的脚本语言会随着讨论深入而逐渐介绍。

脚本 caseclean 负责清理调试算例，还原到初始状态，内容如下

```bash {fileName="caseclean",linenos=table,linenostart=1}
#!/bin/sh
cd "${0%/*}" || exit 1                              # Run from this directory
#------------------------------------------------------------------------------

appPath=$(cd `dirname $0`; pwd)
appName="${casePath##*/}"

cd debug_case

rm -rf log.*
foamCleanTutorials
echo "Cleaning Done."

```

这两个脚本自动获取应用路径和应用名称，所以也适合后续讨论的其他应用使用，只需要复制使用即可。

终端输入命令，使脚本生效

```terminal {fileName="terminal"}
chmod +x caserun caseclean
```


## 2. createTime.H

API 页面 https://api.openfoam.com/2506/createTime_8H.html

我们可以通过终端查找该文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname createTime.H
```

使用 vscode 可以直接点击终端输出的文件路径打开该文件。

在 vscode 中可以通过 OFextension 直接跳转打开该文件，以后不再赘述。

代码具体为

```cpp {fileName="createTime.H",linenos=table,linenostart=1}
Foam::Info<< "Create time\n" << Foam::endl;
// 普通的输出语句

Foam::Time runTime(Foam::Time::controlDictName, args);
// 构造了一个 Foam::Time 类的 runTime 对象
// 构造基于 controlDictName, args
// args 需要提前创建，所以 setRootCase.H 需要前置

```

## 3. Time 类

类 Time 是 OpenFOAM 中的基础类，具有很多的成员数据和成员方法。我们大概挑几处代码作为切入点简单了解一下 Time 类。

API 页面 https://api.openfoam.com/2506/Time_8H.html

终端输入命令，查阅 Time 类的声明

```terminal {fileName="terminal"}
find $FOAM_SRC -iname Time.H
```

打开 Time.H ，内容如下

```cpp {fileName="Time.H",linenos=table,linenostart=1}
...
class Time
:
    public clock,
    public cpuTime,
    public TimePaths,
    public objectRegistry,
    public TimeState
    // Time类继承自更多的基类
{
public:
	...
private:
	...
protected:
	...
private:
	...
public:
	...
	// 基于参数的构造函数
	inline Time
	(
		const word& ctrlDictName, // word类型的引用
		const argList& args, // argList类型的引用
		const bool enableFunctionObjects = true,
		const bool enableLibs = true
	);
	// 找到这个内联函数，就是 createTime.H 第2行的方法原型
	// 第3个和第4个形参默认为 true，可以不用传入
	// 此构造函数对应了 createTime.H 中的构造情况
	
	...
	
	// Searching

	//- Return time instance (location) of \c directory containing
	//- the file \c name (eg, used in reading mesh data).
	//- When \c name is empty, searches for \c directory only.
	//- Does not search beyond \c stopInstance (if set) or \c constant.
	//
	//  If the instance cannot be found:
	//  - FatalError when readOpt is MUST_READ or READ_MODIFIED
	//  - return \c stopInstance (if set and reached)
	//  - return \c constant if constant_fallback is true
	//  - return an empty word if constant_fallback is false
	//  .
	word findInstance
	(
		//! The subdirectory (local) for the search
		const fileName& directory,
		//! The filename for the search. If empty, only search for directory
		const word& name = word::null,
		//! The search type : generally MUST_READ or READ_IF_PRESENT
		IOobjectOption::readOption rOpt = IOobjectOption::MUST_READ,
		//! The search stop instance
		const word& stopInstance = word::null,
		//! Return \c "constant" instead of \c "" if the search failed
		const bool constant_fallback = true
	) const;
	// 可以看到 Time 类中提供了查找机制
	// 从当前时间开始，沿着时间目录从最近时间到最早时间搜索，直到找到包含该目录的时间实例，
	// 或者到达设置的 stopInstance 
	// 如果还是找不到，且 constant_fallback 为 true，则检查 constant 目录
	// 如果仍然找不到，则根据读取选项报错，否则返回空字符串。
	// 该机制一般用于查找网格数据
	// 代码细节暂不深究
	
	...
	
	// Member Functions
		// Access
		// Time类的成员函数
		//- Return start time
		virtual dimensionedScalar startTime() const;

		//- Return end time
		virtual dimensionedScalar endTime() const;

		//- Return the stop control information
		virtual stopAtControls stopAt() const;
	...
	
	
...

```


>[!tip]
>再次啰嗦，对于现阶段来说，读者不宜深挖代码。只要做到心里有数，可以理解且不陌生排斥即可。

## 4. 时间信息

我们可以使用 Time 类的一些方法输出和使用时间信息。

主函数修改如下

```cpp {fileName="ofsp_14_time/ofsp_14_time.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    // #include "createTime.H"
    // 展开 createTime.H 内容
    Foam::Info<< "Create time\n" << Foam::endl;
    Foam::Time runTime(Foam::Time::controlDictName, args);
    // 创建了 runTime 对象
    // 该对象具有 Time 类的成员方法和属性


    Info<< "Case name   :   " << runTime.caseName() << endl;
    Info<< "Root path   :   " << runTime.rootPath() << endl;
    Info<< "Path        :   " << runTime.path() << endl;
    Info<< "Time path   :   " << runTime.timePath() << endl;
    Info<< "Constant path :   " << runTime.constantPath() << endl;
    Info<< "Constant    :   " << runTime.constant() << endl;
    Info<< "System path :   " << runTime.systemPath() << endl;
    Info<< "Controdict  :   " << runTime.controlDict() << endl;
    Info<< "Format      :   " << runTime.writeFormat() << endl;
    Info<< "Version     :   " << runTime.writeVersion() << endl;
    Info<< "Start time  :   " << runTime.startTime() << endl;
    Info<< "End time    :   " << runTime.endTime() << endl;
    Info<< "Time step   :   " << runTime.deltaT() << endl;

    Info<< "# set end time = 10" << endl;
    runTime.setEndTime(10);

    Info<< "# set deltaT = 1" << endl;
    runTime.setDeltaT(1);

    Info<< "Start time  :   " << runTime.startTime() << endl;
    Info<< "End time    :   " << runTime.endTime() << endl;
    Info<< "Time step   :   " << runTime.deltaT() << endl;


    while (runTime.loop())
    {
        Info<< "Time: " << runTime.timeName() << endl;
        runTime.write();
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    // runTime.printExecutionTime(Info);
    // 展开成员函数 printExecutionTime() 的主要内容
    Info<< "Execution time: " << runTime.elapsedCpuTime() << " s" << endl;
    Info<< "Clock time: " << runTime.elapsedClockTime() << " s" << endl;

    Info<< "End\n" << endl;

    return 0;
}
```

终端输入命令，编译运行此项目

```terminal {fileName="terminal"}
wclean
wmake
./caseclean
./caserun
```

终端输出信息如下

```terminal {fileName="terminal"}
Create time

Case name   :   "debug_case"
Root path   :   "/home/aerosand/01_project/ofsp/ofsp_14_time"
Path        :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case"
Time path   :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case/0"
Constant path :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case/constant"
Constant    :   constant
System path :   "/home/aerosand/01_project/ofsp/ofsp_14_time/debug_case/system"
Controdict  :   
{
    application     icoFoam;
    startFrom       startTime;
    startTime       0;
    stopAt          endTime;
    endTime         0.5;
    deltaT          0.005;
    writeControl    timeStep;
    writeInterval   20;
    purgeWrite      0;
    writeFormat     ascii;
    writePrecision  6;
    writeCompression off;
    timeFormat      general;
    timePrecision   6;
    runTimeModifiable true;
}

Format      :   ascii
Version     :   2.0
Start time  :   startTime [0 0 1 0 0 0 0] 0
End time    :   endTime [0 0 1 0 0 0 0] 0.5
Time step   :   deltaT [0 0 1 0 0 0 0] 0.005
# set end time = 10
# set deltaT = 1
Start time  :   startTime [0 0 1 0 0 0 0] 0
End time    :   endTime [0 0 1 0 0 0 0] 10
Time step   :   deltaT [0 0 1 0 0 0 0] 1
Time: 1
Time: 2
Time: 3
Time: 4
Time: 5
Time: 6
Time: 7
Time: 8
Time: 9
Time: 10

Execution time: 0 s
Clock time: 0 s
End
```

通过这个项目，可以看到 runTime 对象不仅仅能访问时间目录，还可以访问 system/, constant/ 等文件夹。

## 5. 小结

本文简单讨论了 Time 类，了解 Time 类成员方法具有的作用。

到目前为止，我们已经大体理解 OpenFOAM 项目中常见的 setRootCase.H 和 createTime.H 头文件，下一篇将讨论另一个必备头文件。

本文完成讨论

- [x] 测试新脚本
- [x] 理解 createTime.H 头文件
- [x] 了解 Time 类
- [x] 理解时间相关的成员方法
- [x] 编译运行 time 项目

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
