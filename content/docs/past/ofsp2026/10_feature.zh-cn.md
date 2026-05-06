---
uid: 20250916191753
title: 10_feature
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
weight: 11
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

上一篇的讨论已经实现了按照关键词读取参数值，但是我们也提到，string 类型并不满足实际应用。这一篇讨论将进一步探索自开发库。

本文主要讨论

- [ ] 了解使用模板类
- [ ] 增加字典注释和报错机制
- [ ] 编译运行 feature 项目

## 1. 项目准备

我们拷贝上个项目来建立本项目。

终端输入命令，通过拷贝来建立本项目

```terminal {fileName="terminal"}
ofsp
cp -r ofsp_10_keyword ofsp_10_feature
code ofsp_10_feature
```

终端输入命令，清理并修改项目

```terminal {fileName="terminal"}
wclean IOdictionary
wclean

mv ofsp_09_keyword.C ofsp_10_feature.C
sed -i s/"09_keyword"/"10_feature"/g Make/files
```

此时的项目文件架构如下

```terminal {fileName="terminal"}
├── IOdictionary
│   ├── IOdictionary.C
│   ├── IOdictionary.H
│   └── Make
│       ├── files
│       └── options
├── Make
│   ├── files
│   └── options
├── ofsp_10_feature.C
└── ofspProperties
```

终端输入命令，测试修改后的项目是否正确

```terminal {fileName="terminal"}
wmake IOdictionary
wmake
ofsp_10_feature
```

结果显示项目正常。

## 2. 区分参数类型

当拿到 `string` 类型的返回值后，我们需要把返回值进一步转换成需要的数据类型。可以使用 template 进行开发。

### 2.1. 开发改进

类的声明 `IOdictionary.H` ，改进后内容如下

```cpp {fileName="/IOdictionary/IOdictionary.H",linenos=table,linenostart=1}
#pragma once

#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

namespace Aerosand // 自定义命名空间，避免命名重复
{

class IOdictionary
{
    
private:
    std::ifstream dictionary_;
    std::string filePath_;

public:

    // Constructor

        IOdictionary(const std::string& filePath);

    // Destructor

        ~IOdictionary();

private:

    // Private Member Function

		// 私有成员函数，仅在类内使用
        template<typename T>
        T convertFromString(const std::string& str);

public:

    // Public Member Function

        template<typename T>
        T lookup(const std::string& keyword);

};


// Private Member Function

template<typename T>
T Aerosand::IOdictionary::convertFromString(const std::string& str)
{
    T value;
    std::istringstream iss(str);
    iss >> value;

    return value;
}

// Public Member Function

template<typename T>
T Aerosand::IOdictionary::lookup(const std::string& keyword)
{
    dictionary_.open(filePath_); // 打开文件

    T returnValue;

    std::string line; // 读入行
    std::string valueLine; // 处理行

    while (std::getline(dictionary_, line)) // 按行读入
    {
		size_t keywordPos = line.find(keyword); // 关键词的第一个字母的位置
		if (keywordPos != std::string::npos) // 如果能找到关键词
		{
			// 截取关键词后的字符串（包含空格）
			valueLine = line.substr(keywordPos + keyword.length());
	
			// 剔除所有的空格
			size_t spacePos = valueLine.find(" ");
			while (spacePos != std::string::npos)
			{
				valueLine.erase(spacePos, 1);
				spacePos = valueLine.find(" ");
			}
		}
    }

    dictionary_.close(); // 关闭文件

    returnValue = convertFromString<T>(valueLine); // 转换为用户指定类型

    return returnValue;

}

}

```


> [!caution]
> 这里需要注意，因为模板的编译机制不同，我们把类内模板函数的定义和声明放在了同一个文件，分开会导致主程序找不到函数的实例化对象。因为涉及到了更多的 C++ 知识，这里暂不深究，知道这么处理就可以了。

类的定义 `IOdictionary.C`，改进后内容如下

```cpp {fileName="/IOdictionary/IOdictionary.C",linenos=table,linenostart=1}
#include "IOdictionary.H"

// Constructor

Aerosand::IOdictionary::IOdictionary(const std::string& filePath)
{
    filePath_ = filePath;
}

// Destructor

Aerosand::IOdictionary::~IOdictionary() // 其实 C++ 会自动关闭
{
    if (dictionary_.is_open())
    {
        dictionary_.close();
    }
}

```

主源码 `ofsp_10_feature.C` ，改进后内容如下

```cpp {fileName="/ofsp_10_feature.C",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>
#include <string>

#include "IOdictionary.H"

int main(int argc, char const *argv[])
{
    Aerosand::IOdictionary ofspProperties("ofspProperties"); // 调用构造函数

	// 调用模板函数
    double viscosity = ofspProperties.lookup<double>("nu");
    std::string application = ofspProperties.lookup<std::string>("application");
    double deltaT = ofspProperties.lookup<double>("deltaT");
    int writeInterval = ofspProperties.lookup<int>("writeInterval");
    bool purgeWrite = ofspProperties.lookup<bool>("purgeWrite");

    std::cout << "nu\t\t" << viscosity << std::endl;
    std::cout << "application\t" << application << std::endl;
    std::cout << "deltaT\t\t" << deltaT << std::endl;
    std::cout << "writeInterval\t" << writeInterval << std::endl;
    std::cout << "purgeWrite\t" << purgeWrite << std::endl;

	// 验证参数读取的类型正确，可以参与计算
    std::cout << "\nWrite time interval = " << deltaT * writeInterval << std::endl;

    return 0;
}

```

### 2.2. 测试

字典文件 `ofspProperties` 内容如下（路径为 `/ofsp/ofsp_10_io/ofspProperties`）

```cpp {fileName="/ofspProperties"}
nu                  0.01
application         laplacianFoam
deltaT              0.005
writeInterval       20
purgeWrite          0
```

终端输入命令，编译运行

```terminal {fileName="terminal"}
wmake IOdictionary
wmake

ofsp_10_feature
```

终端输出信息如下

```terminal {fileName="terminal"}
nu              0.01
application     laplacianFoam
deltaT          0.005
writeInterval   20
purgeWrite      0

Write time interval = 0.1
```

可以看到读取功能正常，并且通过计算验证了读取的参数值的类型也是正确的。

## 3. 风格和报错机制

有了前文对字符串的处理的思路，我们可以进行自定义类的风格开发设计。

### 3.1. 开发设计思路

风格的设计参考 C++，每行末尾加 `;` 表示语句结束，如下所示

```cpp
nu         0.01;
```

注释的设计参考 C++ ，有以下两种形式

```cpp
// nu is viscosity
nu                 0.01;

deltaT             0.005; // we can change deltaT
```

报错的设计简单考虑有以下几种情况

1. 字典文件不存在或者文件名称错误，应该终止程序并输出对应的报错信息
2. 字典中的关键词找不到，应该终止程序并输出对应的报错信息
3. 字典中的关键词存在，但是没有给定值，应该终止程序并输出对应的报错信息

### 3.2. 开发改进

类的声明 `IOdictionary.H` ，进一步改进后内容如下

```cpp {fileName="/IOdictionary/IOdictionary.H",linenos=table,linenostart=1}
#pragma once

#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

namespace Aerosand // 自定义命名空间，避免命名重复
{

class IOdictionary
{
    
private:
    std::ifstream dictionary_;
    std::string filePath_;

public:

    // Constructor

        IOdictionary(const std::string& filePath);

    // Destructor

        ~IOdictionary();

private:

    // Private Member Function

		// 私有成员函数，仅在类内使用
        template<typename T>
        T convertFromString(const std::string& str);

public:

    // Public Member Function

        template<typename T>
        T lookup(const std::string& keyword);

};


// Private Member Function

template<typename T>
T Aerosand::IOdictionary::convertFromString(const std::string& str)
{
    T value;
    std::istringstream iss(str);
    iss >> value;

    return value;
}

// Public Member Function

template<typename T>
T Aerosand::IOdictionary::lookup(const std::string& keyword)
{
    dictionary_.open(filePath_); // 打开文件

    T returnValue;

    bool foundKeyword = false; // 是否找到关键词的指标

    std::string line; // 读入行
    std::string valueLine; // 处理行

    while (std::getline(dictionary_, line)) // 按行读入
    {
        if (line.find("//") == 0) // 跳过行注释
        {
            continue;
        }

        size_t keywordPos = line.find(keyword); // 关键词的第一个字母的位置
        if (keywordPos != std::string::npos) // 如果能找到关键词
        {
            foundKeyword = true; // 找到关键词

            // 截取关键词后的字符串
            valueLine = line.substr(keywordPos + keyword.length());

			// 截取分号之前的字符串
            if (valueLine.find(";"))
            {
                valueLine = valueLine.substr(0,valueLine.find(";"));
            }

			// 截取 // 之前的字符串，其实和上一行冗余了
            if (valueLine.find("//") != std::string::npos)
            {
                valueLine = valueLine.substr(0, valueLine.find("//"));
            }

            // 剔除所有的空格
            size_t spacePos = valueLine.find(" ");
            while (spacePos != std::string::npos)
            {
                valueLine.erase(spacePos, 1);
                spacePos = valueLine.find(" ");
            }

			// 如果处理完发现没有值，报错
            if (valueLine.empty()){
                std::cerr << "\n--> AEROSAND FATAL ERROR: " << std::endl
                    << "cannot find value of " << keyword << std::endl
                    << std::endl;
                exit(1);
            }
        } 
    }

	// 如果没有找到关键词，报错
    if (!foundKeyword)
    {
        std::cerr << "\n--> AEROSAND FATAL ERROR: " << std::endl
            << "cannot find keyword " << keyword << std::endl
            << std::endl;
        exit(1);
    }

    dictionary_.close(); // 关闭文件

    returnValue = convertFromString<T>(valueLine); // 转换为用户指定类型

    return returnValue;

}

}

```

类的定义 `IOdictionary.C`，改进后内容如下

```cpp {fileName="/IOdictionary/IOdictionary.C",linenos=table,linenostart=1}
#include "IOdictionary.H"

// Constructor

Aerosand::IOdictionary::IOdictionary(const std::string& filePath)
{
    filePath_ = filePath;
    
    dictionary_.open(filePath_);
    if (!dictionary_.is_open()) // 如果打开失败，报错
    {
        std::cerr << "\n--> AEROSAND FATAL ERROR: " << std::endl
            << "cannot find file \"" << filePath_ << "\"\n" << std::endl
            << std::endl;
        exit(1);
    }
    dictionary_.close();
}

// Destructor

Aerosand::IOdictionary::~IOdictionary() // 其实 C++ 会自动关闭
{
    if (dictionary_.is_open())
    {
        dictionary_.close();
    }
}

```

主源码保持不变。

### 3.3. 测试

修改本地字典 `ofspProperties` 内容如下

```cpp {fileName="/ofspProperties",linenos=table,linenostart=1}
// This is ofspProperties.

nu                  0.02; // see the difference

// test
deltaT              0.005; 
writeInterval       20;
purgeWrite          0;  // test


application         laplacianFoam;
```

重新编译，读者可以测试字典文件可能出现的多种情况，可以发现结果依然满足设计要求。

## 4. 小结

在进一步开发特性的过程中，我们会意识到读取操作需要面对的数据类型还有很多，特别是 OpenFOAM 自己的类型。而且，需要处理的 OpenFOAM 输入文件格式和异常处理也有很多。这些功能特性毫无疑问需要更好的程序架构和更多的程序开发。

值的高兴的是，通过我们自己动手开发输入输出功能，相信读者现在对 OpenFOAM 的字典文件已经有了一定的概念。

本文完成讨论

- [x] 了解使用模板类
- [x] 增加字典注释和报错机制
- [x] 编译运行 feature 项目


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
