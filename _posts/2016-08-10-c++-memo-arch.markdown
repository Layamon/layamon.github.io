---
layout: post
title: C++内存管理
date: 2016-08-10 15:58
header-img: "img/head.jpg"
categories: 
    - C++
---

* TOC
{:toc}

# C++内存布局

理解每一门语言其运行时的状态的关键一步就是了解该语言运行时的内存布局。比如学习Java就要了解JVM运行时结构；这篇文章简单讲述C++运行时的内存布局，便于初学者对C++有一个概况的了解。

从如下一个简单的代码开始分析，来了解C++的结构。

```c
#include <stdio.h> 
int main(void) 
{ 
	return 0; 
}
```

通过`size`命令，我们知道文件的布局如下所示，分为三部分：text/data/bss。当代码运行起来，这就是全局区域，另外加上栈和堆就是一个应用进程的整体布局。

```bash
» size a.out
   text    data     bss     dec     hex filename
   1127     540       4    1671     687 a.out
```

## text

text就是代码段。首先我们需要了解代码是如何编译生成的？如[下图](https://www3.ntu.edu.sg/home/ehchua/programming/cpp/gcc_make.html)所示，从源码到机器指令代码中间有4个步骤：

1. 预处理：宏展开，引用头文件
2. 编译：转成汇编代码
3. 汇编：将汇编代码转成机器指令
4. 链接：将多个目标文件和库文件进行链接，生成一整个可执行文件。

![img](../image/GCC_CompilationProcess.png)

其中在第4步中，如果是静态库那么每个可执行文件中都会有一个静态库的代码拷贝。这样最终的可执行文件就会比较臃肿；因此，提出了另一个动态（共享）库的链接方式，基于动态库的链接并不会将库代码整合到可执行文件中，而是在可执行文件执行的时候，如果调用了动态库，那么才进行加载。这样可执行文件比较小，并且可以动态更新模块，更加灵活；但是由于具体调用的时候需要对函数入口进行相对寻址，这样效率上比静态库会慢一些；而且动态库是在系统中单独存放，存在被人误删等异常操作，从而影响可执行文件的稳定。

### 动态连接库的系统共享

比如libc.so就是一个常用的动态库，该动态库被系统大多数进程共享使用。在链接过程中，可执行文件中会创建一个**符号映射表**。在app执行的时候，OS将控制权先给`ld.so`，而不是先给app。`ld.so`会找到并引入的lib；然后将控制权转给app。

> ld搜索lib路径先后顺序:
>
> +  DT_RPATH
> +  LD_LIBRARY_PATH (LIBRARY_PATH是静态库的位置，CPLUS_INCLUDE_PATH是头文件的位置)
> +  /etc/ld.so.cache
> +  DT_RUNPATH
> +  /lib(64) or /usr/lib(64)
>
> **undefine-reference错误**：
>
> 在写MakeFile的时候，常常在文件中定义了`LDFLAGS`变量(描述定义了使用了哪些库，以及相应的参数)。但是最后连接的时候，将`*.o`和`LDFLAGS`进行连接，出现undefine-reference问题，因为库中明明有这些符号，但是却没有找到，奇怪了，原因如下：
>
> **在UNIX类型的系统中，编译连接器，当命令行指定了多个目标文件，连接时按照自左向右的顺序来搜索外部函数的定义。也就是说，当所有调用这个函数的目标文件名列出后，再出现包含这个函数定义的目标文件或库文件。否则，就会出现找不到函数的错误，连接是必须将库文件放在引用它的所有目标文件之后**

对于动态库的全局代码段，每个进程维护通过相对寻址来执行。而对于动态库中的非常量全局变量不是共享的，每个进程一个拷贝。

## data

**已经初始化**的全局变量和静态变量。

## bss

> block started by symbol

**没有初始化**的全局变量和静态变量，一般操作系统对其进行置零初始化。

### 类的成员变量初始化

C++的初始化和赋值是两码事，引用类型以及const类型需要在定义的时候初始化。

C++类中的成员变量的初始化是通过构造函数来进行的。如果希望提高初始化的速度可以采用初始化列表（自定义组合类型初始化，减少了一次默认构造函数的调用），但是初始化的顺序依然还是按照**类中出现的先后顺序**进行初始化。另外，C++类中的static类型可看做是全局静态变量，访问需要加上类域。

构造函数分为三种：

#### 默认构造函数

#### 带参数的构造函数

和普通(成员)函数重载类似，只要参数不同，都可重载，是C++多态特性的一部分。

> 但是普通(成员)函数有返回值，对于**只有返回值不同其他都相同的函数不能重载**，比如如下代码编译有问题
>
> ```c++
> #include<iostream>
> class A
> {
> public:
> 	A (){};
> 	virtual ~A () {};
> 
> 	int foo(int a) {
> 	    return 10;
> 	}
> 
> 	char foo(int a) { // compiler error; new declaration of foo()
> 	    return 'a';
> 	}
> private:
> 	/* data */
> };
> 
> 
> int main()
> {
> 	A a;
>     char x = a.foo();
>     getchar();
>     return 0;
> }
> a.cpp:12:7: error: functions that differ only in their return type cannot be overloaded
>         char foo(int a) { // compiler error; new declaration of foo()
>         ~~~~ ^
> a.cpp:8:6: note: previous definition is here
>         int foo(int a) {
>         ~~~ ^
> a.cpp:23:20: error: too few arguments to function call, single argument 'a' was not specified
>     char x = a.foo();
>              ~~~~~ ^
> a.cpp:8:2: note: 'foo' declared here
>         int foo(int a) {
> ```
>
> 因为在C++最后编译的函数符号有函数名和参数拼接而成，类似这样foo_int_int_；参数不同可以区分，但是返回值不同就区分不了了。

#### 拷贝构造函数

一般情况下系统默认会创建一个默认的拷贝函数，执行的拷贝就是简单将成员变量进行复制。但是如果成员变量有句柄等运行时分配的资源，那么需要定义自己的拷贝构造函数进行深拷贝。

## stack

栈内的变量一般就是在函数内部声明的，或者是函数的形参。其作用域也是在**本次调用**内部可见。但是对于static变量，那么就是多次调用都可见。

## heap

heap一般就是动态申请的空间的位置。一般有两种动态空间申请的方法：new/malloc。

### 内存申请

new vs malloc

| NEW                              | MALLOC                   |
| -------------------------------- | ------------------------ |
| 调用构造函数                     | 不                       |
| 是个操作符                       | 是个函数                 |
| 返回确定的数据类型               | 返回void*                |
| 失败抛出异常，也可`new(nothrow)` | On failure, returns NULL |
| 从free store申请空间             | 从heap申请空间           |
| 编译器计算大小                   | 手动设置大小             |

> Free store和heap的区别就是free store上的空间预审请，迟释放。

### 内存释放

delete和 free()一定要和new/malloc()对应使用。

# 内存泄露处理

C++代码应该最头疼的一个问题，如下一个简单的例子。

``` c
typedef char CStr[100];
...
void foo()
{
  ...
  char* a_string = new CStr;
  ...
  delete a_string;
  return;
}
```

当`delete a_string`的时候,就会发生内存泄露，最终内存耗尽（memory exhausted）；

首先我们分配内存后要判断，内存是否分配成功，失败就结束该程序；其次要注意**野指针/悬垂指针问题（Wild pointer/Dangling pointer）**：指针没有初始化，或者指针free后，没有置NULL。

## RAII

**"RAII: Resource Acquisition Is Initialization"**
意思就是任何资源的获取，不管是不是在初始化阶段，都是被一个对象获得，
而相应的释放资源就在该对象的析构函数中,资源不限于内存资源，
包括file handles, mutexes, database connections, transactions等。

c++ 本身不提供垃圾回收的机制（尽管有一些相应的第三方库），
比起Java等类似语言的finally construct的方式，要优。

同样和C语言中的内存管理相比，要优的多。源于c++的封装，比如：



### 智能指针

> 这是很有效的方法，来管理动态分配对象的生命周期。

智能指针从某种意义上来说，不是一个真的指针，但是重载了 `->` `*` `->*`指针运算符，
这使得其表现的像个内建的指针

`auto_ptr, shared_ptr, weak_ptr, unique_ptr`
后三个是c++11支持的，第一个已经被弃用了,相应的在boost中也有只能指针，不过现在c++11已经支持了就不用了，
boost:scoped_ptr 类似于 std:unique_ptr

##### unique_ptr

不可复制

##### shared_ptr

可以复制，维护一个引用计数，当最后一个引用该对象的引用退出，那么才销毁

但是可能带来的问题是 : 

1. dangling reference

``` cpp
// Create the smart pointer on the heap
MyObjectPtr* pp = new MyObjectPtr(new MyObject())
// Hmm, we forgot to destroy the smart pointer,
// because of that, the object is never destroyed!
```

2. circular reference

``` cpp
struct Owner {
   boost::shared_ptr<Owner> other;
};

boost::shared_ptr<Owner> p1 (new Owner());
boost::shared_ptr<Owner> p2 (new Owner());
p1->other = p2; // p1 references p2
p2->other = p1; // p2 references p1
```

##### weak_pointer

配合shared_ptr使用，避免循环引用的问题

shared_ptr：
每一个shared_ptr对象内部，拥有两个指针ref_ptr与res_ptr，一个指向引用计数对象，一个指向实际的资源。
在shared_ptr的拷贝构造等需要创造出其他拥有相同资源的shared_ptr对象时，会首先增加引用计数，然后将ref_ptr与res_ptr复值给新对象。
发生析构时，减小引用计数，查看是否为0，如果是，则释放res_ptr与ref_ptr。
weak_ptr简单介绍：
weak_ptr的引入，我认为是smart_ptr概念的一个补全。一个raw指针，其实有两个含义，一是管理资源的句柄（拥有对象），一是指向一个资源的指针（不拥有对象）。举个例子，一般我们创建一个对象，在使用完之后销毁，那这个指针是拥有那个对象的，指针的作用域就是这个对象的生命周期，这个指针就是第一类指针。我们在使用observer模式时，被监测对象经常会持有所有observer的指针，以便在有更新时去通知他们，但是他并不拥有那些对象，这类指针就是第二类指针。在引入smart_ptr之前，资源的创建与释放都是调用者来做决定，所以一个指针是哪一类，完全由程序员自己控制。但是smart_ptr引入之后，这个概念就凸显出来。试想，在observer例子中，我们不会容许一个对象因为他是某一个对象的观察者就无法被释放。weak_ptr就是第二类指针的实现，他不拥有资源，当需要时，他可以通过lock获得资源的短期使用权。

# 排查内存泄漏的例子

最近做了一个[LeetCode127](https://leetcode.com/problems/word-ladder/description/)，代码在本地运行正确，但是在LeetCode平台上却报出Runtime Error：

```bash
error: AddressSanitizer: heap-use-after-free on address ...
```

但是在本地运行正常的，结果也是正确，代码如下

``` c++
#include <iostream>
#include <vector>
#include <set>
#include <queue>
#include <utility>
using namespace std;

class Solution {
public:
    int ladderLength(string beginWord, string endWord, vector<string>& wordList) {
		int endi=-1;
		queue<pair<int,int>> m_queue;
		for (int i = 0; i < wordList.size(); ++i) {
			if (canTrans(wordList[i], beginWord)) {
				m_queue.push({i,2});
				isvisited.insert({i,2});
			}
			if (endWord == wordList[i]) {
				endi = i;
			}
		}
		if (endi==-1) {
			return 0;
		}

		buildGraph(wordList);

		while (!m_queue.empty()) {
			auto& cur = m_queue.front();
			m_queue.pop();

			if (cur.first == endi) {
				ans = cur.second;
				break;
			}

			for (auto i : graph[cur.first]) {
				if (isvisited.count({i,cur.second+1})) continue;

				isvisited.insert({i,cur.second+1});
				m_queue.push({i,cur.second+1});
			}
		}

		return ans;
    }

	void buildGraph(const vector<string>& wordList) {
		int N = wordList.size();
		graph.resize(N);

		for (int i = 0; i < N; ++i) {
			for (int j = i+1; j < N; ++j) {
				if (canTrans(wordList[i], wordList[j])) {
					graph[i].push_back(j);
					graph[j].push_back(i);
				}
			}
		}
	}

	bool canTrans(const string& s1, const string& s2) {
		int diffCounts = 0;
		for (int i = 0; i < s1.size(); ++i) {
			if (s1[i] != s2[i]) diffCounts++;
		}

		return diffCounts == 1;
	}

	int ans;
	set<int> visited;
	set<pair<int,int>> isvisited;

	vector<vector<int>> graph;
};

//int main(int argc, char *argv[])
//{
//	Solution s;
//
//	vector<string> ss = {"si","go","se","cm","so","ph","mt","db","mb","sb","kr","ln","tm","le","av","sm","ar","ci","ca","br","ti","ba","to","ra","fa","yo","ow","sn","ya","cr","po","fe","ho","ma","re","or","rn","au","ur","rh","sr","tc","lt","lo","as","fr","nb","yb","if","pb","ge","th","pm","rb","sh","co","ga","li","ha","hz","no","bi","di","hi","qa","pi","os","uh","wm","an","me","mo","na","la","st","er","sc","ne","mn","mi","am","ex","pt","io","be","fm","ta","tb","ni","mr","pa","he","lr","sq","ye"};
//
//	std::cout << s.ladderLength("qa", "sq", ss) << std::endl;
//
//	return 0;
//}
```

本地编译运行如下，结果正常。

```
>
g++ -fsanitize=address -O1 -fno-omit-frame-pointer -std=c++11 -g 127.word-ladder.cpp

> ./a.out
5
```

那么，爆出的内存错误是哪里呢？