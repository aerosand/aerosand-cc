---
uid: 20250827141152
title: 05_vector
date: 2025-08-27
update: 2026-03-26
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - ofsp2026
  - OpenFOAM
  - ofsp
excludeSearch: false
toc: true
weight: 6
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

In the previous section, we briefly reviewed several common basic classes. This section attempts to discuss some details of the `vector` class to help readers become familiar with the development process, tool usage, and compilation principles.

This section primarily discusses:

- [ ] Practicing source code navigation
- [ ] Discussing parts of the implementation of the `vector` class
- [ ] Understanding the use of native libraries
- [ ] Compiling and running a `vector` project

## 1. Vector Class

API page: [https://api.openfoam.com/2506/classFoam_1_1Vector.html](https://api.openfoam.com/2506/classFoam_1_1Vector.html)

Run the following command in the terminal to search locally:

```terminal {fileName="terminal"}
find $FOAM_SRC -iname vector
```

Run the following command to open the class folder:

```terminal {fileName="terminal"}
code $FOAM_SRC/OpenFOAM/primitives/Vector
```

The file structure of this class is as follows:

```terminal {fileName="terminal"}
tree -L 1
.
├── bools/
├── complex/
├── floats/
├── ints/
├── lists/
├── Vector.H
└── VectorI.H
```

Reading the code description in `Vector.H`, we learn that this class is a template class for 3D Vectors. It inherits from the `VectorSpace` class and adds encapsulation for 3-component construction, component element interfaces, dot product, cross product, and other methods.

Specifically for the code in `Vector.H`, simple methods are implemented directly in the code, while complex methods extensively use inline functions to improve code performance. OpenFOAM specifically provides the `VectorI.H` file for implementing inline functions.

Enter the `Vector/int` folder and examine the code in `Vector/ints/labelVector.H`:

```cpp {filName="Vector/ints/labelVector.H"}
...
typedef Vector<label> labelVector;
...
```

Enter the `Vector/floats` folder and examine the code in `Vector/floats/vector.H`:

```cpp {fileName="Vector/floats/vector.H",linenos=table,linenostart=1}
...
//! \class Foam::floatVector
//! \brief A Vector of values with float precision
typedef Vector<float> floatVector;

//! \class Foam::doubleVector
//! \brief A Vector of values with double precision
typedef Vector<double> doubleVector;

// With float or double precision (depending on compilation)
typedef Vector<scalar> vector;

// With float or double precision (depending on compilation)
typedef Vector<solveScalar> solveVector;
...
```

It can be seen that the folders representing different data types (`/bools`, `/complex`, `/floats`, `/ints`, `/lists`) are all `typedef` aliases (type aliases) of the `Vector` template class for different basic data types.

> [!warning]
> Do not delve into all the details of the code at this stage.

## 2. Source Code Discussion

### 2.1. Declaration

Let us examine the code in `Vector/Vector.H` section by section.

API page: [https://api.openfoam.com/2506/Vector_8H_source.html](https://api.openfoam.com/2506/Vector_8H_source.html)

>[!tip]
>On the API code page, you can click to jump to different header files, classes, functions, etc.

```cpp {fileName="Vector/Vector.H",linenos=table,linenostart=1}
#ifndef Foam_Vector_H
#define Foam_Vector_H
// Preprocessor directive to prevent multiple inclusions of the header file

#include "contiguous.H" // Provides functionality for type contiguity checking
#include "VectorSpace.H" // Base class

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

namespace Foam // Namespace
{

// Forward Declarations
template<class T> class List; // Forward declaration
// Declares the List template class, which may be used later without needing the full header file

/*---------------------------------------------------------------------------*\
                           Class Vector Declaration
\*---------------------------------------------------------------------------*/

template<class Cmpt>
class Vector // Definition of the Vector template class
:
    public VectorSpace<Vector<Cmpt>, Cmpt, 3> // Inherits from base class, 3-dimensional vector space
{
public:

    // Typedefs // Define type aliases

        //- Equivalent type of labels used for valid component indexing
        typedef Vector<label> labelType;
        // Uses label type for components


    // Member Constants

        //- Rank of Vector is 1
        static constexpr direction rank = 1; // Defines rank as first-order


    //- Component labeling enumeration
    enum components { X, Y, Z }; // Enumeration corresponding to vector components


    // Generated Methods

        //- Default construct
        Vector() = default; // Default constructor

        //- Copy construct
        Vector(const Vector&) = default; // Copy constructor

        //- Copy assignment
        Vector& operator=(const Vector&) = default; // Copy assignment operator


    // Constructors

        //- Construct initialized to zero
        inline Vector(const Foam::zero); // Constructor initializing to zero

        //- Copy construct from VectorSpace of the same rank
        template<class Cmpt2>
        inline Vector(const VectorSpace<Vector<Cmpt2>, Cmpt2, 3>& vs);
        // Constructs a vector from a VectorSpace with different component types

        //- Construct from three components
        inline Vector(const Cmpt& vx, const Cmpt& vy, const Cmpt& vz);
        // Constructs a vector directly from three component values

        //- Construct from Istream
        inline explicit Vector(Istream& is);
        // Constructs a vector by reading from an input stream
        // The explicit keyword prevents implicit conversions


    // Member Functions

    // Component Access

        //- Access to the vector x component
        const Cmpt& x() const noexcept { return this->v_[X]; }

        //- Access to the vector y component
        const Cmpt& y() const noexcept { return this->v_[Y]; }

        //- Access to the vector z component
        const Cmpt& z() const noexcept { return this->v_[Z]; }
        
        // Uses enumeration values X, Y, Z as indices to access the v_ array in the base class
        // noexcept indicates that these functions do not throw exceptions

        //- Access to the vector x component
        Cmpt& x() noexcept { return this->v_[X]; }

        //- Access to the vector y component
        Cmpt& y() noexcept { return this->v_[Y]; }

        //- Access to the vector z component
        Cmpt& z() noexcept { return this->v_[Z]; }
        
        // Declares non-const member functions for accessing and modifying vector components


    // Vector Operations

        //- Return \c this (for point which is a typedef to Vector\<scalar\>)
        inline const Vector<Cmpt>& centre
        (
            const Foam::UList<Vector<Cmpt>>&  /* (unused) */
        ) const noexcept;
        // Defined in the inline file
        // Returns the vector itself (useful for point types)
        // The parameter is unused but retained for interface consistency

        //- The length (L2-norm) of the vector
        inline scalar mag() const;
        // Defined in the inline file
        // Returns the magnitude (length) of the vector

        //- The length (L2-norm) squared of the vector.
        inline scalar magSqr() const;
        // Defined in the inline file
        // Returns the squared magnitude of the vector (avoids square root operation, improving performance)

        //- The L2-norm distance from another vector.
        //- The mag() of the difference.
        inline scalar dist(const Vector<Cmpt>& v2) const;
        // Defined in the inline file
        // Returns the distance to another vector

        //- The L2-norm distance squared from another vector.
        //- The magSqr() of the difference.
        inline scalar distSqr(const Vector<Cmpt>& v2) const;
        // Defined in the inline file
        // Returns the squared distance to another vector

        //- Inplace normalise the vector by its magnitude
        //  For small magnitudes (less than ROOTVSMALL) set to zero.
        //  Will not be particularly useful for a vector of labels
        inline Vector<Cmpt>& normalise(const scalar tol = ROOTVSMALL);
        // Defined in the inline file
        // Normalizes the vector (converts to a unit vector)
        // The tol parameter is a tolerance for handling very small vectors

        //- Inplace removal of components that are collinear to the given
        //- unit vector.
        inline Vector<Cmpt>& removeCollinear(const Vector<Cmpt>& unitVec);
        // Defined in the inline file
        // Removes components collinear with the given unit vector

        //- Scalar-product of \c this with another Vector.
        inline Cmpt inner(const Vector<Cmpt>& v2) const;
        // Defined in the inline file
        // Returns the dot product (inner product) with another vector

        //- Cross-product of \c this with another Vector.
        inline Vector<Cmpt> cross(const Vector<Cmpt>& v2) const;
        // Defined in the inline file
        // Returns the cross product (outer product) with another vector


    // Comparison Operations

        //- Lexicographically compare \em a and \em b with order (x:y:z)
        static inline bool less_xyz
        (
            const Vector<Cmpt>& a,
            const Vector<Cmpt>& b
        );
        // Defined in the inline file
        // Compares two vectors lexicographically in the order X → Y → Z

        //- Lexicographically compare \em a and \em b with order (y:z:x)
        static inline bool less_yzx
        (
            const Vector<Cmpt>& a,
            const Vector<Cmpt>& b
        );
        // Defined in the inline file
        // Compares two vectors lexicographically in the order Y → Z → X

        //- Lexicographically compare \em a and \em b with order (z:x:y)
        static inline bool less_zxy
        (
            const Vector<Cmpt>& a,
            const Vector<Cmpt>& b
        );
        // Defined in the inline file
        // Compares two vectors lexicographically in the order Z → X → Y
};


// * * * * * * * * * * * * * * * * * Traits  * * * * * * * * * * * * * * * * //

// Type traits, not to be delved into at this stage

//- Data are contiguous if component type is contiguous
template<class Cmpt>
struct is_contiguous<Vector<Cmpt>> : is_contiguous<Cmpt> {};
// If the component type Cmpt is contiguous, then Vector<Cmpt> is also contiguous

//- Data are contiguous label if component type is label
template<class Cmpt>
struct is_contiguous_label<Vector<Cmpt>> : is_contiguous_label<Cmpt> {};
// If the component type Cmpt is label and contiguous, then Vector<Cmpt> is also

//- Data are contiguous scalar if component type is scalar
template<class Cmpt>
struct is_contiguous_scalar<Vector<Cmpt>> : is_contiguous_scalar<Cmpt> {};
// If the component type Cmpt is scalar and contiguous, then Vector<Cmpt> is also


template<class Cmpt>
class typeOfRank<Cmpt, 1>
{
public:

    typedef Vector<Cmpt> type;
};


template<class Cmpt>
class symmTypeOfRank<Cmpt, 1>
{
public:

    typedef Vector<Cmpt> type;
};


template<class Cmpt>
class typeOfSolve<Vector<Cmpt>>
{
public:

    typedef Vector<solveScalar> type;
};

// Not to be delved into at this stage

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

} // End namespace Foam

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

#include "VectorI.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

#endif
```

### 2.2. Inline

Many method definitions are in the inline file.

API page: [https://api.openfoam.com/2506/VectorI_8H_source.html](https://api.openfoam.com/2506/VectorI_8H_source.html)

Examine the code in `Vector/VectorI.H`:

```cpp {fileName="Vector/VectorI.H",linenos=table,linenostart=1}
// * * * * * * * * * * * * * * * * Constructors  * * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>::Vector(const Foam::zero)
 :
     Vector::vsType(Zero)
 {}
 // Constructor initializing to zero
 // - Passes the Zero constant
 
 
 template<class Cmpt>
 template<class Cmpt2>
 inline Foam::Vector<Cmpt>::Vector
 (
     const VectorSpace<Vector<Cmpt2>, Cmpt2, 3>& vs
 )
 :
     Vector::vsType(vs)
 {}
 // Template constructor
 // - Constructs from a VectorSpace with different component types
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>::Vector
 (
     const Cmpt& vx,
     const Cmpt& vy,
     const Cmpt& vz
 )
 {
     this->v_[X] = vx;
     this->v_[Y] = vy;
     this->v_[Z] = vz;
 }
 // Constructor from three components
 // - Directly accesses the v_ array member inherited from VectorSpace 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>::Vector(Istream& is)
 :
     Vector::vsType(is)
 {}
 // Constructor constructing a vector from an input stream
 
 
 // * * * * * * * * * * * * * * * Member Functions  * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline const Foam::Vector<Cmpt>& Foam::Vector<Cmpt>::centre
 (
     const Foam::UList<Vector<Cmpt>>&
 ) const noexcept
 {
     return *this;
 }
 // Directly returns a reference to the current object (*this)
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::magSqr() const
 {
     return
     (
         Foam::magSqr(this->x())
       + Foam::magSqr(this->y())
       + Foam::magSqr(this->z())
     );
 }
 // Computes the squared magnitude of the vector
 // - Uses OpenFOAM's magSqr() function to compute the square of each component
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::mag() const
 {
     return ::sqrt(this->magSqr());
 }
 // Computes the magnitude of the vector
 // - First computes the squared magnitude, then takes the square root using the standard sqrt function
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::distSqr(const Vector<Cmpt>& v2) const
 {
     return
     (
         Foam::magSqr(v2.x() - this->x())
       + Foam::magSqr(v2.y() - this->y())
       + Foam::magSqr(v2.z() - this->z())
     );
 }
 // Computes the squared distance to another vector
 // - Computes the sum of squares of the differences of each component
 
 
 template<class Cmpt>
 inline Foam::scalar Foam::Vector<Cmpt>::dist(const Vector<Cmpt>& v2) const
 {
     return ::sqrt(this->distSqr(v2));
 }
 // Computes the distance to another vector
 // - First computes the squared distance, then takes the square root
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>& Foam::Vector<Cmpt>::normalise(const scalar tol)
 {
     #ifdef __clang__
     volatile  // Use volatile to avoid aggressive branch optimization
     #endif
     const scalar s = this->mag();
 
     if (s < tol)
     {
         *this = Zero;
     }
     else
     {
         *this /= s;
     }
 
     return *this;
 }
 // Vector normalization
 // - Uses #ifdef __clang__ and the volatile keyword to prevent overly aggressive optimization by the Clang compiler
 // - If the magnitude is less than the tolerance tol, sets the vector to zero
 // - Otherwise, divides the vector by its magnitude to make it a unit vector
 // - Returns a reference to the current object
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>&
 Foam::Vector<Cmpt>::removeCollinear(const Vector<Cmpt>& unitVec)
 {
     *this -= (*this & unitVec) * unitVec;
     return *this;
 }
 // Removes components collinear with the given unit vector
 // - Computes the projection of the current vector onto the given unit vector direction
 // - Subtracts this projection component from the current vector
 // - Returns a reference to the current object
 
 
 // * * * * * * * * * * * * * * * Member Operations * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline Cmpt Foam::Vector<Cmpt>::inner(const Vector<Cmpt>& v2) const
 {
     const Vector<Cmpt>& v1 = *this;
 
     return (v1.x()*v2.x() + v1.y()*v2.y() + v1.z()*v2.z());
 }
 // Computes the dot product (inner product)
 // - Computes the sum of the products of the corresponding components of the two vectors
 
 
 template<class Cmpt>
 inline Foam::Vector<Cmpt>
 Foam::Vector<Cmpt>::cross(const Vector<Cmpt>& v2) const
 {
     const Vector<Cmpt>& v1 = *this;
 
     return Vector<Cmpt>
     (
         (v1.y()*v2.z() - v1.z()*v2.y()),
         (v1.z()*v2.x() - v1.x()*v2.z()),
         (v1.x()*v2.y() - v1.y()*v2.x())
     );
 }
 // Computes the cross product (outer product)
 // - Computes the components of the resulting vector according to the cross product formula
 // - Returns a new Vector object
 
 
 // * * * * * * * * * * * * * Comparison Operations * * * * * * * * * * * * * //
 
 template<class Cmpt>
 inline bool
 Foam::Vector<Cmpt>::less_xyz(const Vector<Cmpt>& a, const Vector<Cmpt>& b)
 {
     return
     (
         (a.x() < b.x())  // Component is less
      ||
         (
             !(b.x() < a.x())  // Equal? Check next component
          &&
             (
                 (a.y() < b.y())  // Component is less
              ||
                 (
                     !(b.y() < a.y())  // Equal? Check next component
                  && (a.z() < b.z())
                 )
             )
         )
     );
 }
 // Lexicographically compares two vectors in the order X → Y → Z
 // - First compares the X component; if not equal, returns the result
 // - If the X components are equal, compares the Y component
 // - If the Y components are also equal, compares the Z component
 
 
 template<class Cmpt>
 inline bool
 Foam::Vector<Cmpt>::less_yzx(const Vector<Cmpt>& a, const Vector<Cmpt>& b)
 {
     return
     (
         (a.y() < b.y())  // Component is less
      ||
         (
             !(b.y() < a.y())  // Equal? Check next component
          &&
             (
                 (a.z() < b.z())  // Component is less
              ||
                 (
                     !(b.z() < a.z())  // Equal? Check next component
                  && (a.x() < b.x())
                 )
             )
         )
     );
 }
 // Lexicographically compares two vectors in the order Y → Z → X
 
 
 template<class Cmpt>
 inline bool
 Foam::Vector<Cmpt>::less_zxy(const Vector<Cmpt>& a, const Vector<Cmpt>& b)
 {
     return
     (
         (a.z() < b.z())  // Component is less
      ||
         (
             !(b.z() < a.z())  // Equal? Check next component
          &&
             (
                 (a.x() < b.x())  // Component is less
              ||
                 (
                     !(b.x() < a.x())  // Equal? Check next component
                  && (a.y() < b.y())
                 )
             )
         )
     );
 }
 // Lexicographically compares two vectors in the order Z → X → Y
 
 
 // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
 
 namespace Foam
 {
 
 // * * * * * * * * * * * * * * * Global Functions  * * * * * * * * * * * * * //
 
 //- Linear interpolation of vectors a and b by factor t
 template<class Cmpt>
 inline Vector<Cmpt> lerp
 (
     const Vector<Cmpt>& a,
     const Vector<Cmpt>& b,
     const scalar t
 )
 {
     const scalar onet = (1-t);
 
     return Vector<Cmpt>
     (
         onet*a.x() + t*b.x(),
         onet*a.y() + t*b.y(),
         onet*a.z() + t*b.z()
     );
 }
 // Linear interpolation function
 // - Computes the linear interpolation result of two vectors
 // - t is the interpolation parameter, in the range [0, 1]
 // - When t=0, returns a; when t=1, returns b; intermediate values return a linear combination of the two
 
 
 // * * * * * * * * * * * * * * * Global Operators  * * * * * * * * * * * * * //
 
 //- Dummy innerProduct for scalar
 //  Allows the construction of vtables for virtual member functions
 //  involving the inner-products of fields
 //  for which a "NotImplemented" specialization for scalar is provided.
 template<class Cmpt>
 class innerProduct<Vector<Cmpt>, scalar>
 {
 public:
 
     typedef scalar type;
 };
 // Type trait for the specialized inner product of Vector<Cmpt> and scalar
 // - Defines the result type of the inner product as scalar
 
 
 template<class Cmpt>
 inline Cmpt operator&(const Vector<Cmpt>& v1, const Vector<Cmpt>& v2)
 {
     return v1.inner(v2);
 }
 // Overload of the dot product operator
 // - Calls the inner member function to compute the dot product
 
 
 template<class Cmpt>
 inline Vector<Cmpt> operator^(const Vector<Cmpt>& v1, const Vector<Cmpt>& v2)
 {
     return v1.cross(v2);
 }
 // Overload of the cross product operator
 // - Calls the cross member function to compute the cross product
 
 
 template<class Cmpt>
 inline Vector<Cmpt> operator*(const Cmpt& s, const Vector<Cmpt>& v)
 {
     return Vector<Cmpt>(s*v.x(), s*v.y(), s*v.z());
 }
 // Overload of the scalar-vector multiplication operator (scalar first)
 // - Multiplies each component of the vector by the scalar
 // - Returns a new vector
 
 template<class Cmpt>
 inline Vector<Cmpt> operator*(const Vector<Cmpt>& v, const Cmpt& s)
 {
     return s*v;
 }
 // Overload of the vector-scalar multiplication operator (scalar second)
 // - Achieves commutativity by calling the previous implementation
 
 
 
 // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
 
 } // End namespace Foam
```

> [!warning]
> The above code discussion is primarily intended to help readers become familiar with the use of C++ in OpenFOAM, overcome any strangeness or fear of the C++ language, and facilitate understanding of the code in subsequent practical sections. There is no need to spend additional time reading more OpenFOAM source code at this stage, nor to delve into code details. This will be covered in the `ofsc` series later.

## 3. Project Setup

Run the following commands in the terminal to create the project for this section:

```terminal {fileName="terminal"}
ofsp
mkdir ofsp_05_vector
code ofsp_05_vector
```

Continue using terminal commands or the VS Code interface to create additional files. The final file structure is as follows:

```terminal {fileName="terminal"}
tree
.
├── Aerosand
│   ├── Aerosand.C
│   ├── Aerosand.H
│   └── Make
│       ├── files
│       └── options
├── Make
│   ├── files
│   └── options
└── ofsp_05_vector.C
```

## 4. Development Library

### 4.1. Library Source Code

The code in `Aerosand.H` is:

```cpp {fileName="/Aerosand/Aerosand.H",linenos=table,linenostart=1}
#pragma once

class Aerosand
{
public:
    void SetLocalTime(double t);
    double GetLocalTime() const;


private:
    double localTime_;
};

```

The code in `Aerosand.C` is:

```cpp {fileName="/Aerosand/Aerosand.C",linenos=table,linenostart=1}
#include "Aerosand.H"

void Aerosand::SetLocalTime(double t) {
    localTime_ = t;
}

double Aerosand::GetLocalTime() const {
    return localTime_;
}

```

### 4.2. Library Make

The library `Make/files` is:

```wmake {fileName="/Aerosand/Make/files",linenos=table,linenostart=1}
Aerosand.C

LIB = $(FOAM_USER_LIBBIN)/libAerosand

```

The development library has no other dependencies; the library `Make/options` is left empty.

### 4.3. Library Compilation

Run the following commands in the terminal to compile the library:

```terminal {fileName="terminal"}
wclean Aerosand
wmake Aerosand
```

## 5. Main Project

### 5.1. Main Source Code

The code in `ofsp_05_vector.C` is:

```cpp {fileName="/ofsp_05_vector.C",linenos=table,linenostart=1}
#include <iostream>

#include "Aerosand.H"

#include "vector.H"

using namespace Foam;

int main()
{
    scalar s(3.14);
    vector v1(1, 2, 3);
    vector v2(2, 3, 4);
    Info<< s << " * " << v1 << " = "  << s * v1 << nl 
        << "distance:  " << v1.dist(v2) << nl
        << "normalise: " << v1.normalise(1e-6) << nl
        << "inner product: " << v1.inner(v2) << nl
        << "inner product: " << (v1 & v2) << nl // Parentheses used to improve operator precedence
        << "cross product: " << v1.cross(v2) << nl
        << "cross product: " << (v1^v2) << nl // Parentheses used to improve operator precedence
        << endl;

	label a = 1;
    scalar pi = 3.1415926;
    Info<< "Hi, OpenFOAM!" << " Here we are." << nl
	    << a << " + " << pi << " = " << a + pi << nl
	    << a << " * " << pi << " = " << a * pi << nl
	    << endl;


    Aerosand mySolver;
    mySolver.SetLocalTime(0.2);
    Info<< "\nCurrent time step is : " << mySolver.GetLocalTime() << endl;

    return 0;
}

```

### 5.2. Project Make

The project `Make/files` is:

```wmake {fileName="/Make/files",linenos=table,linenostart=1}
ofsp_05_vector.C

EXE = $(FOAM_USER_APPBIN)/ofsp_05_vector

```

Because we included `vector.H`, whose path is `$FOAM_SRC/OpenFOAM/primitives/Vector/\frac{floats}{vector.H`, we can determine from the Make file location that this file belongs to the OpenFOAM library at `$FOAM_SRC/OpenFOAM`. The affiliation is as follows:

```terminal {fileName="terminal"}
$FOAM_SRC/OpenFOAM
├── lnInclude/
├── ...
├── Make
│   ├── files
│   └── options
└── primitives
	├── ...
	└── Vector
		├── ...
		├── floats
		│   ├── ...
		│   └── vector.H
		├── Vector.H
		└── VectorI.H
```

In principle, we should include the OpenFOAM library in the project `Make/options`. In practice, OpenFOAM’s build system (`wmake`) automatically handles the dependencies of essential built-in libraries. We only need to add third-party libraries, self-developed libraries, and libraries for certain optional modules.

> [!important]
> The `$FOAM_SRC/OpenFOAM` library is automatically included as a dependency; any classes used from it do not require the user to link again.

The project `Make/options` is:

```wmake {fileName="/Make/options",linenos=table,linenostart=1}
EXE_INC = \
    -IAerosand/lnInclude

EXE_LIBS = \
    -L$(FOAM_USER_LIBBIN) \
    -lAerosand

```

## 6. Compilation and Execution

Run the following commands in the terminal to compile and run the project:

```terminal {fileName="terminal"}
wclean
wmake
ofsp_05_vector
```

The output is as follows:

```terminal {fileName="terminal"}
3.14 * (1 2 3) = (3.14 6.28 9.42)
distance:  1.73205
normalise: (0.267261 0.534522 0.801784)
inner product: 5.34522
inner product: 5.34522
cross product: (-0.267261 0.534522 -0.267261)
cross product: (-0.267261 0.534522 -0.267261)

Hi, OpenFOAM! Here we are.
1 + 3.14159 = 4.14159
1 * 3.14159 = 3.14159


Current time step is : 0.2
```

## 7. Summary

This section has completed the following discussions:

- [ ] Practicing source code navigation
- [ ] Discussing parts of the implementation of the `vector` class
- [ ] Understanding the use of native libraries
- [ ] Compiling and running a `vector` project

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

