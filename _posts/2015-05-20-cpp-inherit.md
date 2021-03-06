---
layout: post
title:  "C++的继承"
categories: C++
tags: C++
author: 肖邦
---

* content
{:toc}

> C++ 继承




* 继承的基本概念和语法
* 公有继承的基本特点
* 继承方式与访问控制
* 子类的构造与析构
* 子类的拷贝构造与拷贝赋值
* 子类的操作符重载
* 名字隐藏与重载
* 私有继承与保护继承
* 多重继承、钻石继承与虚继承

## 一、继承的基本概念和语法

* 共性与个性
    ```
    人    (行为) 吃饭、睡觉          (属性) 姓名、年龄
    学生  (行为) 吃饭、睡觉、学习     (属性）姓名、年龄、学号
    教师  (行为) 吃饭、睡觉、授课     (属性）姓名、年龄、工资
    ```
* 超集与子集
* 基类与子类
* 继承与派生
* 继承的语法
    ```
    class 子类：继承方式1 基类1，继承方式2 基类2, ... { ... };
    ```
* 继承方式
    ```
    公有继承  -  public
    保护继承  -  protected
    私有继承  -  private
    ```
* 继承与派生的举例
    ```cpp
    class Human {
    public:
        void eat (const string& food) { ... }
        void sleep (int hours) { ... }
        string m_name;
        int m_age;
    };

    /* 学生类继承于Human类 */
    class Student: public Human {
    public:
        void learn (const string& course) { ... }
        int m_no;
    };

    /* 教师类继承于Human */
    class Teacher :public Human {
    public:
        void teach (const string& course) { ... }
        float m_salary;
    };
    ```


## 二、公有继承的基本特点
* 子类对象任何时候都可以被当做基类类型的对象
    - 示例说明
        ```
        class User { ... };                      /* 基类         */
        class StudentUser :public User { ... };  /* 学生类（子类） */
        StudentUser student (...);               /* 实例化子类对象 */
        User* puser = &student;
        User& ruser = student;
        ```
    - 编译器认为 `访问范围缩小` 是安全的。
    - 子类对象的基类子对象所占内存时子类对象的一个子集，将子类对象地址交给基类子对象时，其操作范围是缩小的，只能够操作基类对象的数据，不能够操作子类对象的特有部分，可以将子类对象看作是特殊的基类子对象，所以安全。
    - 从逻辑层面，学生类是从人类派生而来的，学生是特殊的人类，所以可以将学生看作是人类
    - 从范围方面，基类是超集合，子类是子集(条件特殊)
* 基类类型的指针或引用不能隐式转换为子类类型
    - 示例说明
    ```cpp
    class User {...};
    class StudentUser :public User {...};
    User user (...);
    StudentUser* pstudent = static_cast<StudentUser*>(&user);
    StudentUser& rstudent = static_cast<StudentUser&>(user);
    ```
    - 编译器认为 `访问范围扩大` 是危险的。
* 编译器对类型安全的检测仅仅基于指针或引用本身
    - 示例说明
    ```cpp
    class User {...};
    class StudentUser :public User {...};
    StudentUser student (...);
    User* puser = &student;
    User& ruser = student;
    StudentUser* pstudent = static_cast<StudentUser*>(puser);
    StudentUser& rstudent = static_cast<StudentUser&>(ruser);
    ```
    - 基类指针或引用的实际目标，究竟是不是子类对象，完全由程序员自己判断
    - 编译器看指针类型，指针类型决定了它的行为和视野
* 在子类中可以 "直接访问" 基类的所有公有和保护成员，就如同它们是在子类中声明的一样
* 基类的私有成员在子类中虽然存在却不可见，故无法直接访问
* 尽管基类的公有和保护成员在子类中直接可见，但仍然可以在子类中重新定义这些名字，子类中的名字会隐藏所有基类中的同名定义
* 如果需要在子类中或通过子类访问一个在基类中定义却为子类所隐藏的名字，可以借助作用域限定操作符 "::" 实现


## 三、继承方式与访问控制
* 类成员的访问控制限定符与访问控制属性。
* 基类中的公有、保护和私有成员，在其公有、保护和私有子类中的访问控制属性，会因继承方式而异。
* 当通过子类访问其所继承的基类的成员时，需要考虑继承方式对访问控制属性的影响。


## 四、子类的构造与析构

* **子类构造函数隐式调用基类构造函数**。如果子类的构造函数没有显式指明其基类部分的构造方式，那么编译器会选择其基类的缺省构造函数，构造该子类对象中的基类子对象
* **子类构造函数显式调用基类构造函数**。子类的构造函数可以在初始化表中显式指明其基类部分的构造方式，即通过其基类的特定构造函数，构造该子类对象中的基类子对象。
* 子类对象的构造过程。
    ```
    构造基类子对象 -->  构造成员变量  -->  执行构造代码
    ```
* 子类与基类构造初始化
    - 在子类构造函数中，初始化子类时，不能够将基类的成员变量在子类初始化表中直接初始化，因为毕竟是继承来的，跟自己的不太一样。只能够在构造函数体内以赋值的方式进行初始化。如果子类不能够直接访问基类的私有成员，可以将私有成员改为 protected ，但是这种方法不好，不方便，破坏了基类的封装性。
    - 可以在子类构造函数中的初始化表中显式方式指明基类的构造方式：Human(age,name)
    - 继承不会改变作用域，不存在 "无形的手" 的复制粘贴行为。
* 阻断继承
    - 子类的构造函数无论如何都会调用基类的构造函数，构造子类对象中的基类子对象
    - 如果把基类构造函数定义为 private，那么该类的子对象就无法被实例化为对象
    - 在C++中可以用这种方法阻断一个类被扩展
* **子类析构函数隐式调用基类析构函数**。子类的析构函数在执行完其中的析构代码，并析构完所有的成员变量以后，会自动调用其基类的析构函数，析构该子类对象中的基类子对象。
* **基类析构函数不会调用子类析构函数**。通过基类指针析构子类对象，实际被析构的仅仅是子类对象中的基类子对象，子类的扩展部分将失去被析构的机会，极有可能形成内存泄露。
* 子类对象的析构的过程。
    ```
    执行析构代码  -->  析构成员变量  -->  析构基类子对象
    ```


## 五、子类的拷贝构造与拷贝赋值
* **子类没有定义拷贝构造函数**。编译器为子类提供的缺省拷贝构造函数，会自动调用其基类的拷贝构造函数，构造该子类对象中的基类子对象
* 子类定义了拷贝构造函数，但没有显式指明其基类部分的构造方式编译，器会选择其基类的缺省构造函数，构造该子类对象中的基类子对象
* 子类定义了拷贝构造函数，同旪显式指明了其基类部分以拷贝方式构造，子类对象中的基类部分和扩展部分一起被复制
* 子类没有定义拷贝赋值运算符函数。编译器为子类提供的缺省拷贝赋值运算符函数，会自动调用其基类的拷贝赋值运算符函数，复制该子类对象中的基类子对象
* 子类定义了拷贝赋值运算符函数，但没有显式调用其基类的拷贝赋值运算符函数
* 子类定义了拷贝赋值运算符函数，同时显式调用了其基类的拷贝赋值运算符函数。子类对象中的基类部分和扩展部分一起被复制
* 拷贝赋值举例：
    ```cpp
    Teacher& operator= (const Teacher& that) {
        if (&that != this) {           /* 防止自赋值，自赋值时什么都不做 */
            Human::operator= (that);   /* 手动调用基类的拷贝赋值，不要递归调用 */
            m_salary = that.m_salary;  /* 只需拷贝子类特有的成员变量 */
        } /* 基类中会有默认的拷贝赋值函数，不受构造函数的影响 */
    }     /* 在子类中调用基类的拷贝赋值函数，可以实现基类子对象的拷贝赋值 */
    ```


## 六、子类的操作符重载
* 在为子类提供操作符重载定义时，往往需要调用其基类针对该操作符所做的重载定义，完成部分工作
* 通过将子类对象的指针或引用向上造型为其基类类型的指针或引用，可以迫使针对基类的操作符重载函数在针对子类的操作符重载函数中被调用
* 举例： (注意使用友元) 向上造型
    ```cpp
    friend ostream& operator<< (ostream& os,const Manager& manager) {
        return os << (Employee&)manager << ", "<< manager.m_title;
    }
    ```


## 七、名字隐藏与重载
* 继承不会改变类成员的作用域，基类的成员永远都是基类的成员，并不会因为继承而变成子类的成员
* 因为作用域的不同，分别在子类和基类中定义的同名成员函数(包括静态成员函数)，并不构成重载关系，相反是一种隐藏关系，除非通过using声明将基类的成员函数引入子类的作用域，形成重载。隐藏是存在的，但是看不见。  S1.Human::who ()  // 基类隐藏函数用法
* 任何时候，无论在子类的内部还是外部，总可以通过作用域限定操作符 "::"，显式地调用那些在基类中定义即为子类所隐藏的成员函数。
* 初始化的 2 种方式：
    ```cpp
    int a = 12;   // 初始化 a 为 12
    int b = a;    // 将b初始化为 a
    int b (a);    // 等效于方式1的初始化方式
    ```
* 使用 using 改变作用域
    ```
    using Base::foo;    /* 将基类的foo引入当前作用域，实现了作用域的改变 */
    using  将一个作用域中的名称引用到另外一个作用域中去
    s1.Human::foo();    /* 调用基类被隐藏的名称 */
    ```


## 八、私有继承和保护继承
* 私有继承亦称实现继承，旨在于子类中将其基类的公有和保护成员私有化，既禁止从外部通过该子类访问这些成员，也禁止在该子类的子类中访问这些成员
* 保护继承是一种特殊形式的实现继承，旨在于子类中将其基类的公有和保护成员进行有限的私有化，只禁止从外部通过该子类访问这些成员，但并不禁止在该子类的子类中访问这些成员
* 私有子类和保护子类类型的指针或引用，不能隐式转换为其基类类型的指针或引用
* 私有继承：为了防止某些类中的共有和保护接口因为继承而扩散



## 九、多重继承、钻石继承与虚继承
* 一个类可以同时从多个基类继承实现代码
    - 课堂教学系统
    - 交通工具系统
    - 零件装配系统

* 多重继承的内部布局和类型转换
    - 子类对象中的多个基类子对象，按照 "继承表" 的顺序依次被构造，并 "从低地址到高地址" 排列，析构的顺序与构造严格 "相反"
    - 将继承自多个基类的子类类型的指针，隐式或静态转换为它的基类类型，编译器会根据各个基类子对象在子类对象中的内存布局，进行适当的偏移计算，以保证指针的类型与其所指向目标对象的类型一致。
    - 反之，将该子类的任何一个基类类型的指针静态转换为子类类型或另一个基类类型，编译器同样会进行适当的偏移计算。
    - 无论在哪个方向上，重解释类型转换 (reinterpret_cast) 都不会进行任何偏移计算
    - 引用的情况与指针类似，因为引用的本质就是指针。
    - 多重继承举例（指针的相互转换）
        ```cpp
        SmartPhone sp("13901442457","MP4","Android");
        Phone* phone = &sp;        /*  00  */
        Player* player = &sp;      /*  04  */
        Computer* computer = &sp;  /*  08  */
        // 将子类指针赋值给基类子对象，可以隐式转换。
        // 赋值过程中会发生类型转换的，会有一定的偏移量计算的，保证指针指向正确的基类子对象

        /*psp = 00,自动反向偏移，由基类指针向子类指针转换 */
        SmartPhone* psp = static_cast<SmartPhone*>computer;

        /* psp = 08,不会改变指针的值，只是做强制类型转换，重解释不考虑类型 */
        psp = reinterpret_cast<SmartPhone*>computer;  
        /* 由基类指针向另一个基类指针转换，结果未知？？？ */
        player = (PLayer*)computer; /* 暂时就强制转换吧！ */
        ```
* 围绕多重继承，历来争议颇多
    - 现实世界中的实体本来就具有同时从多个来源共同继承的特性，因此多重继承有助于面向现实世界的问题域直接建立逻辑模型
    - 多重继承可能会在大型程序中引入令人难以察觉的 BUG，并极大地增加对类层次体系进行扩展的难度

* 名字冲突问题
    - 如果在子类的多个基类中，存在同名的标识符，而且子类又没有隐藏该名字，那么任何试图在子类中，或通过子类对象访问该名字的操作，都将引发歧义，除非通过作用域限定操作符 "::" 显式指明所属基类

* 钻石继承问题
    - 一个子类继承自多个基类，而这些基类又源自共同的祖先，这样的继承结构称为钻石继承（菱形继承）。
    - 派生多个中间子类的公共基类子对象，在继承自多个中间子类的汇聚子类的对象中，存在多个实例
    - 在汇聚子类中，或通过汇聚子类对象，访问公共基类的成员，会因继承路径的不同而导致不一致
    - 通过虚继承，可以保证公共基类子对象在汇聚子类对象中，仅存一份实例，且为多个中间子类子对象所共享。

* 虚继承、虚基类、虚表、虚表指针
    - 在继承表中使用 virtual 关键字
    - 位于继承链"最末端"的子类的构造函数负责构造虚基类子对象
    - 虚基类的"所有"子类（无论是直接的还是间接的）都必须在其构造函数中"显式指明"该虚基类子对象的"构造方式"，否则编译器将选择以"缺省方式"构造该子对象
    - 虚基类的所有子类（无论直接的还是间接的）都必须在其拷贝构造函数中显式指明以拷贝方式构造其基类子对象，否则编译器将选择以缺省方式构造该子对象。
    - 与构造函数和拷贝构造函数的情况不同，无论是否存在虚基类，"拷贝赋值运算符函数"的实现没有区别。
    - 汇聚子类对象中的每个中间子类子对象都持有一个虚表指针,该指针指向一个被称为虚表的指针数组的中部,该数组的高地址侧存放虚函数指针,低地址侧则存放所有虚基类子对象相对于每个中间子类子对象起始地址的偏移量。
    - 某些C++实现会将虚基类子对象的绝对地址直接存放在中间子类子对象中,而另一些实现(比如微软)则提供了单独的虚基类表,但它们的基本原理都一样。
    - 包含虚继承的对象模型