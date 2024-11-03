[Introduction to the Pointer-to-Member Function
](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#1)[C++ Grammar](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#2)
[Pointer-to-member function is not regular pointer](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#3)
[C++ Type conversion rules](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#4)
[Pointer-to-member function array and an application](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#5)
[Member function call and this pointer](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#6)
[Conclusion](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#7)

## Introduction to the Pointer-to-Member Function

**Pointer-to-member** function is one of the most rarely used **C++ grammar**features. Even experienced C++ programmers are occasionally be confused. This article is a tutorial to beginners, and also shares my findings about the under-the-hood mechanism with more experienced programmers. Before we move on, let’s first take a look at a piece of code that might be a surprise at the first sight.

```cpp
//mem_fun1.cpp
#include <iostream>

class Foo{
public:
  Foo(int i=0){ _i=i;}
  void f(){
    std::cout<<"Foo::f()"<<std::endl;
  }
private:
  int _i;
};

int main(){
  Foo *p=0;
  p->f();
}
Output:
Foo::f()
```

Why can we call a member function through a NULL pointer? It seems that the compiler doesn’t care what value “p” holds. Only the type of “p” counts. We will leave the answer of this question to a later section. For now, what we care about is that the compiler knows exactly which function to call and this is just the well-known “static binding”. Because member functions can have static binding (not always, discussed later on), so their addresses are determined at compile time (not always). Intuitively, there should be a way to hold the address of member functions and here comes the pointer-to-member functions.



## C++ Grammar

The following grammar shows how to declare a pointer-to-member function.

```cpp
Return_Type (Class_Name::* pointer_name) (Argument_List);

Return_Type:   member function return type.
Class_Name:    name of the class in which the member function is declared.
Argument_List: member function argument list.
pointer_name:  a name we'd like to call the pointer variable.
```

e.g. We have a class Foo and a member function f:

```cpp
int Foo::f(string);
```

We could come up with a name for the pointer-to-member function as fptr, then we have:

```cpp
Return_Type:   int
Class_Name:    Foo
Argument_List: string

declaration of a  pointer-to-member function named "fptr":
  int (Foo::*fptr) (string);
```

To assign a member function to the pointer, the grammar is:

```cpp
  fptr= &Foo::f;
```

Of course declaration and initialization can be absorbed by one definition:

```cpp
  int (Foo::*fptr) (string) = &Foo::f;
```

To invoke the member function through the pointer, we use the pointer-to-member selection operators, either .* or ->* . The following code demonstrates the basics.

```cpp
#include <iostream>
#include <string>
using std::string;

class Foo{
public:
  int f(string str){
    std::cout<<"Foo::f()"<<std::endl;
    return 1;
  }
};

int main(int argc, char* argv[]){
  int (Foo::*fptr) (string) = &Foo::f;
  Foo obj;
  (obj.*fptr)("str");//call: Foo::f() through an object
  Foo* p=&obj;
  (p->*fptr)("str");//call: Foo::f() through a pointer
}
```

Notice that “.*fptr” binds fptr to the object “obj”, on the other hand “->*fptr” binds fptr to the object pointed to by “p”. (Another difference is that we can overload ->*, but not .* ) . The parenthesis around (obj.*fptr) and (p->*fptr) is grammatically mandatory.



## Pointer-to-member function is not regular pointer

Pointer-to-member function doesn’t hold the “exact address” like a regular pointer does. We can imagine it holds the “relative address” of where the function is in the class layout. Let’s now demonstrate the difference.

Now we make only one change in class Foo. Member function f is now “static”.

```cpp
#include <iostream>
#include <string>
using std::string;

class Foo{
public:
  static int f(string str){
    std::cout<<"Foo::f()"<<std::endl;
    return 1;
  }
};

int main(int argc, char* argv[]){
  //int (Foo::*fptr) (string) = &Foo::f;//error
  int (*fptr) (string) = &Foo::f;//correct
  (*fptr)("str");//call Foo::f()
}
```

A “static” member function has no “this” pointer, it is the same as a regular global function, except it shares the name scope of class Foo with other class members (In our case the name scope is Foo::). So the “static” member function is NOT part of the class. The pointer-to-member function grammar doesn’t work on regular function pointers, such as a pointer to “static” member function shown above. The error information of

```cpp
int (Foo::*fptr) (string) = &Foo::f;
```

from the compiler (g++ 4.2.4 ) is: ” cannot convert ‘int (*)(std::string)’ to ‘int (Foo::*)(std::string)”. This example demonstrates that pointer-to-member function is not regular pointer, otherwise, why does C++ bother to invent such grammar? Because it is different from regular pointer, the type conversion rules are also counter-intuitive.



## C++ Type Conversion rules

### Non-virtual case

Of course, pointer-to-member function (non-static member functions) can not be converted to regular pointers.(while, if one really really wants, using assembly technique and this can be done in a brute force way.) As we see in the previous section, pointer-to-member function is not regular pointer. Pointer-to-member function represents the “offset” rather than an “absolute address”. But what about the conversion between pointer-to-member function themselves?

```cpp
//memfun4.cpp
#include <iostream>
class Foo{
public:
  int f(char* c=0){
    std::cout<<"Foo::f()"<<std::endl;
    return 1;
  }
};

class Bar{
public:
  void b(int i=0){
    std::cout<<"Bar::b()"<<std::endl;
  }
};

class FooDerived:public Foo{
public:
  int f(char* c=0){
    std::cout<<"FooDerived::f()"<<std::endl;
    return 1;
  }
};

int main(int argc, char* argv[]){
  typedef  int (Foo::*FPTR) (char*);
  typedef  void (Bar::*BPTR) (int);
  typedef  int (FooDerived::*FDPTR) (char*);

  FPTR fptr = &Foo::f;
  BPTR bptr = &Bar::b;
  FDPTR fdptr = &FooDerived::f;

  //Bptr = static_cast<void (Bar::*) (int)> (fptr); //error
  fdptr = static_cast<int (Foo::*) (char*)> (fptr); //OK: contravariance

  Bar obj;
  ( obj.*(BPTR) fptr )(1);//call: Foo::f()
}

Output:
Foo::f()
```

In the above, we first introduce our friend “typedef”. It makes the definition and type information much clear to programmers. What is the type of “fptr”, btw? It is of the type:

```cpp
int (Foo::*) (char*);
```

or equivalently, FPTR. If we look closely in the above code:

```cpp
Bptr = static_cast<void (Bar::*) (int)> (fptr);//error
```

is an error, because different non-static non-virtual pointers-to-member function have strong type and can not be converted from one another. However,

```cpp
fdptr = static_cast<int (Foo::*) (char*)> (fptr);
```

is correct! This contravariance rule appears to be the opposite of the rule that we can assign a pointer to a derived class to a pointer to its base class (the “is-a” relationship). Nevertheless, the rule preserves the fundamental guarantee that FooDerived::* can be applied to any “interface” that Foo::* can be applied. The last line of the code:

```cpp
  Bar obj;
  ( obj.*(BPTR) fptr )(1);//call: Foo::f()
```

Although we want to call Bar::b(), but Foo::f() is called because fptr has static binding. (also see [Member function call and this pointer](https://www.codeguru.com/cplusplus/c-tutorial-pointer-to-member-function/#6)
)

### Virtual case

We only change all member functions to be virtual and all class definition is the same as the previous case.

```cpp
#include <iostream>
class Foo{
public:
  virtual int f(char* c=0){
    std::cout<<"Foo::f()"<<std::endl;
    return 1;
  }
};

class Bar{
public:
  virtual void b(int i=0){
    std::cout<<"Bar::b()"<<std::endl;
  }
};

class FooDerived:public Foo{
public:
  int f(char* c=0){
    std::cout<<"FooDerived::f()"<<std::endl;
    return 1;
  }
};

int main(int argc, char* argv[]){
  typedef  int (Foo::*FPTR) (char*);
  typedef  void (Bar::*BPTR) (int);
  FPTR fptr=&Foo::f;
  BPTR bptr=&Bar::b;

  FooDerived objDer;
  (objDer.*fptr)(0);//call: FooDerived::f(), not Foo::f()

  Bar obj;
  ( obj.*(BPTR) fptr )(1);//call: Bar::b() , not Foo::f()
}

Output:
FooDerived::f()
Bar::b()
```

As can be seen, when the member function is virtual, pointer-to-member function can have polymorphism and FooDerived::f() is called.Bar::b() now is also correctly called. Because “a pointer to a virtual member can safely be passed between different address spaces as long as the same object layout is used in both.” ( [Bjarne Stroustrup](http://www2.research.att.com/~bs/C++.html) , “[The C++ Programming Language](http://www2.research.att.com/~bs/C++.html)“). When the function is virtual, the compiler will generate virtual-table to store the address of virtual functions. This is the major difference to non-virtual member functions and hence the run time behavior is different.

## Pointer-to-member function array and an application

An important application of pointer-to-member functions is to generate the response events according to inputs. The following Printer class and the pointer array “pmf” demonstrate this.

```cpp
#include <stdio.h>
#include <string>
#include <iostream>
class Printer{//An abstract printing machine
public:
  void Copy(char * buff, const char * source){//copy the file
    strcpy(buff, source);
  }
  void Append(char * buff, const char * source){//extend the file
    strcat(buff, source);
  }
};

enum OPTIONS { COPY, APPEND };//two possible commands in the menu.
typedef void(Printer::*PTR) (char*, const char*);//pointer-to-member function

void working(OPTIONS option,
Printer* machine,
char* buff,
const char* infostr){

  PTR pmf[2]= {&Printer::Copy, &Printer::Append}; //pointer array

  switch (option){
  case COPY:
    (machine->*pmf[COPY])(buff, infostr);
    break;
  case APPEND:
    (machine->*pmf[APPEND])(buff, infostr);
    break;
  }
}

int main(){
  OPTIONS option;
  Printer machine;
  char buff[40];//target

  working(COPY, &machine, buff, "Strings ");
  working(APPEND, &machine, buff, "are concatenated! ");

  std::cout<<buff<<std::endl;
}
Output:
  Strings are concatenated!
```

In the above code, working is a function to carry out the printing work given 1. the menu option, 2. an available printing machine, 3. target, 4. source. The source is represented by two character strings “Strings ” and “are concatenated!” The pointer-to-member function array is used to select corresponding action according to different menu options. Another important application of pointer-to-member functions can be found in STL mem_fun().



## Member function call and this pointer

Now we look back at the beginning of this article. Why a null pointer can call a member function? For a non-virtual function call like: p->f(), the compiler will generate code like the following:

```cpp
Foo* const this=p;
void Foo::f( Foo *const this){
    std::cout<<"Foo::f()"<<std::endl;
}
```

So the function Foo::f can be called no matter what the value of “p” is. It is just like a global function! “p” is passed as “this pointer” to the function argument. The “this pointer” is not dereferenced in the function (in our special case) and therefore the compiler will let us go. What if we want to see the value of member data _i? Then the compiler need to dereference the “this pointer” and as a result, an undefined behavior. For a virtual function call, the correct version of the member function need to be found through virtual-table, then “this pointer” is passed to the correct version of the function. That’s why pointer-to-member function for non-virtual, virtual, static member functions are implemented in different ways.



## Conclusion

In conclusion, what we learned here is:
1. The grammar of pointer-to-member function declaration and definition.
2. The grammar to invoke member functions by pointer-to-member selection operators.
3. Use typedef to write clearer code.
4. Difference of non-virtual, virtual, static member functions.
5. Pointer-to-member function vs. regular pointer to member.
6. Pointer-to-member function conversion rules in different situations.
7. How to use pointer-to-member function array to solve a practical design problem.
8. How the member function call is reinterpreted by the compiler.

I hope this tutorial can open a door for us to explore more advanced topics related to the issues addressed above, such as pointer-to-member function under multiple inheritance, virtual inheritance, and also compiler implementation such as the [Microsoft Thunk technique](http://support.microsoft.com/kb/155763), etc.

Thank you for reading. I hope this article can be helpful.

### Additional Resources

[Pointers to member functions](http://www.parashift.com/c++-faq-lite/pointers-to-members.html)

[IBM: Pointers to members (C++ only)](http://publib.boulder.ibm.com/infocenter/comphelp/v8v101/index.jsp?topic=%2Fcom.ibm.xlcpp8a.doc%2Flanguage%2Fref%2Fcplr034.htm)

[Cprogramming.com](http://www.cprogramming.com/)