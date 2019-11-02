---
title: 纯虚函数调用crash原因
date: 2019-11-03 01:18:01
tags: c++
---
写个小程序，构造函数中调用纯虚函数，然后从汇编角度看崩溃原因  
<!--more-->
```C++
#include <iostream>
using namespace std;
class A;
class B;
class C;
class D;

B* b = NULL;
C* c = NULL;
D* d = NULL;
class A{
    public:
        A() {}
        virtual void func() = 0;
};
class D : public A{
    public:
        D() { d = this; }
    private:
        int d1;
};
class B{
    public:
        B() {
            cout << this << endl;
            b = this;
            d->func(); //crash address
        }
        void func() {
            cout << "this is b " << this << endl;
        }
    private:
        int b1;
};
class C : public D, public B{
    public:
        C() {
            c = this;
            this->func();
            cout << this << endl;
        }
        void func() {
            cout << "this is c " << this << endl;
        }
    private:
        int c1;
};
int main()
{    C c;
}
```

上面程序C继承D类和B类， D类继承抽象A类  

在main函数，构造C类。在C类构造过程中如下：  

Class A --> Class D --> Class B --> Class C  


### 抽象类A构造

打印抽象类A的虚函数表指针：\_vptr.A=0x400c70  
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/a2903e98-b32b-4edc-8d59-371843c93707.png"/>
</center>


### D类构造
D类构造过程中把D对象的this指针赋值给变量d  

### B类构造
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/2bf3b15a-58f0-4cd4-ae92-521409a8327d.png"/>
</center>
- 在B类构造函数用
- B对象构造函数中调用d->func(), 汇编代码已经打出来了

```C++
mov 0x20170a(%rip), %rax  ;获取全局变量d地址 保存到寄存器 rax中
mov (%rax),%rax
mov (%rax),%rax           ;取虚函数表第一个函数地址, 保存到寄存器 rax中
mov 0x2016fd(%rip),%rdx   
callq *%rax               ；调用rax寄存器中保存到函数地址
```

<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/7892275f-aa3f-4748-9c04-8f957d2cf933.png"/>
</center>
看下rax寄存器内容，rax=0x7ffffffd880  



<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/7892275f-aa3f-4748-9c04-8f957d2cf933.png"/>
</center>
- 看下d变量，d=0x7ffffffd880, 跟寄存器rax保存到值一样  
- 看下d变量地址，d=0x6022d8，跟上面到汇编代码的注释一样  


<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/bd9d75ea-7721-4f9c-94b8-c948892acc91.png"/>
</center>
- 打印下d对象虚函数表指针指向的虚函数表地址， vtable_address=0x400d28, 注意是D类的虚函数表地址, 也是A类的虚函数表地址


<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/5cf93891-765d-451a-8909-9f4c3f2f72f8.png"/>
</center>
- 打印下vtable_adress=0x400d28对应下的虚函数地址， func_address=0x400950，显然纯虚函数地址并没有等于0，它指向的地址是随机的 

<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/3ecd24d5-e7f7-4702-a462-0c1152a57763.png"/>
</center>
- 继续单步执行汇编，顺利调用func_address=0x400950的函数，函数来源于抽象类A的虚函数表
- 往下的堆栈就看不懂了

