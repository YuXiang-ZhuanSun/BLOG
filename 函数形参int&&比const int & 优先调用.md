# 函数实参为右值，形参int&&比const int & 优先被调用

const int & 和int && 做函数形参，都可以接受右值。

当两个函数同时重载时，实参为右值会调用哪个呢？

实参为右值会优先调用形参为 int&& 的函数，这一步由编译器自动完成。

## const int & 为什么可以接受右值？int&却不可以

```c++
int & a = 5;  //非法，非常量左值引用只能引用左值。
              //[Error] cannot bind non-const lvalue reference of type 'int&' to an rvalue of type 'int'
const int & b = 6;  //合法，常量左值引用可以引用右值，

foo(int & x);

foo(1);//非法的
int t = 1;
foo(t);
```

引用是变量的别名，由于右值没有地址，没法被修改，所以左值引用无法指向右值。

但是，const左值引用是可以指向右值的：const左值引用不会修改指向值，因此可以指向右值。

这里面其实分隐含的两步：

* 找一个存储空间，存下右值（有人会说这算是有一个temporary lvalue，
* 左值引用引用这个空间

const int &在c++没有引入右值引用之前，常用作右值函数传参。

```c++
void reff(const int & a)
{
	cout<<"const int &"<<a<<endl;
}
```

上面这个函数可以接受右值做参数，也可以接受左值做参数。看例子：

```c++
#include <bits/stdc++.h>

using namespace std;

void reff(const int & a)
{
	cout<<"const int &"<<a<<endl;
}


int main()
{
	reff(3);//纯右值
	int x=0;
	reff(x);//左值
	reff(move(x));//强制类型转换的右值
	return 0;
} 
```

这个程序不会报错，都会调用`void reff(const int & a)`正常运行。

## int && 被优先调用

在程序中添加`void reff(int && a)`函数，

```c++
#include <bits/stdc++.h>

using namespace std;

void reff(const int & a)
{
	cout<<"const int &"<<a<<endl;
}

void reff(int && a)
{
	cout<<"int &&"<<a<<endl;
}

int main()
{
	reff(3);//右值实参，优先调用int&&
	int x=0;
	reff(x);//左值实参，调用const int&&
	reff(move(x));//右值实参，优先调用int&&
	return 0;
} 
```

运行结果

```
int &&3
const int &0
int &&0
```

至此，我们就证明了函数实参为右值，形参int&&比const int & 优先被调用。

## 其他总结

const int &做函数形参

* 可以接受左值变量，是左值变量的引用，但不会修改左值变量
* 可以接受右值，不过优先级比int &&做函数形参低。

即使定义了int &&参数的函数，const int &做参数的函数也是有用武之地的：可以接受左值。

