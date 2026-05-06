---
uid: 20251020195957
title: 12_arg
date: 2025-10-20
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
weight: 13
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

OpenFOAM 的很多应用都可以通过命令行来给定运行参数。虽然用户有时候会觉得有些陌生，但是用好命令行可以高效灵活的处理各种问题。

我们同样会从 C++ 开始，简单介绍C++中的命令行参数，之后进一步讨论 OpenFOAM 的命令行参数。

本文主要讨论

- [ ] 了解 OpenFOAM 命令行
- [ ] 理解 C++ 中的命令行参数
- [ ] 理解命令行参数的功能和选项
- [ ] 编译运行 arg 项目

## 1. OpenFOAM 命令行

参考阅读 https://doc.openfoam.com/2312/fundamentals/command-line/

OpenFOAM 命令行基础的使用格式如下

```terminal {fileName="terminal"}
<application> <options> <arguments>
```

比如，实践中有典型的命令行使用如下

```terminal {fileName="terminal"}
blockMesh -case debug_case
```

使用 `-help` 选项可以查看更多的命令行选项，例如终端输入 `blockMesh -help` ，终端会输出如下内容

```terminal {fileName="terminal"}
Usage: blockMesh [OPTIONS]
Options:
  -case <dir>       Case directory (instead of current directory)
  -dict <file>      Alternative blockMeshDict
  -merge-points     Geometric point merging instead of topological merging
                    [default for 1912 and earlier].
  -no-clean         Do not remove polyMesh/ directory or files
  -region <name>    Specify mesh region (default: region0)
  -sets             Write cellZones as cellSets too (for processing purposes)
  -time <time>      Specify a time to write mesh to (default: constant)
  -verbose          Force verbose output. (Can be used multiple times)
  -write-vtk        Write topology as VTU file and exit
  -doc              Display documentation in browser
  -help             Display short help and exit
  -help-compat      Display compatibility options and exit
  -help-full        Display full help and exit

Block mesh generator.

  The ordering of vertex and face labels within a block as shown below.
  For the local vertex numbering in the sequence 0 to 7:
    Faces 0, 1 (x-direction) are left, right.
    Faces 2, 3 (y-direction) are front, back.
    Faces 4, 5 (z-direction) are bottom, top.

                        7 ---- 6
                 f5     |\     :\     f3
                 |      | 4 ---- 5     \
                 |      3.|....2 |      \
                 |       \|     \|      f2
                 f4       0 ---- 1
    Y  Z
     \ |                f0 ------ f1
      \|
       o--- X

Using: OpenFOAM-2406 (2406) - visit www.openfoam.com
Build: _9bfe8264-20241212 (patch=241212)
Arch:  LSB;label=32;scalar=64
```


一般，数据处理和后处理的时候也会大量使用命令行。

例如，使用

```terminal {fileName="terminal"}
foamPostProcess -help
```

具体的使用技巧暂不展开，以后会专门讨论。

我们先简单回顾一下C++中命令行参数的使用。

## 2. 项目准备

终端输入命令，建立项目

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_12_arg
code ofsp_12_arg
```

通过 vscode ，使用 F1 输入 `Create C++ project` ，输入 `ofsp_12_arg` 作为项目名称，选择 `/ofsp` 作为父目录，初始化项目。

终端输入命令，测试初始项目

```terminal {fileName="terminal"}
make run
```

终端输出 `Hello world!` 表示初始项目运行成功。

## 3. 命令行参数

我们在初学 C++ 的时候，主函数的参数一般留空，即有如下形式

```cpp
...
int main() // 省略参数列表
{
	...
}
```

当进一步深入 C++ 开发时，了解到主函数的更一般写法为

```cpp
...
int main(int argc, char *argv[]) {}
// 或者
int main(int argc, char **argv) {}
```

其中

- `argc` 是 `argument count` 的缩写，保存程序运行时传递给主函数的参数个数
- `argv` 是 `argument vector` 的缩写，保存程序运行时传递给主函数的具体参数的字符型指针，每个指针都指向一个具体的参数。
	- `argv[0]` 指向程序运行时的全路径名称
	- `argv[1]` 指向程序运行时命令行中执行程序名后第一个字符串
	- `argv[2]` 指向程序运行时命令行中执行程序名后第二个字符串
	- 其他以此类推

## 4. 参数顺序

主源码 `src/main.cpp` 如下所示

```cpp {fileName="/src/main.cpp",linenos=table,linenostart=1}
#include <iostream>

int main(int argc, char *argv[])
{
    std::cout << "Number of arguments = " << argc << std::endl;

    for (int i=0; i<argc; ++i)
    {
        std::cout << "Argument " << i << ": "
            << argv[i] << std::endl;
    }

    return 0;
}
```

终端输入命令，编译运行

```terminal {fileName="terminal"}
make run
```

终端输出运行结果如下

```terminal {fileName="terminal"}
Number of arguments = 1
Argument 0: ./output/main
```

如果运行时增加参数，例如

```terminal {fileName="terminal"}
./output/main hi hey hello
```

终端输出运行结果如下

```terminal {fileName="terminal"}
Number of arguments = 4
Argument 0: ./output/main
Argument 1: hi
Argument 2: hey
Argument 3: hello
```


通过运行结果可以看到，`argc` 当然就是参数的总个数，`argv[0]`则是也就是应用名称本身，其他参数按顺序类推。

比如，对于以下命令行

```terminal {fileName="terminal"}
blockMesh -case debug_case
```

其中，`argc` 等于 `3`，而 `argv[0]` 是`blockMesh`，`argv[1]` 是 `-case`，`argv[2]` 是 `debug_case` 。

总结来说，命令行中的每一个参数其实都可以在程序中通过主函数参数被调用，以便参加程序运行和计算。

## 5. 参数功能

基于该项目，我们修改主源码，为命令行参数添加一些功能。

主源码 `src/main.cpp` 如下所示

```cpp {fileName="/src/main.cpp",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>
#include <string>
using namespace std;

int main(int argc, char *argv[])
{
    std::cout << "Number of arguments = " << argc << std::endl;

    for (int i = 0; i < argc; ++i)
    {
        std::cout << "Argument " << i << ": "
                  << argv[i] << std::endl;
    }

    // 如果没有额外命令行参数，执行此判断
    if (argc < 2) {
        cout << "Usage: " << argv[0] << " <filename> [mode]" << endl;
        cout << "Modes: read (default), write" << endl;
        return 1;
    }
    
    string filename = argv[1];
    string mode = (argc > 2) ? argv[2] : "read"; // 缺省mode设置为read
    
    if (mode == "read") { // 实现read模式
        ifstream file(filename);
        if (!file) { // 错误判断
            cout << "Error: Cannot open file " << filename << endl;
            return 1; // 终止程序
        }
        
        string line;
        cout << "File content:" << endl;
        // 输出文件中所有line
        while (getline(file, line)) {
            cout << line << endl;
        }
        file.close();
        
    } else if (mode == "write") { // 实现write模式
        ofstream file(filename, ios::app); // Append mode
        if (!file) { // 错误判断
            cout << "Error: Cannot create/write to file " << filename << endl;
            return 1;
        }
        
        cout << "Enter content to write (empty line to finish):" << endl;
        string line;
        // 只要line不是空白就持续按行输入
        while (getline(cin, line) && !line.empty()) {
            file << line << endl;
        }
        file.close();
        cout << "Content written to file" << endl;
        
    } else { // 模式错误判断
        cout << "Error: Unsupported mode '" << mode << "'" << endl;
        return 1;
    }
    
    return 0;
}
```

终端输入命令，编译程序

```terminal {fileName="terminal"}
make
```

终端输入命令，查看缺省输出

```terminal {fileName="terminal"}
./output/main
```

终端输出如下

```terminal {fileName="terminal"}
Number of arguments = 1
Argument 0: ./output/main
Usage: ./output/main <filename> [mode]
Modes: read (default), write
```

终端输入命令，对文件进行写入

```terminal {fileName="terminal"}
./output/main data.txt write
```

终端输出提示消息如下

```terminal {fileName="terminal"}
Number of arguments = 3
Argument 0: ./output/main
Argument 1: data.txt
Argument 2: write
Enter content to write (empty line to finish):

```

终端输入测试信息，输入结束`Enter`空白行结束输入

```terminal {fileName="terminal"}
ofsp
This is a ofsp test.

```

终端输入命令，对文件进行写出

```terminal {fileName="terminal"}
./output/main data.txt read
```

终端输出如下

```terminal {fileName="terminal"}
Number of arguments = 3
Argument 0: ./output/main
Argument 1: data.txt
Argument 2: read
File content:
ofsp
This is a ofsp test.
```

可以看到通过输入不同的命令行参数，可以实现程序的不同功能。

## 6. 参数选项

我们继续开发该项目，实现命令行参数选项。

主源码 `src/main.cpp` 如下所示

```cpp {fileName="/src/main.cpp",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
using namespace std;

// 函数-显示帮助信息
void displayHelp(const string& programName) {
    cout << "Usage: " << programName << " [OPTIONS] <filename> [mode]" << endl;
    cout << "A simple file read/write utility @ Aerosand" << endl << endl;
    cout << "Arguments:" << endl;
    cout << "  filename          Name of the file to read/write" << endl;
    cout << "  mode              Operation mode: read or write (default: read)" << endl << endl;
    cout << "Options:" << endl;
    cout << "  -h, --help        Display this help message" << endl;
    cout << "  -l                Show line numbers when reading" << endl;
    cout << "  -v, --version     Display version information" << endl;
    cout << "  -a, --append      Use append mode when writing (default)" << endl;
    cout << "  -o, --overwrite   Use overwrite mode when writing" << endl << endl;
    cout << "Examples:" << endl;
    cout << "  " << programName << " data.txt read" << endl;
    cout << "  " << programName << " -l notes.txt" << endl;
    cout << "  " << programName << " -o log.txt write" << endl;
    cout << "  " << programName << " -a data.txt write" << endl;
}

// 函数-显示版本信息
void displayVersion() {
    cout << "Version v1.0 @ Aerosand" << endl;
    cout << "This is a ofsp program from Aerosand" << endl;
}

// 结构体-存储配置的命令行参数
struct Config {
    string filename;
    string mode = "read";
    bool showLineNumbers = false;
    bool showHelp = false;
    bool showVersion = false;
    bool appendMode = true;  // 默认为追加模式
    bool overwriteMode = false;
};

// 函数-解析命令行参数
Config parseArguments(int argc, char* argv[]) {
    Config config;
    vector<string> positionalArgs;
    
    // 跳过程序名，遍历其余命令行参数
    for (int i = 1; i < argc; i++) {
        string arg = argv[i];
        
        if (arg == "-h" || arg == "--help") {
            config.showHelp = true;
        } else if (arg == "-l") {
            config.showLineNumbers = true;
        } else if (arg == "-v" || arg == "--version") {
            config.showVersion = true;
        } else if (arg == "-a" || arg == "--append") {
            config.appendMode = true;
            config.overwriteMode = false;
        } else if (arg == "-o" || arg == "--overwrite") {
            config.overwriteMode = true;
            config.appendMode = false;
        } else if (arg[0] == '-') {
            // 未知选项的异常处理
            cerr << "Warning: Unknown option '" << arg << "'" << endl;
            cerr << "Use '" << argv[0] << " --help' for usage information" << endl;
        } else {
            // 除此之外，文件名称和模式
            positionalArgs.push_back(arg);
        }
    }
    
    // 处理文件名称和模式
    if (positionalArgs.size() > 0) {
        config.filename = positionalArgs[0];
    }
    if (positionalArgs.size() > 1) {
        config.mode = positionalArgs[1];
    }
    
    return config;
}

int main(int argc, char* argv[])
{
    // 解析命令行参数
    Config config = parseArguments(argc, argv);
    
    // 如果显示帮助信息
    if (config.showHelp) {
        displayHelp(argv[0]);
        return 0;
    }
    
    // 如果显示版本信息
    if (config.showVersion) {
        displayVersion();
        return 0;
    }
    
    // 空白文件的异常处理
    if (config.filename.empty()) {
        cerr << "Error: No filename specified" << endl;
        cerr << "Use '" << argv[0] << " --help' for usage information" << endl;
        return 1;
    }
    
    // 模式错误的异常处理
    if (config.mode != "read" && config.mode != "write") {
        cerr << "Error: Invalid mode '" << config.mode << "'. Use 'read' or 'write'" << endl;
        return 1;
    }
    
    // 两种模式的实现
    if (config.mode == "read") {
        ifstream file(config.filename);
        if (!file) {
            cerr << "Error: Cannot open file " << config.filename << endl;
            return 1;
        }
        
        string line;
        int lineNumber = 1;
        cout << "File content:" << endl;
        
        while (getline(file, line)) {
            if (config.showLineNumbers) { // 增加行号的显示
                cout << lineNumber << ": " << line << endl;
            } else {
                cout << line << endl;
            }
            lineNumber++;
        }
        file.close();
        
    } else if (config.mode == "write") {
        // 根据选项决定使用追加模式还是覆盖模式
        ios_base::openmode openMode; // C++标准库的类型
        if (config.overwriteMode) {
            openMode = ios::out;  // 标准库中的覆盖模式
        } else {
            openMode = ios::app;  // 标准库中的追加模式（默认）
        }
        
        ofstream file(config.filename, openMode);
        if (!file) { // 文件名称的异常处理
            cerr << "Error: Cannot create/write to file " << config.filename << endl;
            return 1;
        }
        
        cout << "Enter content to write (empty line to finish):" << endl;
        string line;
        while (getline(cin, line) && !line.empty()) {
            file << line << endl;
        }
        file.close();
        
        // 根据模式显示不同的成功消息
        if (config.overwriteMode) {
            cout << "Content written to file (overwritten)" << endl;
        } else {
            cout << "Content appended to file" << endl;
        }
    }
    
    return 0;
}
```

终端输入命令，编译程序

```terminal {fileName="terminal"}
make
```

终端输入命令，查看帮助信息

```terminal {fileName="terminal"}
./output/main -h
```

终端输出如下

```terminal {fileName="terminal"}
Usage: ./output/main [OPTIONS] <filename> [mode]
A simple file read/write utility @ Aerosand

Arguments:
  filename          Name of the file to read/write
  mode              Operation mode: read or write (default: read)

Options:
  -h, --help        Display this help message
  -l                Show line numbers when reading
  -v, --version     Display version information
  -a, --append      Use append mode when writing (default)
  -o, --overwrite   Use overwrite mode when writing

Examples:
  ./output/main data.txt read
  ./output/main -l notes.txt
  ./output/main -o log.txt write
  ./output/main -a data.txt write
```

终端输入命令，查看版本信息

```terminal {fileName="terminal"}
./output/main --version
```

终端输出如下

```terminal {fileName="terminal"}
Version v1.0 @ Aerosand
This is a ofsp program from Aerosand
```

终端输入命令，对文件进行覆盖写入

```terminal {fileName="terminal"}
./output/main data.txt write -o
Enter content to write (empty line to finish):
ofsp

Content written to file
```

终端输入命令，对文件进行写出

```terminal {fileName="terminal"}
./output/main -l data.txt
```

终端输出如下

```terminal {fileName="terminal"}
File content:
1: ofsp
```

终端输入命令，对文件进行追加写入

```terminal {fileName="terminal"}
./output/main data.txt write -a
Enter content to write (empty line to finish):
This is a ofsp test from Aerosand.

Content appended to file
```

终端输入命令，对文件进行写出

```terminal {fileName="terminal"}
./output/main data.txt read -l
```

终端输出如下

```terminal {fileName="terminal"}
File content:
1: ofsp
2: This is a ofsp test from Aerosand.
```

可以看到，此项目可以执行多种命令行参数，也可以执行命令行选项，例如在输出中显式行号，选择文件内容写入的方式。

>[!tip]
>以上的代码均未考虑架构和优化等，只是为了让读者简单理解命令行参数的基本内容。

## 7. 小结

本项目讨论了 C++ 命令行参数的基本用法和实现，相信读者已经对命令行参数有了更多的了解。基于这些讨论，下一篇将过渡到 OpenFOAM 命令行参数的讨论。

本文完成讨论

- [x] 了解 OpenFOAM 命令行
- [x] 理解 C++ 中的命令行参数
- [x] 理解命令行参数的功能和选项
- [x] 编译运行 arg 项目

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
