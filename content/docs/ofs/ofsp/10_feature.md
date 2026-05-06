---
uid: 20250916191753
title: 10_feature
date: 2025-09-16
update: 2026-03-26
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
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.



## 0. Preface

The previous discussion achieved reading parameter values by keyword, but as we mentioned, the `string` type does not meet practical application requirements. This section will further explore the self-developed library.

This section primarily discusses:

- [ ] Understanding the use of template classes
- [ ] Adding dictionary comments and error reporting mechanisms
- [ ] Compiling and running a feature project

## 1. Project Preparation

We will copy the previous project to create this project.

Run the following commands in the terminal to create this project by copying:

```terminal {fileName="terminal"}
ofsp
cp -r ofsp_10_keyword ofsp_10_feature
code ofsp_10_feature
```

Run the following commands in the terminal to clean and modify the project:

```terminal {fileName="terminal"}
wclean IOdictionary
wclean

mv ofsp_09_keyword.C ofsp_10_feature.C
sed -i s/"09_keyword"/"10_feature"/g Make/files
```

The project file structure at this point is as follows:

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

Run the following commands in the terminal to test whether the modified project is correct:

```terminal {fileName="terminal"}
wmake IOdictionary
wmake
ofsp_10_feature
```

The results show that the project is functioning normally.

## 2. Distinguishing Parameter Types

After obtaining the return value of type `string`, we need to further convert it to the required data type. Templates can be used for this development.

### 2.1. Development Improvements

The class declaration in `IOdictionary.H` has been improved as follows:

```cpp {fileName="/IOdictionary/IOdictionary.H",linenos=table,linenostart=1}
#pragma once

#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

namespace Aerosand // Custom namespace to avoid naming conflicts
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

		// Private member function, used only within the class
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
    dictionary_.open(filePath_); // Open the file

    T returnValue;

    std::string line; // Read line
    std::string valueLine; // Process line

    while (std::getline(dictionary_, line)) // Read line by line
    {
		size_t keywordPos = line.find(keyword); // Position of the first character of the keyword
		if (keywordPos != std::string::npos) // If the keyword is found
		{
			// Extract the string after the keyword (including spaces)
			valueLine = line.substr(keywordPos + keyword.length());
	
			// Remove all spaces
			size_t spacePos = valueLine.find(" ");
			while (spacePos != std::string::npos)
			{
				valueLine.erase(spacePos, 1);
				spacePos = valueLine.find(" ");
			}
		}
    }

    dictionary_.close(); // Close the file

    returnValue = convertFromString<T>(valueLine); // Convert to the user-specified type

    return returnValue;

}

}

```


> [!caution]
> Note that due to the different compilation mechanism of templates, we have placed the definition and declaration of the template member functions in the same file. Separating them would cause the main program to fail to find the instantiated objects of the functions. Since this involves more C++ knowledge, we will not delve into it here; it is sufficient to know that this approach works.

The class definition in `IOdictionary.C` has been improved as follows:

```cpp {fileName="/IOdictionary/IOdictionary.C",linenos=table,linenostart=1}
#include "IOdictionary.H"

// Constructor

Aerosand::IOdictionary::IOdictionary(const std::string& filePath)
{
    filePath_ = filePath;
}

// Destructor

Aerosand::IOdictionary::~IOdictionary() // C++ will automatically close it
{
    if (dictionary_.is_open())
    {
        dictionary_.close();
    }
}

```

The main source code in `ofsp_10_feature.C` has been improved as follows:

```cpp {fileName="/ofsp_10_feature.C",linenos=table,linenostart=1}
#include <iostream>
#include <fstream>
#include <string>

#include "IOdictionary.H"

int main(int argc, char const *argv[])
{
    Aerosand::IOdictionary ofspProperties("ofspProperties"); // Call the constructor

	// Call the template function
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

	// Verify that the types of the read parameters are correct and can participate in calculations
    std::cout << "\nWrite time interval = " << deltaT * writeInterval << std::endl;

    return 0;
}

```

### 2.2. Testing

The content of the dictionary file `ofspProperties` is as follows (located at `/ofsp/ofsp_10_feature/ofspProperties`):

```cpp {fileName="/ofspProperties"}
nu                  0.01
application         laplacianFoam
deltaT              0.005
writeInterval       20
purgeWrite          0
```

Run the following commands in the terminal to compile and run:

```terminal {fileName="terminal"}
wmake IOdictionary
wmake

ofsp_10_feature
```

The terminal output is as follows:

```terminal {fileName="terminal"}
nu              0.01
application     laplacianFoam
deltaT          0.005
writeInterval   20
purgeWrite      0

Write time interval = 0.1
```

It can be seen that the reading functionality works normally, and the calculation verifies that the types of the read parameter values are correct.

## 3. Style and Error Reporting Mechanism

With the ideas for string processing from the previous section, we can proceed with the style development and design of our custom class.

### 3.1. Development and Design Approach

The style design references C++, where each line ends with `;` to indicate the end of a statement, as shown below:

```cpp
nu         0.01;
```

Comment design references C++, with the following two forms:

```cpp
// nu is viscosity
nu                 0.01;

deltaT             0.005; // we can change deltaT
```

The error reporting design considers the following simple scenarios:

1. The dictionary file does not exist or the file name is incorrect. The program should terminate and output the corresponding error message.
2. The keyword in the dictionary cannot be found. The program should terminate and output the corresponding error message.    
3. The keyword exists in the dictionary but no value is assigned. The program should terminate and output the corresponding error message.

### 3.2. Development Improvements

The class declaration in `IOdictionary.H` has been further improved as follows:

```cpp {fileName="/IOdictionary/IOdictionary.H",linenos=table,linenostart=1}
#pragma once

#include <iostream>
#include <fstream>
#include <string>
#include <sstream>

namespace Aerosand // Custom namespace to avoid naming conflicts
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

		// Private member function, used only within the class
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
    dictionary_.open(filePath_); // Open the file

    T returnValue;

    bool foundKeyword = false; // Indicator of whether the keyword is found

    std::string line; // Read line
    std::string valueLine; // Process line

    while (std::getline(dictionary_, line)) // Read line by line
    {
        if (line.find("//") == 0) // Skip line comments
        {
            continue;
        }

        size_t keywordPos = line.find(keyword); // Position of the first character of the keyword
        if (keywordPos != std::string::npos) // If the keyword is found
        {
            foundKeyword = true; // Keyword found

            // Extract the string after the keyword
            valueLine = line.substr(keywordPos + keyword.length());

			// Extract the string before the semicolon
            if (valueLine.find(";"))
            {
                valueLine = valueLine.substr(0,valueLine.find(";"));
            }

			// Extract the string before //, redundant with the previous line
            if (valueLine.find("//") != std::string::npos)
            {
                valueLine = valueLine.substr(0, valueLine.find("//"));
            }

            // Remove all spaces
            size_t spacePos = valueLine.find(" ");
            while (spacePos != std::string::npos)
            {
                valueLine.erase(spacePos, 1);
                spacePos = valueLine.find(" ");
            }

			// If after processing, no value is found, report an error
            if (valueLine.empty()){
                std::cerr << "\n--> AEROSAND FATAL ERROR: " << std::endl
                    << "cannot find value of " << keyword << std::endl
                    << std::endl;
                exit(1);
            }
        } 
    }

	// If the keyword is not found, report an error
    if (!foundKeyword)
    {
        std::cerr << "\n--> AEROSAND FATAL ERROR: " << std::endl
            << "cannot find keyword " << keyword << std::endl
            << std::endl;
        exit(1);
    }

    dictionary_.close(); // Close the file

    returnValue = convertFromString<T>(valueLine); // Convert to the user-specified type

    return returnValue;

}

}

```

The class definition in `IOdictionary.C` has been improved as follows:

```cpp {fileName="/IOdictionary/IOdictionary.C",linenos=table,linenostart=1}
#include "IOdictionary.H"

// Constructor

Aerosand::IOdictionary::IOdictionary(const std::string& filePath)
{
    filePath_ = filePath;
    
    dictionary_.open(filePath_);
    if (!dictionary_.is_open()) // If opening fails, report an error
    {
        std::cerr << "\n--> AEROSAND FATAL ERROR: " << std::endl
            << "cannot find file \"" << filePath_ << "\"\n" << std::endl
            << std::endl;
        exit(1);
    }
    dictionary_.close();
}

// Destructor

Aerosand::IOdictionary::~IOdictionary() // C++ will automatically close it
{
    if (dictionary_.is_open())
    {
        dictionary_.close();
    }
}

```

The main source code remains unchanged.

### 3.3. Testing

Modify the local dictionary `ofspProperties` with the following content:

```cpp {fileName="/ofspProperties",linenos=table,linenostart=1}
// This is ofspProperties.

nu                  0.02; // see the difference

// test
deltaT              0.005; 
writeInterval       20;
purgeWrite          0;  // test


application         laplacianFoam;
```

Recompile, and readers can test various possible scenarios with the dictionary file. It can be observed that the results still meet the design requirements.

## 4. Summary

In the process of further developing features, we will realize that there are many data types that the reading operation needs to handle, especially OpenFOAM's own types. Moreover, there are also many OpenFOAM input file formats and exception handling cases to address. These functional features undoubtedly require better program architecture and more program development.

The good news is that through our own development of input and output functionality, I believe readers now have a certain conceptual understanding of OpenFOAM's dictionary files.

This section has completed the following discussions:

- [x] Understanding the use of template classes
- [x] Adding dictionary comments and error reporting mechanisms
- [x] Compiling and running a feature project


## Support us

>[!tip]
>Hopefully, the sharing here can be helpful to you.
>
>If you find this content helpful, your comments or donations would be greatly appreciated. Your support helps ensure the ongoing updates, corrections, refinements, and improvements to this and future series, ultimately benefiting new readers as well.
>
>The information and message provided during donation will be displayed as an acknowledgment of your support.

{{< cards >}}
  {{< card link="/" title="Support" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="AliPay" >}}
{{< /cards >}}


> Copyright @ 2026 Aerosand
> 
> - Course (text, images, etc.)：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
> - Code derived from OpenFOAM：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - Other code：[MIT License](https://opensource.org/licenses/MIT)


