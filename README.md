# C_CPP_Undefind_Behavior

未定义行为(Undefind Behavior)：是指 C/C++ 标准中没有定义的行为，程序执行未定义行为的结果完全不可预测。

下面将详细列举 C/C++ 中常见的未定义行为，一些个人难以理解的会作 [_number] 标记并做详细的解释。

--------------------------------------------------------------------------------------------------------------------------------------------

## C 常见的未定义行为

(1) 访问数组越界

[2] 访问空指针，或悬空指针。

(3) 访问未初始化的变量。

[4] 修改字符串的字面量。（或者叫只读内存区域）  

[5] 返回局部变量的地址 （编译器会发现这个问题。 function returns address of local variable）。

(6) 类型转换，如 double->int 导致的截断。

(7) 除数为 0 （如 7 / 0 会触发未定义行为，但是现代的编译器都会发现这个问题。 warning: division by zero）

(8) 调用未实现的函数 （和 (7) 一样，现代编译器也会捕获这个问题。 undefined reference to 'function_name'）

[9] 对同一变量进行两次赋值的序列点 （啥玩意？）。

--------------------------------------------------------------------------------------------------------------------------------------------

### [2] 访问空指针，或悬空指针

```C

#include <stdio.h>
#include <stdlib.h>

int main(int argc, char const *argv[])
{
    int *ptr = (int *)malloc(10 * sizeof(int));

    free(ptr);          /*此时，这个指针以及被释放， ptr 为悬空指针*/

    *ptr = 1000;        /*如果继续访问该指针，会触发未定义行为，导致程序奔溃*/

    return 0;
}
```

所以需要这样修改：

```C
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char const *argv[])
{
    int *ptr = (int *)malloc(10 * sizeof(int));

    free(ptr);          /*此时，这个指针以及被释放， ptr 为悬空指针*/

    *ptr = NULL;        /*将已经被释放的指针的值设置为 NULL*/

    return 0;
}
```

--------------------------------------------------------------------------------------------------------------------------------------------

### [4] 修改字符串的字面量。（或者叫只读内存区域）  

```C
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char const *argv[])
{
    char str[10] = "hello";     /*此时字符串 hello 是保存在只读空间的，不可修改。*/

    str[0] = 'H';               /*强行修改该字符串会触发未定义行为，结果不可预测*/

    return 0;
}
```

所以需要这样做：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char const *argv[])
{
    char str[5] = "hello";     /*此时字符串 hello 是保存在只读空间的，不可修改。*/
    char _str[5];

    strcpy(_str, str);          /*将字符串拷贝到新字符串*/

    _str[0] = 'H';              /*然后再进行修改*/

    /*当然使用 const 关键字也是可以的。*/
    const char str[5] = "hello";
}
```

--------------------------------------------------------------------------------------------------------------------------------------------

### [5] 返回局部变量的地址

```C
#include <stdio.h>
#include <stdlib.h>

int *function();

int *function()
{
    int number = 10;           /*在栈空间声明了临时变量 作用域为这个函数*/

    return &number;             /*返回这个变量的地址，但是这个变量已经被销毁。。。。*/
}

int main(int argc, char const *argv[])
{
    int *ptr = function();

    printf("%d", *ptr);         /*如果强行访问这个地址，结果是不可预测的。*/

    return 0;
}
```

所以需要这样修改：

```C
#include <stdio.h>
#include <stdlib.h>

int *global_ptr = NULL;

int *function();

int *function()
{
    global_ptr = (int *)malloc(sizeof(int));    /*在使用 malloc 函数堆空间里面申请内存*/
    static number = 0;                          /*或者使用 static 关键字，改变变量的存储方式和作用域*/

    *global_ptr = 0;

    return global_ptr;             /*返回这个指针变量*/
}

int main(int argc, char const *argv[])
{
    int *ptr = function();

    printf("%d", *ptr);         /*此时访问这个地址，结果是 0*/

    free(global_ptr);

    return 0;
}
```

--------------------------------------------------------------------------------------------------------------------------------------------

## C++ 常见的未定义行为

(1) 修改 const 对象

(2) 违反类的访问规则（直接用指针偏移去访问类的私有成员）

[3] 引用删除后的对象

(4) 类型名冲突

(5) 读取未定义的位域

[6] 重入非重入的函数

(7) 引用未初始化的变量 (int a; int &b = a;)

(8) 虚函数表类型错误

(9) 使用无效的虚函数指针

[10] 多线程数据竞争

[11] 违反 One Defination Rule (ODR)

[12] 异常规范不匹配

[13] 没有正确的使用 new 和 delete

--------------------------------------------------------------------------------------------------------------------------------------------

### [3] 引用删除后的对象

首先了解一下引用和指针的几个区别：

(1) 引用必须在初始化时绑定到一个对象,之后不能修改为指向另一个对象。指针可以在任何时候指向不同的对象。

(2) 引用在语法上更像对象本身,可以直接访问对象成员。指针需要解引用才能访问所指对象。

(3) 引用不能为空(nullptr)，指针可以为空。

```C++
/*test.h*/
/*篇幅需要，大部分成员方法的实现都是内联的。*/
#include <iostream>
class My_Class
{
    private:
        int value;

    public:
        My_Class()  { value = 0; }
        My_Class(int _val) { value = _val; }

        void show_value() {
            std::cout << value << std::endl;
        }

        ~My_Class() {}
};

/*Test.cpp*/
#include "./test.h"

My_Class & function(int _val);      /*一个返回 My_Class 类引用的函数*/

My_Class & function(int _val)
{
    My_Class my_class(_val);

    return my_class;            /*返回 my_class的引用，但此时这个对象已经被销毁。。。。*/
}
    
int main(int argc, char const *argv[])
{
    My_Class &class_ref = function(11);     /*声明一个 My_Class 类型的引用，接受函数 function 的返回值*/

    std::cout << class_ref.show_value();    /*但由于那个对象已经被销毁，如果强行访问，结果不可预测。*/
}
```

当然修改的方式也很简单，使用 static 关键字改变存储方式和作用域，或者使用 new 操作符在堆(heap)中申请内存即可。

--------------------------------------------------------------------------------------------------------------------------------------------

### [6] 重入非重入的函数

重入函数 (Reentrant function)，是指可以同时被调用多次的函数，比如线程安全函数就是重入的。

非重入函数 (Non-Reentrant function)，是指不能同时被多次调用，需要单线程串行执行。

调用一个非重入函数的同时，再次调用他本身，就会产生未定义行为。

```C++
int rand();     /*rand 是一个全局函数，非重入不安全*/

int func()
{
    return rand() - rand();     /*两次调用之间，全局状态被改变，结果不安全。*/
}
```

修改方法也很简单 使用 `thread_local` 关键字，使用本地线程存储。

```C++
thread_local int t_rand();

int t_func()
{
    return t_rand() - t_rand();     /*使用了  thread_local 关键字后，每一个线程都有独立的 t_rand()*/
}
```

--------------------------------------------------------------------------------------------------------------------------------------------

### [11] 违反 One Defination Rule

One Defination Rule (ODR) 是关于程序中函数、对象等实体定义的一个重要规则，它可以概括为:

整个C++程序中,任何实体(函数、对象、类等)的定义只能有且只能有一个。（也就是说,一个实体在全局命名空间中只能定义一次,不允许重复定义。）

这个规则确保了C++程序中的实体在语义上都是唯一确定的,不会出现歧义。

#### 一些 One Definition Rule 的关键点

1.函数、全局变量、静态成员变量等在程序生命周期内只能定义一次。

2.非内联函数的定义只能出现在一个编译单元(.cpp文件)中。

3.类的定义通常应该只出现在头文件中,不要在多个文件中重复定义。

4.违反ODR会导致链接错误、运行时异常等严重问题。

5.模板和内联函数有一些特殊规则,可以通过实例化导致多次定义。

6.命名空间、本地变量等存在多个定义是允许的。

--------------------------------------------------------------------------------------------------------------------------------------------

### [10] 多线程数据竞争

在 C++ 多线程编程中，会出现多个进程同时访问同一片数据，由于线程切换的不确定性，会导致数据异常。

#### 常见的线程竞争有

1.多个线程在同一段时间读取和写入共享数据,由于代码执行顺序不确定,导致最终结果无法预测。

2.死锁(Deadlock):多个线程互相持有对方需要的锁,形成了循环等待。

3.饥饿(Starvation):一个线程由于优先级太低,始终得不到CPU调度执行,也无法访问共享资源。


接下来用代码来详解上述三条：

##### 多个线程在同一段时间读取和写入共享数据,由于代码执行顺序不确定,导致最终结果无法预测

```C++
#include <iostream>
#include <thread>

int global_data = 0;

void thread_func();

/*
    一个函数，在没有锁的情况下访问 global_data，对这个数据进行累加
*/
void thread_func()                      
{
    for (int i = 0; i < 1000000; ++i)
    {
        ++global_data;
    }
}

int main(int argc, char const *argv[])
{
    /*创建 3 个进程*/
    std::thread th_1(thread_func);      
    std::thread th_2(thread_func);
    std::thread th_3(thread_func);

    th_1.join();        /*等待 3 个进程结束*/
    th_2.join();
    th_2.join();

    std::cout << global_data << '\n';       //相信我，在没有锁的情况下，出来的结果会很奇怪~
}
```

当然，修改的方法也很简单，发一把公共的锁，只有某个进程持有这把锁的时候，才准许访问这片数据。
此时，就需要引入 mutex 头文件， 修改的代码如下：

```C++
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;             /*一把公共的锁*/

int global_value = 0;

void thread_01()
{
    for (int i = 0; i < 10000000; i++)
    {
        /*
            当进程需要修改数据的时候，先给他上把锁，
            确保只有这个进程可以修改数据。
        */
        mtx.lock();         
        ++global_value;     /* 修改数据 */

        /*
            数据修改结束后，把锁放回去，让别的进程来拿锁。
        */
        mtx.unlock();
    }
}

int main(int argc, char const *argv[])
{
    /*创建 3 个进程*/
    std::thread th_1(thread_01);
    std::thread th_2(thread_01);
    std::thread th_3(thread_01);

    th_1.join();    /*等待 3 个进程结束*/
    th_2.join();
    th_3.join();

    std::cout << global_value << '\n';  /*最后的结果是正确的 30000000*/

    return EXIT_SUCCESS;
}
```

##### 死锁(Deadlock):多个线程互相持有对方需要的锁,形成了循环等待

```C++
#include <fstream>
#include <iostream>
#include <mutex>
#include <thread>
#include <string>

using namespace std;

class LogFile
{
    private:
        std::mutex _mu;
        std::mutex _mu2;
        ofstream f;

    public:
        LogFile()
        {
            f.open("log.txt");
        }
        ~LogFile()
        {
            f.close();
        }
        void shared_print(string msg, int id)
        {
            std::lock_guard<std::mutex> guard(_mu);
            std::lock_guard<std::mutex> guard2(_mu2);
            f << msg << id << endl;
            cout << msg << id << endl;
        }
        void shared_print2(string msg, int id)
        {
            std::lock_guard<std::mutex> guard(_mu2);
            std::lock_guard<std::mutex> guard2(_mu);
            f << msg << id << endl;
            cout << msg << id << endl;
        }
};

void function_1(LogFile &log)
{
    for (int i = 0; i > -100; i--)
        log.shared_print2(string("From t1: "), i);
}

int main(int argc, char const *argv[])
{
    LogFile log;
    std::thread t1(function_1, std::ref(log));

    for (int i = 0; i < 100; i++)
        log.shared_print(string("From main: "), i);

    t1.join();

    return EXIT_SUCCESS;
}
```

上面线程 A 先获取了锁_mu，线程 B 获取了锁_mu2，进而线程 A 还需要获取锁_mu2才能继续执行，但是由于锁_mu2被线程 B 持有还没有释放，线程 A 为了等待锁_mu2资源就阻塞了。
线程 B 这时候需要获取锁_mu才能往下执行，但是由于锁_mu被线程A持有，导致 B 也进入阻塞，这就是死锁现象。

当然，解决死锁也不难，使用 std::lock() 就能够保证将多个互斥锁同时上锁，代码如下：

```C++
#include <iostream>
#include <thread>
#include <string>
#include <mutex>
#include <fstream>

using namespace std;

class LogFile
{
    std::mutex _mu;
    std::mutex _mu2;
    ofstream f;

public:
    LogFile()
    {
        f.open("log.txt");
    }
    ~LogFile()
    {
        f.close();
    }
    void shared_print(string msg, int id)
    {
        std::lock(_mu, _mu2);
        std::lock_guard<std::mutex> guard(_mu, std::adopt_lock);
        std::lock_guard<std::mutex> guard2(_mu2, std::adopt_lock);
        f << msg << id << endl;
        cout << msg << id << endl;
    }
    void shared_print2(string msg, int id)
    {
        std::lock(_mu, _mu2);
        std::lock_guard<std::mutex> guard(_mu2, std::adopt_lock);
        std::lock_guard<std::mutex> guard2(_mu, std::adopt_lock);
        f << msg << id << endl;
        cout << msg << id << endl;
    }
};

void function_1(LogFile &log)
{
    for (int i = 0; i > -100; i--)
    {
        log.shared_print2(string("From t1: "), i);
    }
}

int main(int argc, char const *argv[])
{
    LogFile log;
    std::thread t1(function_1, std::ref(log));

    for (int i = 0; i < 100; i++)
        log.shared_print(string("From main: "), i);

    t1.join();

    return EXIT_SUCCESS;
}
```

#### 饥饿(Starvation):一个线程由于优先级太低,始终得不到CPU调度执行,也无法访问共享资源

这玩意模拟不出来的吧。。。。。

--------------------------------------------------------------------------------------------------------------------------------------------

[13] 没有正确的使用 new 和 delete

在 《C++ Primer Plus》中，对于这两个关键字的的使用给出了如下意见：
1）若使用 new 来初始化指针，则务必使用 delete 去释放。

2）new 和 delete 必须相互兼容，new 对应 delete，new[] 对应 delete[]

3）在类构造函数中，必须用相同的方式使用 new，因为只有一个析构函数，所有的构造函数都必须与它兼容。

4）不要去 delete 去删除没有用 new 关键字分配内存的指针，这会造成未知的错误。

下面举一个错误的例子：

```C++
int main(int argc, char const *argv[])
{
    int *array = new int[100];                /*使用 new 关键字开辟了由 100 个 int 类型组成的数组，并返回它的首地址给指针变量 array*/

    delete array;                            /*但这里的 delete 使用错误，应该使用 delete[] 去释放 array*/

    return EXIT_SUCCESS;
}
```

Date:2023.09.18

Author: Jesse_EC
