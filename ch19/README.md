# 19.1 控制内存分配

## 19.1.1 重载new和delete

``` cpp
string *sp = new string("a value"); // 分配并初始化一个string对象
string* arr = new string[10]; // 分配10个默认 初始化 的string对象
```

1. 调用 **operator new / operator new[]** 标准库函数
    分配足够大的、原始的、未命名的内存空间便于存储特定类型的对象/特定类型的对象数组

2. 编译器运行相应的构造函数以构造这些对象，并为其传入初始值
3. 对象被分配了空间并构造完成，返回一个指向该对象的指针



```cpp
delete sp; // 销毁*sp，然后释放sp指向的内存空间
delete[] arr; // 销毁数组中的元素，然后释放对应的内存空间
```

1. 对sp所指的对象或者arr所指的数组中的元素执行相应的析构函数
2. 编译器调用名为 **operator delete / operator delete[]** 标准库函数释放内存空间



如果希望控制内存分配过程，则需要定义自己的 operator new 函数和 operator delete 函数。即使在标准库中已经存在这两个函数的定义，仍可以定义自己的版本。编译器不会对重复定义提出报错，相反会使用自定义版本替换标准库定义版本。



> Warning: 自定义全局的 **operator new** 函数和 **operator delete** 函数后，就担负起了控制动态内存分配的职责，这两个函数必须是正确的。因为它们是程序整个处理过程中至关重要的一部分。



可以在全局作用域中定义 operator new 函数和 operator delete 函数，也可以将它们定义为成员函数。编译器发现一条 new 表达式或 delete 表达式后，将在程序中查找可供调用的 operator 函数。如果被分配（释放）的对象是类类型，则编译器首先在基类的作用域中查找。如果此类含有 operator new 成员或 operator delete 成员，则相应的表达式将调用这些成员。否则，编译器在全局作用域查找批评的函数。此时如果编译器找到了用户自定义的版本，则使用该版本执行 new 表达式或 delete 表达式；如果没找到，则使用标准库定义的版本。

1. 遇到 operator new 或 operator delete
2. 如果对象是类类型，首先在基类作用域中查找
3. 否则在全局作用域中查找
4. 如果没找到则使用标准库定义的版本

当然，可以使用作用域运算符令 new 表达式或 delete 表达式忽略在类中的函数，直接执行全局作用域的版本。如 ::new 只在全局作用域中查找匹配的 operator new 函数，::delete 只在全局作用域中查找匹配的 operator delete 函数。



**operator new 接口和 operator delete 接口**

标准库定义了 operator new 函数和 operator delete 函数的 8 个重载版本，其中前 4 个版本可能抛出 bad_alloc 异常，后 4 个版本则不会抛出异常

```cpp
// 这些版本可能抛出异常
void* operator new(size_t);              // 分配一个对象
void* operator new[](size_t);            // 分配一个数组
void operator delete(void*) noexcept;   // 释放一个对象
void operator delete[](void*) noexcept; // 释放一个数组

// 这些版本承诺不会抛出异常，参见 12.1.2 节（第409页）
void* operator new(size_t, nothrow_t&) noexcept;
void* operator new[](size_t, nothrow_t&) noexcept;
void operator delete(void*, nothrow_t&) noexcept;
void operator delete[](void*, nothrow_t&) noexcept;
```

类型nothrow_t 是定义在 new 头文件中的一个 struct，这个类型中不包含任何成员。
new 头文件还定义了一个名为 nothrow 的 const 对象，用户可以通过这个对象请求 new 的非抛出版本。与析构函数类似，operator delete 也不允许抛出异常。当我们重载这些运算符时，必须使用 noexcept 异常说明符指定其不抛出异常。

应用程序可以自定义上面函数版本的任意一个，前提是自定义版本必须位于全局作用域或类作用域中。当我们将上述运算符函数定义成类的成员时，它们是隐性静态的，无需显式声明 static ，但是声明了也没问题。因为 operator new 用在对象构造之间，而 operator delete 用在对象销毁之后，所以这两个成员 （new 和 delete）必须是静态的，而且它们不能操纵类的任何成员。

- 可以自定义任意一个上述 8 个函数
- 自定义版本只能在全局作用域或者类作用域中
- 自定义上述函数是隐性静态的
- 因为在使用 operator new 时还未构造对象，使用 operator delete 时对象已经被销毁，所以这两个成员在泪中必须是静态的，而且它们不能操纵类的任何成员（调用它们的时候类的成员都不存在了）



对于 operator new 函数和 operator new[] 函数，返回类型必须是 void* ，第一个形参类型必须是size_t，且该形参不能有默认实参。当对一个对象分配空间时使用 operator new；为一个数组分配空间时使用 operator new[]。当编译器调用 operator new 时，把存储指定类型对象所需字节数传给 size_t形参，当调用 operator new[]时，传入函数的是存储数组中所有元素所需的空间。

- operator new / operator new[] 的第一个形参类型必须是 size_t，且size_t不能有默认实参。
- 对一个对象分配空间时使用 operator new，为一个数组分配分配空间使用 operator new[]
- 调用operator new传入的是存储对象需要的字节数，调用operator new[]传入的是存储数组所有元素所需的字节数。

想要自定义 operator new 函数，可以为其提供额外的形参。此时，用到这些自定义函数的new表达式必须使用new的定位形式（参考12.1.2节，第409页）将实参传给新增的形参。尽管可以自定义具有任何形参的operator new，但是下面这个函数不能被用户重载：

`void operator new(size_t, void*);`

这种形式只供标准库使用，不能被用户重新定义。



对于 operator delete 函数和operator delete[] 函数，它们的返回类型必须是void，第一个形参的类型必须是void\*，执行一条delete表达式将调用相应的operator函数，并用指向待释放内存的指针来初始化 void\*形参。当我们将operator delete 或 operator delete[] 定义为类的成员时，该函数可以包含另外一个类型为 size_t的形参。此时，该形参的初始值是第一个形参所指对象的字节数。size_t形参可用于删除继承体系中的对象。如果基类又一个虚析构函数，则传递给operator delete的字节数将因待删除指针所指对象的动态类型不同而有所区别。而且，实际运行的operator delete函数版本也由对象的动态类型决定。

- operator delete / operator delete[]的返回类型必须是void，第一个形参必须是void*
- 执行一个delete表达式会调用对应的operator函数，使用待释放内存的指针来初始化void*形参。
- 将operator delete或operator delete[]定义成类的成员时，该函数可以包含另外一个类型为size_t的形参，这个形参的初始值时第一个形参所指对象的字节数
- size_t形参可用于删除继承体系中的对象。
- 如果基类又一个虚析构函数，则传递给operator delete的字节数将因待删除指针指向对象的动态类型不同而有所区别，实际运行的operator delete函数版本也由对象的动态类型决定

**malloc函数与free函数**

当你定义了自己的全局operator new 和 operator delete后，这两个函数必须以某种方式执行分配内存与释放内存的操作。

malloc函数接受一个表示待分配字节数的size_t，返回指向分配空间的指针或者返回0表示分配失败。free函数接受一个void*，它是malloc返回的指针的副本，free将相关内存返回给系统，调用free(0)没有任何意义

```cpp
void* operator new(size_t size) {
    if (void* mem = malloc(size)) return mem;
    else throw bad_alloc();
}

void operator delete(void* mem) noexcept {
    free(mem);
}
```

