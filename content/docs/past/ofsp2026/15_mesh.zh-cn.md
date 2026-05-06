---
uid: 20251112184331
title: 15_mesh
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
weight: 16
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

除了 setRootCase.H 和 createTime.H，还有一个必然会出现的头文件 createMesh.H。

本文主要讨论

- [ ] 了解和网格相关的类
- [ ] 理解 createMesh.H 头文件
- [ ] 练习网格相关的方法
- [ ] 编译运行 mesh 项目

## 1. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
foamNewApp ofsp_15_mesh
cd ofsp_15_mesh
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity debug_case
code .
```

测试初始求解器，提供脚本和说明。

## 2. mesh

我们先不看 createMesh.H 的源代码，先从应用的需求切入，也就是，我们需要基于具体的 case 建立的计算用的 mesh ，这个 mesh 当然也有自己的数据（包括 point, face 等等），也要有自己的方法（包括返回点列，返回面心等等）。

### 2.1. primitiveMesh

API 页面 https://api.openfoam.com/2506/classFoam_1_1primitiveMesh.html

OpenFOAM 提供了基于几何要素的 primitiveMesh 类。

终端输入命令，查阅 primitiveMesh 类的声明

```terminal {fileName="terminal"}
find $FOAM_SRC -iname primitiveMesh.H
```

可以看到该 primitiveMesh 类并不继承其他类

我们大概挑几处代码作为切入点简单了解一下 primitiveMesh 类。

```cpp {fileName="primitiveMesh.H",linenos=table,linenostart=1}
class primitiveMesh
{
...
    // Constructors

        //- Construct from components
        primitiveMesh
        (
            const label nPoints,
            const label nInternalFaces,
            const label nFaces,
            const label nCells
        );
    ...
    
    // Access
    
	    // Primitive mesh data

			//- Return mesh points		
			virtual const pointField& points() const = 0;
...
```

通过这个构造汉书可以看到 primitiveMesh 类从基本的几何要素，点、面、单元构造对象。

几何要素，点、面、单元由 blockMesh 生成，或者由第三方网格软件生成再转换成 OpenFOAM 格式。无论如何，点、面、单元被生成在 case/constant/polyMesh/ 文件夹下。这里的几何数据只是数据而已，不具备任何的方法。

注意此类是纯虚类（抽象类），无法直接实体化。

### 2.2. polyMesh

在 primitiveMesh 纯虚类（抽象类）的基础上，OpenFOAM 又提供派生的 polyMesh 类。polyMesh 类基本上还是几何拓扑的，此外提供有一些几何拓扑的方法。

API 页面 https://api.openfoam.com/2506/classFoam_1_1polyMesh.html

终端输入命令，查阅 polyMesh 的声明

```terminal {fileName="terminal"}
find $FOAM_SRC -iname polyMesh.H
```

我们大概挑几处代码作为切入点简单了解一下 polyMesh 类。

```cpp {fileName="polyMesh.H",linenos=table,linenostart=1}
...
class polyMesh
:
	public objectRegistry, // 继承自这两个类
	public primitiveMesh
{
public:
	...
private:
	...
public:
	...
	// Constructors
		//- Read construct from IOobject
		explicit polyMesh(const IOobject& io, const bool doInit = true);
		// 基于 IOobject 对象构造 polyMesh
	...
...

```

使用 `polyMesh` 类，我们可以构造 `mesh` 对象，并进行一些网格操作。

对于几何要素，可以封装起来方便后续使用。

OpenFOAM 提供 IOobject 类来封装接入这些数据。

API 页面 https://api.openfoam.com/2506/classFoam_1_1IOobject.html

终端输入命令，查找 IOobject.H 

```terminal {fileName="terminal"}
find $FOAM_SRC -iname IOobject.H
```

大概挑几处代码作为切入点简单了解一下 IOobject 类。

```cpp {fileName="IOobject.H",linenos=table,linenostart=1}
...
class IOobject
:
	public IOobjectOption
{
public:
	...
private:
	...
protected:
	...
public:
	...
	// Constructors
	inline IOobject
	(
		const word& name, // 基于的对象名称
		const fileName& instance, // 基于的对象文件名称
		const objectRegistry& registry, // 注册相关，暂时无需了解
		IOobjectOption::readOption rOpt, // 读取选项
		IOobjectOption::writeOption wOpt = IOobjectOption::NO_WRITE, // 写入选项
		bool registerObject = true, // 有默认值，可忽略
		bool globalObject = false // 有默认值，可忽略
		// 暂时了解用法即可
	);
	...
...

```

修改主源码如下

```cpp {fileName="ofsp_15_mesh/ofsp_15_mesh.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    Foam::word regionName(Foam::polyMesh::defaultRegion);

    Foam::polyMesh mesh
    (
        IOobject
        (
            // fvMesh::defaultRegion,
            // 可以直接使用defaultRegion，也可以新建一个regionName
            regionName,
            runTime.timeName(),
            runTime,
            IOobject::MUST_READ
        )
    );

    Info<< "Max cell centre: " << max(mesh.cellCentres()) << endl;
    Info<< "Max cell volumes: " << max(mesh.cellVolumes()) << endl;
    Info<< "Max cell face cetres: " << max(mesh.faceCentres()) << endl;
	// 为了减少输出，每个都套了一个 max 函数

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

有读者可能会注意到 IOobject 的构造是基于 runTime 的，而不是直接基于 constant/polyMesh 的。回忆上篇讨论中 Time 类的查找机制，可以知道 Time 类具有复杂的查找机制。

>[!tip]
>这里涉及到源代码的实现细节，目前阶段不用深究也不建议过渡深究。

编译运行

终端输出信息如下

```terminal {fileName="terminal"}
Create time

Max cell centre: (0.0975 0.0975 0.005)
Max cell volumes: 2.5e-07
Max cell face cetres: (0.1 0.1 0.01)

ExecutionTime = 0 s  ClockTime = 0 s

End
```


### 2.3. fvMesh

OpenFOAM 在 polyMesh 类的基础上，添加了有限体积的方法，进而派生出了 fvMesh 类。

API 页面 https://api.openfoam.com/2506/classFoam_1_1fvMesh.html

终端输入命令，查阅 fvMesh 类的声明

```terminal {fileName="terminal"}
find $FOAM_SRC -iname fvMesh.H
```

大概挑几处代码作为切入点简单了解一下 fvMesh 类。

```cpp {fileName="fvMesh.H",linenos=table,linenostart=1}
...
class fvMesh
:
    public polyMesh, // 继承自 polyMesh
    public lduMesh,
    public fvSchemes,
    public surfaceInterpolation,    // needs input from fvSchemes
    public fvSolution,
    public data
{
protected:
	...
public:
	...
    // Constructors

        //- Construct from IOobject
        explicit fvMesh(const IOobject& io, const bool doInit=true);
        // 基于 IOobject 构造对象
	...
...

```

使用 fvMesh 类，我们可以构造 mesh 对象，并进行一些操作，包括网格的操作。

主源码修改为

```cpp {fileName="ofsp_15_mesh/ofsp_15_mesh.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    Foam::word regionName(Foam::polyMesh::defaultRegion);

    Foam::fvMesh mesh
    (
        IOobject
        (
            regionName,
            runTime.timeName(),
            runTime,
            IOobject::MUST_READ
        )
    );

    Info<< "Max cell centre: " << max(mesh.C()) << endl;
    Info<< "Max cell volumes: " << max(mesh.V()) << endl;
    Info<< "Max cell face cetres: " << max(mesh.Cf()) << endl;


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

编译运行

终端输出信息如下

```terminal {fileName="terminal"}
Create time

Max cell centre: max(C) [0 1 0 0 0 0 0] (0.1 0.1 0.005)
Max cell volumes: max(V) [0 3 0 0 0 0 0] 2.5e-07
Max cell face cetres: max(Cf) [0 1 0 0 0 0 0] (0.1 0.1 0.005)

ExecutionTime = 0 s  ClockTime = 0 s

End
```

读者也可以把 max 函数去掉重新编译查看结果。基于这三个子类和父类的讨论，我们可以想见 createMesh.H 主要是什么代码语句。

## 3. createMesh.H

API 页面 https://api.openfoam.com/2506/createMesh_8H.html

终端输入命令，查阅此头文件

```terminal {fileName="terminal"}
find $FOAM_SRC -iname createMesh.H
```

为了方便理解，我们查阅一下 OpenFOAM 2.0x 版本的代码，

Github 仓库文件链接如下 https://github.com/OpenFOAM/OpenFOAM-2.0.x/blob/master/src/OpenFOAM/include/createMesh.H

代码内容如下

```cpp {fileName="createMesh.H",linenos=table,linenostart=1}
//
// createMesh.H
// ~~~~~~~~~~~~

    Foam::Info
        << "Create mesh for time = "
        << runTime.timeName() << Foam::nl << Foam::endl;

    Foam::fvMesh mesh
    (
        Foam::IOobject
        (
            Foam::fvMesh::defaultRegion,
            runTime.timeName(),
            runTime,
            Foam::IOobject::MUST_READ
        )
    );
```

可以看到，这和我们在钱文写的代码一样。

现代版本的代码如下

```cpp {fileName="createMesh.H",linenos=table,linenostart=1}
Foam::autoPtr<Foam::fvMesh> meshPtr(nullptr);
// 创建自动指针，用来管理 fvMesh

// "getRegionOption.H"
Foam::word regionName
(
    args.getOrDefault<word>("region", Foam::polyMesh::defaultRegion)
);
// 获取区域名称，默认为 defaultRegion

if (args.dryRun() || args.found("dry-run-write")) // 是否 dry-run 测试模式
{
    Foam::Info
        << "Operating in 'dry-run' mode: case will run for 1 time step.  "
        << "All checks assumed OK on a clean exit" << Foam::endl;

    Foam::FieldBase::allowConstructFromLargerSize = true;

    // Create a simplified 1D mesh and attempt to re-create boundary conditions
    meshPtr.reset
    (
        new Foam::simplifiedMeshes::columnFvMesh(runTime, regionName)
    );
    // 创建1D简化网格

    // Stop after 1 iteration of the simplified mesh

    if (args.found("dry-run-write")) // 是否写入
    {
        // Using saWriteNow triggers function objects execute(), write()
        runTime.stopAt(Foam::Time::saWriteNow);

        // Make sure mesh gets output to the current time (since instance
        // no longer constant)
        meshPtr().setInstance(runTime.timeName());
    }
    else
    {
        // Using saNoWriteNow triggers function objects execute(),
        // but not write()
        runTime.stopAt(Foam::Time::saNoWriteNow);
    }

    Foam::functionObject::outputPrefix = "postProcessing-dry-run";
}
else // 非测试模式
{
    Foam::Info << "Create mesh";
    if (!Foam::polyMesh::regionName(regionName).empty())
    {
        Foam::Info << ' ' << regionName;
    }
    Foam::Info << " for time = " << runTime.timeName() << Foam::nl;
    // 如果区域名称非空（默认区域通常是 region0）

	// 通过指针创建 fvMesh 的对象
    meshPtr.reset
    (
        new Foam::fvMesh
        (
            Foam::IOobject
            (
                regionName,
                runTime.timeName(),
                runTime,
                Foam::IOobject::MUST_READ
            ),
            false
        )
    );
    meshPtr().init(true);   // initialise all (lower levels and current)

    Foam::Info << Foam::endl;
}

Foam::fvMesh& mesh = meshPtr();
```

两个版本的代码对比来看，我们大概可以理解该文件的主要内容是什么，以及现代版本增加了哪些机制。

## 4. 网格信息

我们可以综合上述讨论，在项目中使用网格信息。

主源码修改如下

```cpp {fileName="ofsp_15_mesh/ofsp_15_mesh.C",linenos=table,linenostart=1}
#include "fvCFD.H"

#include "IOmanip.H" // 输出格式控制

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    Info<< "Mesh directory: " << mesh.meshDir() << endl; // methods from polyMesh
	// 输出mesh存放的文件夹路径

    const pointField& p = mesh.points(); // mesh对象里保存的点
    Info<< "Number of points: " << p.size() << endl;
    for (int i=0; i<3; ++i) // 显示前3个点
    {
        Info<< "("
            << setf(ios_base::scientific)
            << setw(15)
            << p[i][0]; // 第i点的坐标分量1
        Info<< ", "
            << setf(ios_base::scientific)
            << setw(15)
            << p[i][1]; // 第i点的坐标分量2
        Info<< ", "
            << setf(ios_base::scientific)
            << setw(15)
            << p[i][2] // 第i点的坐标分量3
            << ")" << endl;
    }

    const faceList& f = mesh.faces(); // mesh对象里的面
    Info<< "Number of faces: " << f.size() << endl;
    forAll(f,i) // OpenFOAM提供的遍历方法
    {
        Info<< "("
            << setw(6) << f[i][0] // 组成第i面的第1个点的序号
            << setw(6) << f[i][1] // 组成第i面的第2个点的序号
            << setw(6) << f[i][2] // 组成第i面的第3个点的序号
            << setw(6) << f[i][3] // 组成第i面的第4个点的序号
            << ")" << endl;
    }
    // 可以打开 debug_case/constant/polyMesh/faces 比较

    const labelList& fOwner = mesh.faceOwner();
    Info<< "Number of face owner: " << fOwner.size() << endl;
    forAll(fOwner,i)
    {
        Info<< setw(6) << fOwner[i] << endl;
    }
    // 输出自然列表序号的面的owner的单元序号
	// 例如输出
	// 0                    // 0号面的owner是单元0
	// 0                    // 1号面的owner是单元0
	// 1                    // 2号面的owner是单元1
	// 2                    // 3号面的owner是单元2
	// ...
	// 以此类推

    const labelList& fNeigh = mesh.faceNeighbour();
    Info<< "Number of face neighbour: " << fNeigh.size() << endl;
    forAll(fNeigh,i)
    {
        Info<< setw(6) << fNeigh[i] << endl;
    }
    // 输出自然列表序号的面的neighbour的单元序号
    // 例如
    // 1                   // 0号面的neighbour是单元1
    // 2                   // 1号面的neighbour是单元2
    // 3                   // 2号面的neighbour是单元3
    // 3                   // 3号面的neighbour是单元3
    // 其他的面不是任何单元的neighbour

    const polyBoundaryMesh& bm = mesh.boundaryMesh();
    Info<< "Number of boundary mesh: " << bm.size() << endl;
    forAll(bm,i)
    {
        Info<< "Boundary name: " << bm[i].name()
            << "\tBoundary type: " << bm[i].type()
            << endl;
    }
    // 输出设置的边界


    Info<< nl << endl;

    Info<< "Bounding box: " << mesh.bounds() << endl;
    Info<< "Mesh volume: " << sum(mesh.V()).value() << endl;

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

为了方便结果显示，修改调试算例 debug_case/system/blockMeshDict 中设置的网格划分数量

```cpp {fileName="debug_case/system/blockMeshDict",linenos=table,linenostart=1}
...
blocks
(
    hex (0 1 2 3 4 5 6 7) (2 2 1) simpleGrading (1 1 1)
);
...

```

编译运行

终端输出信息如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Mesh directory: "polyMesh"
Number of points: 18
(   0.000000e+00,    0.000000e+00,    0.000000e+00)
(   5.000000e-02,    0.000000e+00,    0.000000e+00)
(   1.000000e-01,    0.000000e+00,    0.000000e+00)
Number of faces: 20
(     1     4    13    10)
(     3    12    13     4)
(     4    13    14     5)
(     4     7    16    13)
(     6    15    16     7)
(     7    16    17     8)
(     0     9    12     3)
(     3    12    15     6)
(     2     5    14    11)
(     5     8    17    14)
(     0     1    10     9)
(     1     2    11    10)
(     0     3     4     1)
(     3     6     7     4)
(     1     4     5     2)
(     4     7     8     5)
(     9    10    13    12)
(    12    13    16    15)
(    10    11    14    13)
(    13    14    17    16)
Number of face owner: 20
     0
     0
     1
     2
     2
     3
     0
     2
     1
     3
     0
     1
     0
     2
     1
     3
     0
     2
     1
     3
Number of face neighbour: 4
     1
     2
     3
     3
Number of boundary mesh: 3
Boundary name: movingWall       Boundary type: wall
Boundary name: fixedWalls       Boundary type: wall
Boundary name: frontAndBack     Boundary type: empty


Bounding box: (0.000000e+00 0.000000e+00 0.000000e+00) (1.000000e-01 1.000000e-01 1.000000e-02)
Mesh volume: 1.000000e-04

ExecutionTime = 0.000000e+00 s  ClockTime = 0.000000e+00 s

End
```

## 5. 网格方法

我们来练习使用更多的网格方法（成员方法）

主源码修改为

```cpp {fileName="ofsp_15_mesh/ofsp_15_mesh.C",linenos=table,linenostart=1}
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"

    #include "createMesh.H"

    Info<< "Time                    : " << runTime.timeName() << nl
        << "Number of mesh cells    : " << mesh.C().size() << nl
        << "Number of internal faces: " << mesh.Cf().size() << nl
        << endl;

    for (label cellI=0; cellI<mesh.C().size(); ++cellI) // C++原生方式遍历
    {
        if (cellI%100 == 0)
        {
            Info<< "Cell " << cellI
                << " with center at " << mesh.C()[cellI] // 返回单元中心坐标
                << endl;
        }
    }
    Info<< nl << endl;


    forAll(mesh.owner(),faceI) // OpenFOAM方式遍历
    {
        if (faceI%200 == 0)
        {
            Info<< "Internal face " << faceI
                << " with center at " << mesh.Cf()[faceI] // 返回面中心坐标
                << " with owner cell " << mesh.owner()[faceI] // 返回单元序号
                << " and neighbour cell " << mesh.neighbour()[faceI] // 返回单元序号
                << endl;
        }
    }
    Info<< nl << endl;

    forAll(mesh.boundaryMesh(), patchI)
    {
        Info<< "Patch " << patchI
            << " is " << mesh.boundary()[patchI].name() // 返回边界名称
            << " with " << mesh.boundary()[patchI].Cf().size() << " faces. "
            // 返回边界单元面的数量
            << "Start from face " << mesh.boundary()[patchI].start()
            // 返回边界其实面序号
            << endl;
    }
    Info<< nl << endl;

    label nIndex(0); // 自己设置一个序号
    forAll(mesh.boundaryMesh(), patchI) // 边界中遍历
    {
        Info<< "Patch " << patchI << nl
            << "\tits face " << nIndex
            << " adjacent to cell " << mesh.boundary()[patchI].patch().faceCells()[nIndex] << nl
            // 返回相邻单元序号
            << "\tits normal vector " << mesh.boundary()[patchI].Sf()[nIndex] << nl
            // 返回边界中面的面法向量
            << "\tits surface area " << mag(mesh.boundary()[patchI].Sf()[nIndex]) << nl
            // 返回面大小
            << endl;
    }
    Info<< nl << endl;


    const faceList& fcs = mesh.faces();
    const pointField& pts = mesh.points(); // 也可以使用 List<point>& 类型
    const List<point>& cents = mesh.faceCentres(); // 所有面心

    forAll(fcs,faceI) // 在所有面中遍历
    {
        if (faceI%200 == 0)
        {
            if (faceI < mesh.Cf().size()) // 如果是内部面
            {
                Info<< "Internal face ";
            } else // 否则是边界面
            {
                forAll(mesh.boundary(),patchI)
                {
                    if ( (mesh.boundary()[patchI].start() <= faceI) &&
                        (faceI < mesh.boundary()[patchI].start() +
                        mesh.boundary()[patchI].Cf().size()))
                        // 如果在此边界内
                    {
                        Info<< "Face on patch " << patchI << ", faceI ";
                        break;
                    }
                }
            }
            Info<< faceI << " with centre at " << cents[faceI] // 返回面心坐标
                << " has " << fcs[faceI].size() << " vertices: "; // 返回面节点数量
            forAll(fcs[faceI],vertexI) // 在该面遍历
            {
                Info<< " " << pts[fcs[faceI][vertexI]]; // 输出节点坐标
            }
            Info<< endl;
        }
    }
    Info<< nl << endl;

	// 一般来说， empty 的边界需要特别注意，有时候需要专门处理
	// 需要找出 empty 的边界的面
    forAll(mesh.boundaryMesh(),patchI) // 遍历组成边界的所有面
    {
        const polyPatch& pp = mesh.boundaryMesh()[patchI];
        if (isA<emptyPolyPatch>(pp)) // 如果面是empty的
        {
            Info<< "Patch " << patchI
                << ": " << mesh.boundary()[patchI].name() // 返回边界名称
                << " is empty."
                << endl;
        }
    }
    Info<< nl << endl;

    word patchName("movingWall");
    label patchID = mesh.boundaryMesh().findPatchID(patchName);
    // 找到符合名称的边界面
    Info<< "Retrived patch " << patchName
        << " at index " << patchID
        << " using its name only."
        << endl;
    Info<< nl << endl;


    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}

```

编译运行

终端输出信息如下

```terminal {fileName="terminal"}
Create time

Create mesh for time = 0

Time                    : 0
Number of mesh cells    : 4
Number of internal faces: 4

Cell 0 with center at (0.025 0.025 0.005)


Internal face 0 with center at (0.05 0.025 0.005) with owner cell 0 and neighbour cell 1


Patch 0 is movingWall with 2 faces. Start from face 4
Patch 1 is fixedWalls with 6 faces. Start from face 6
Patch 2 is frontAndBack with 0 faces. Start from face 12


Patch 0
        its face 0 adjacent to cell 2
        its normal vector (0 0.0005 0)
        its surface area 0.0005

Patch 1
        its face 0 adjacent to cell 0
        its normal vector (-0.0005 0 0)
        its surface area 0.0005

Patch 2
        its face 0 adjacent to cell 0
        its normal vector (0 0 -0.0025)
        its surface area 0.0025



Internal face 0 with centre at (0.05 0.025 0.005) has 4 vertices:  (0.05 0 0) (0.05 0.05 0) (0.05 0.05 0.01) (0.05 0 0.01)


Patch 2: frontAndBack is empty.


Retrived patch movingWall at index 0 using its name only.



ExecutionTime = 0.01 s  ClockTime = 0 s

End
```

## 6. 小结

通过讨论，我们简单认识了和网格相关的类，以及 createMesh.H 到底是什么。

到此为止，必备头文件 setRootCase.H，createTime.H，createMesh.H 全部已经讨论清楚。

本文完成讨论

- [x] 了解和网格相关的类
- [x] 理解 createMesh.H 头文件
- [x] 练习网格相关的方法
- [x] 编译运行 mesh 项目

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
