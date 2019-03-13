# Initialization并不只是Constructor

## 问题背景
我们是一个使用C++的游戏服务器项目，由于C++的特性，我们需要将业务的逻辑跟状态部分分离并将状态部分存放在共享内存，以保证在服务器出现coredump时能够迅速重新启动并且在共享内存的基础上恢复原有的服务状态。

但是我们对于这些记录状态的class有一个奇怪的要求：必须写一个空的默认构造函数。

## 问题分析
经了解，这个要求的来源是对数据恢复流程的一次修改引出的。

我们记录状态的class都是非POD的，有虚表存在。而在重启恢复状态时，这些虚表指向的地址不一定对（尤其是代码进行了重编之后，新进程的虚拟内存分布肯定不一样了），所以需要通过placement new来递归调用一遍构造函数，以此保证所有class实例的正确性。

而某次修改的时候，把这段代码加了一对()。

```cpp
new (pDescTmp) DESCCLASSNAMENOC
```

修改为

```cpp
new (pDescTmp) DESCCLASSNAMENOC()
```
这个修改导致没有提供user-defined默认构造函数的class的实例，在恢复时成员变量数据全部被清0.当时他们用GDB调试时发现，gcc编译器在后面那段代码执行合成的默认构造函数时，插入了memset从而导致数据清零，而有user-defined的默认构造函数则不会插入memset。

## 解释
这个行为很让我困惑，因为资料上对自动合成构造函数的描述都说，它的执行效果跟user-defined的空构造函数一致(it has exactly the same effect as a user-defined constructor with empty body and empty initializer list)。

带着困惑，我去翻了翻cppreference网站跟C++标准，才发现变量在构造过程中并不是单纯调用Constructor来对class提供初始值，而是通过Initialization来提供初始值，而如何调用Constructor以及在调用同时进行哪些其他操作，是用Initialization的分类来决定的。

```cpp
T()
new T ()
```

上面这两者是value initialization，而

```cpp
new T
```

是default initialization。

按照标准value initialization会对所有类型为class/array 的成员变量递归执行value initialization，而对于基础类型执行zero initialization，然后调用默认构造函数。

而default initialization会直接调用默认构造函数。

虽然标准里对placement new时如何区别initialization分类没有做出规定（也许有隐藏，但是我们没找到，而且Stack Overflow上的人也是这么说的)，但是相信gcc实现时应该是参考new operator的部分。
