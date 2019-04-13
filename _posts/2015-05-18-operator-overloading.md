---
layout: post
title:  "C++ 的操作符重载"
categories: C++
tags: C++
author: 肖邦
---

* content
{:toc}

> C++ 的操作符重载




* 操作符标记与操作符函数
* 典型双目操作符的重载
* 典型单目操作符的重载
* 输入输出操作符的重载
* 其它操作符的重载
* 不能被重载的操作符
* 操作符重载的局限性


## 一、操作符标记与操作符函数

* 操作符标记
    - 单目操作符：`-`、`++`、`--`、`*`、`->`、`!`、`~` 等
    - 双目操作符：`+`、`-`、`+=`、`-=`、`>>`、`<<`、`[]` 等
    - 三目操作符：`?:`
* 操作符函数
    - 在特定条件下，编译器有能力把一个由操作数和操作符共同组成的表达式，解释为一个全局或成员函数的调用，该全局函数或成员函数被称作操作符函数。
    - 通过定义操作符函数，可以实现针对自定义类型的运算法则，并使之内置类型一样参与各种表达式的运算
* 双目操作符表达式：`L#R`
    - 成员函数形式:L.operator# (R)。左操作数是调用对象，右操作数是参数对象
    - 全局函数形式:::operator# (L, R)。左操作数是第一参数，右操作数是第二参数
* 单目操作符表达式:#O/O#
    - 成员函数形式:O.operator# ()
    - 全局函数形式:::operator# (O)
* 三目操作符表达式:F#S#T
    - 无法重载


## 二、友元
* 可以通过friend关键字，把一个全局函数、另一个类的成员函数或者另一个类整体，声明为某个类的友元。
* 友元拥有访问授权类任何非公有成员的特权。
* 友元声明可以出现在授权类的公有、私有和保护等任何区域，且不受访问控制限定符的约束
* 友元不是成员，其作用域并不隶属于授权类，也不拥有授权类类型的 this 指针
* 操作符函数常被声明为其参数类型的友元


## 三、典型双目操作符的重载
* 运算类双目操作符： +、-、*、/ 等。
    - 左右操作数均可为左值或右值
    - 表达式的值（即返回值）为右值。
    - 成员函数的形式：
        ```cpp
        class LEFT {
        const RESULT operator# (const RIGHT& right) const { ... }
        };
        ```
    - 全局形式：
        ```cpp
        class RESULT operator# (const LEFT& left,const RIGHT& right) { ... }
        ```
    - 注： RESULT -- 返回值类型  LEFT -- 左操作数类型   RIGHT -- 右操作数类型
        ```
        const 的加与不加要根据实际情况。判断依据：是否可以作为右值或者左值(能否修改)
        ```

* 赋值类双目操作符：=、+=、-=、*=、/= 等
    - 右操作数为左值或右值，但左操作数必须是左值。
    - 表达式的值为左值，且为左操作数本身（而非副本）
    - 成员函数形式：
        ```cpp
        class LEFT {
            LEFT& operator# (const RIGHT& right) { ... }
        };
        ```
    - 全局函数形式：
        ```cpp
        LEFT& operator# (LEFT& left,const RIGHT& right) { ... }
        ```

## 四、典型单目操作符的重载

* 运算类单目操作符：-、~、！等
    - 操作数为左值或右值
    - 表达式的值为右值
    - 成员函数形式：
        ```cpp
        class OPERAND {
            const RESULT operator# (void) const { ... }
        };
        ```
    - 全局函数形式：
        ```cpp
        const RESULT operator# (const OPERAND& operand) { ... }
        ```
* 前自增减类单目操作符：前++、前--
    - 操作数为左值
    - 表达式的值为左值，且为操作数本身（而非副本）
    - 成员函数形式：
        ```cpp
        class OPERAND {
          OPERAND& operator# (void) { ... }
        };
        ```
    - 全局函数形式：
        ```cpp
        OPERAND& operator# (OPERAND& operand) { ... }
        ```
* 后自增减类单目操作符：后++、后--
    - 操作数为左值
    - 表达式的值为右值，且为自增减以前的值
    - 成员函数形式
        ```cpp
        class OPERAND {
          const OPERAND operator# (int) { ... }
        };
        ```
    - 全局函数形式：
        ```cpp
        const OPERAND operator# (OPERAND& operand,int) { ... }
        ```


## 五、输入输出操作符的重载

* 输出操作符重载：`<< `
    - 左操作数为左值形式的输出流(ostream)对象，右操作数为左值或者右值
    - 表达式的值为左值，且为左操作数本身（而非副本）
    - 左操作数的类型为 "ostream" ，若以成员函数形式重载该操作符，就应将其定义为 ostream 类的成员，该类为标准库提供，无法添加新的成员，因此只能以全局函数形式重载该操作符，一般要访问 RIGHT 类的私有成员,需要加 friend 友元
        ```cpp
        friend ostream& operator<< (ostream& os,const RIGHT& right) { ... }
        ```

* 输入操作符：`>>`
    - 左操作数为左值形式的输入流（iostream）对象，右操作数为左值。
    - 表达式的值为左值，且为左操作数本身（而非副本）
    - 左操作数的类型为istream，若以成员函数形式重载该操作符，就应该将其定义为istream 类的成员，该类为标准库提供，无法添加新的成员，因此只能以全局函数形式重载该操作符。
        ```cpp
        friend istream& operator>> (istream& is,RIGHT& right) { ... }
        ```

## 六、其它操作符的重载

**1、下标操作符： "[]"**

* 常用于在容器类型中以 "下标" 方式获取数据元素
* 非常容器的元素为左值，常容器的元素为右值
* 举例：
    ```cpp
    class Array {
    public:
      int& operator[] (size_t i) { return m_array[i] }  /* 非常版本 */
      const int& operator[] (size_t i) const {          /* 常版本 */
        return const_cast<Array&>(*this)[i];
      }
    private:
      int m_array[256];
    };
    ```
    使用：
    ```cpp
    Array array;
    array[100] = 1000;          // array.operator[] (100) = 1000;
    const Array& carr = array;
    cout << carr[100] << endl;  // cout << carr.operator[] (100) << endl;
    ```
**2、函数操作符："()"**

* 如果一个类重载了函数操作符，那么该类的对象就可以被当作函数来调用，其参数和返回值就是函数操作符函数的参数和返回值。
* 参数的个数、类型以及返回值的类型，没有限制
* 唯一可以带有缺省参数的操作符函数
* 举例：
    ```cpp
    class Less {
    public；
        bool operator() (int a,int b) const {
            return a < b;
        }
    };
    ```
    使用：
    ```cpp
    Less less;
    cout << less (100,200) << endl;   /* false */ 
    // cout << less.operator() (100,200) << endl;
    ```

**3、解引用和间接成员访问操作符**

* 如果一个类重载了解引用和间接引用访问操作符，那么该类的对象就可以被当做指针来使用
* 用例说明：
    ```cpp
    class Integer {
    public:
        Integer (const int& val = 0): m_val (val) {}
        int& value (void) { return m_val; }
        const int& value (void) const { return m_val; }
    private:
        int m_val;
    };

    class IntegerPointer {
    public:
        IntegerPoint (Integer* p = NULL) :m_p (p) {}
        Integer& operator* (void) const { return *m_p; }
        Integer* operator-> (void) cosnt { return m_p; }
    private:
         Integer* m_p;
    };

    IntegerPointer ip (new Integer (100));
    (*ip).value ()++;    // ip.operator* ().value()++;
    cout << ip->value() << endl; 
    // cout << ip.operator-> ()->value () << endl;
    ```
**4、自定义类型转换(2 种方式)**

* 通过构造函数实现自定义类型转换 
    ```
    class 目标类型 {
        [explicit] 目标类型 (const 源类型& src)  { ... }
    };
    ```
* 通过类型转换操作符函数实现自定义类型转换
    ```
    class 源类型 {
        [explicit] operator 目标类型 (void) const { ... }
    };
    ```
* 若源类型是基类类型，则只能通过 "构造函数" 实现自定义类型转换。

**5、自定义类型转换的总结及注意事项：**

* 若目标类型是基本类型，则只能通过"类型转换操作符函数"实现自定义类型转换
* 若源类型和目标类型都不是基本类型，则既可以通过构造函数也可以通过类型转换操作符函数实现自定义类型转换，但不要两者同时使用，引发歧义错
* 若源类型和目标类型都是基本类型，则无法实现自定义类型转换，基本类型间的类型转换规则完全由编译器内置。
* 举例：
    ```cpp
    class Integer {
    public；
        Integer (const int& val = 0) :m_val (val) {}
        operator int (void) const { return m_val; }
    private:
        int m_val;
    };

    Integer integer (100);
    integer = 300;
    int i = integer;
    // int i = integer.operator int ();
    ```
**6、对象创建操作符：new/new[]**

* 如果一个类重载了 new/new[] 操作符，那么当通过 new/new[] 创建该类的对象/对象数组时，将首先调用该操作符函数 分配内存，然后再调用该类的构造函数：
    ```
    class 类名 {
        static void* operator new (size_t size) { ... }
        static void* operator new[] (size_t size) { ... }
    };
    ```

* 包含自定义析构函数的类，通过 new[] 创建对象数组，所分配的内存会在低地址部分预留出 sizeof(size_t) 个字节，存放 "数组长度"
* 举例：
    ```cpp
    class Dummy {
    public:
        Dummy (void) {}
        ~Dummy (void) {}
        static void* operator new (size_t size) { return malloc (size); }
        static void* operator new[] (size_t size) { return malloc (size); }
    };

    Dummy* dummy = new Dummy;
    // Dummy* dummy = (Dummy*)Dummy::operator new (sizeof (Dummy));
    // dummy->Dummy ();
    Dummy* dummies = new Dummy[10];
    // Dummy* dummies = (Dummy*)((size_t*) Dummy::operator new[] (sizeof (size_t ) +10 * sizeof (Dummy)) + 1);
    // *((size_t*)dummies - 1) = 10;
    // for (size_t i = 0; i<*((size_t*)dummies-1); ++i) 
    //      (dummies + i)->Dummy (); 
    ```
**7、对象销毁操作符：delete/delete[]**

* 如果一个类重载了 delete/delete[] 操作符，那么当通过 delete/delete[]销毁该类的对象/对象数组时，将首先调用该类的析构函数，然后再调用该操作符函数释放内存
    ```
    class 类名 {
        static void operator delete (void* p) { ... }
        static void operator delete[] (void* p) { ... }
    };
    ```
* 包含自定义析构函数的类，提供 delete[]销毁对象数组，会根据低地址部分预存的数组长度，从高地址到低地址依次对每个数组元素调用析构函数
* 堆对象一定是通过delete调用析构函数
* 举例：
    ```cpp
    class Dummy {
    public:
        Dummy (void) {}
        ~Dummy (void) {}
        static void operator delete (void* p) { free (p) }
        static void operator delete[] (void* p) { free (p) }  
    };

    delete dummy;      /* 下边是解释语句 */
    dummy->~Dummy ();  
    Dummy::operator delete (dummy);
    delete[] dummies;  /* 下边是解释语句 */
    for (size_t i = *((size_t*)dummies - 1) - 1;;--i) {
        (dummies + i)->~Dummy ();
        if (i == 0) break;
    } 
    Dummy::operator delete[] ((size_t*)dummies - 1);
    ```
**8、对操作符重载的正确认识及体会**

* 作用：能够使代码变得简单化，使我们可以更抽象的方式和更高的层次去理解所谓指针，站在C++的高度应该给指针一个新的概念：支持这种运算符规则的都可以看作指针。
* 关注行为，不注重细节。不关注属性，注重行为


## 七、智能指针auto_ptr

* 常规指针的缺点：
    - 当一个常规指针离开它的作用域时，只有该指针变量本身所占据的内存空间（通常是4个字节）会被释放，而它所指向的动态内存并为得到释放
    - 在某些特殊情况下，包含 `free/delete/delete[]` 的代码根本就执行不到，形成内存泄漏
* 智能指针的优点：
    - 智能指针是一个封装了常规指针的类类型对象，当它离开作用域时，其析构函数负责释放该常规指针所指向的动态内存
    - 以正确方式创建的智能指针，其析构函数总会执行
* 智能指针与常规指针的一致性：
    - 为了使智能指针也能像常规指针一样，通过 `*`* 操作符解引用，通过 `->` 操作符访问其目标的成员，就需要对这两个操作符进行重载
* 智能指针与常规指针的不一致性
    - 任何时候，针对一个对象，只允许有一个智能指针持有其地址，否则该对象将在多个智能指针中被析构多次(double free)
    - 智能指针的拷贝构造函数和拷贝赋值需要做特殊处理，对其所持有的对象地址，以指针间的转移代替复制
    - 智能指针的转义语义与常规指针的复制语义不一致
    - 智能指针不能用于数组(指针不知道数组的长度~~~)


## 八、操作符重载的限制

* 不是所有的操作符都能重载，以下操作符不能重载
    - 作用域限定操作符(::)
    - 直接成员访问操作符(.)
    - 直接成员指针解引用操作符(.*)
    - 条件操作符(?:)
    - 字节长度操作符(sizeof)
    - 类型信息操作符(typeid)
* 无法重载所有操作数均为基本类型的操作符
    - `1 + 1 = 8 ？`
* 无法通过操作符重载改变操作符的优先级
* 无法通过操作符重载改变操作数的个数
    - `50%  --> 0.5  ?`
* 无法通过操作符重载发明新的操作符
    - `x**y  -->  x ^y ？`
* 操作符重载着力于对一致性的追求
    -`(3+4i)+(1+2i) = 2+2i ?`
* 操作符重载的价值在于提高代码的可读性，而不是沦为少数 `算符控` 们赖以卖弄的奇技淫巧
* 匿名对象或变量都是具有常属性的。

