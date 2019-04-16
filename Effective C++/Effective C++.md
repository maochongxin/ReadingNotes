title: Effective C++读书笔记
---

> 随写随记

习惯C++
--

+ 视C++为一个语言联邦
	+ C
	+ C with class
	+ Template C++
	+ STL

+ 尽量以 const,enum,inline 替换 #define

使用 #define的 缺点:
	
	1. 宏被预处理器替换后,无法快速确定其意义
	2. 对浮点常量, 使用常量会比宏导致较小量的码
	3. #define无法为class创建专属常量. 一旦被定义,在其后编译过程中有效(除非某处被#undef)
	4. 宏展开可能导致错误
	
enum的好处:

	1. 避免让别人获得一个 pointer 或 reference 指向某个整数常量.
	2. 避免非必要的内存分配(取决于编译器实现——是否为整数型 const 对象分配空间)
	
	
+ 尽可能使用 const

1. 令声明const, 降低因错误而造成意外
2. const成员函数: 确认该函数可作用于 const 对象身上.
3. 两个成员函数如果只是常量性不同, 可以被重载.
4. 编译器强制执行「成员函数只有在不改变对象之人和成员变量时才称为const」, 但程序中应使用“概念上的常量性”.
5. 当 const 和 non-const 成员函数有着实质等价的实现时,令 non-const 调用 const版本可避免代码重复

+ 确定对象使用前被初始化

1. 初始化 != 赋值
2. 成员初始化的时间在构造函数自动调用之时,比进入函数本体的时间早(初始化列表).
3. 使用初始化列表初始化在某些情况下可以提高效率.
4. 初始化列表可以避免成员变量是 const 或 reference 时不能赋值的问题.
5. 将每个 non-local static 对象搬到一个属于自己的 static 函数里,返回一个 reference 指向它所包含的对象.可以避免「定义于不同编译单元内的 non-local static 对象初始化相对次序不确定问题」.


构造/析构/赋值运算
--

+ 了解C++默默编写并调用哪些函数

1. 如果未声明 copy 构造函数, copy assignment 操作符(operator=),和析构函数, (当被调用时)那么编译器会自己声明上述函数.如果没有声明任何构造函数,编译器会声明一个 default 构造函数. 并且上述所有函数有是 public 且 inline. 除非该 class 的 base class自身

2. operator= 只有生出的代码合法且有机会证明它有意义才会生成.



+ 若不想使用编译器自动生成的函数,就该明确拒绝

1. 将 copy constructor 和 operator= 声明为private
2. 继承 Uncopyable 类

```C++

class Uncopyable {
protect:
	Uncopyable() {}
	~Uncopyanle() {}
private:
	Uncopyable(const Uncopyable&);
	Uncopyable& operator=(const Uncopyable&);

}

```


+ 为多态基类声明 virtual 析构函数

1. 只为有 virtual 函数的 class 声明 virtual 析构函数 (因为虚函数表会增大对象体积)
2. 适当的时候可以将 base class 的析构函数 设置为 pure-virtual 析构函数(当希望一个 class 作为抽象类)
3. 给 base classes 一个virtual 析构函数只适用于 多态性质的 base classes 上.

+ 别让异常逃离析构函数
1. 析构函数不要吐出异常. 如果一个虚构函数调用的函数会吐出异常, 析构函数应该捕获任何异常, 并吞下(不传播)或者结束程序.

2. 如果某个操作可能在失败时抛出异常,而又**存在某种需要必须处理该异常**, 那么这个异常必须来自析构函数以外的某个函数.

+ 绝不在构造和析构过程中调用 virtual 函数

1. 在构造函数和析构函数中, virtual 函数会失去多态性.原因是 virtual 函数指针直到子类对象被构建成功后才会指向子类虚函数表, 根据构造次序, 先构造 base class 部分, 再构造 derived class 部分, base class 构造函数调用的时候, 虚函数指针还指向 base class 的虚函数表,所以调用的是 base class 的函数.也就是在 base class 构造期间, 对象的类型是 base class 而不是 derived class, virtual 函数和运行期类型信息 都会把对象视为 base class 类型.
2. 

+ 令 operator= 返回一个 reference* this

1. 为了连续赋值

+ 在 operator= 中处理“自我赋值”

若不处理可能会存在delete掉自身后又访问自身的行为, (补充知识: new 出的资源被释放后, 并不会立马清空数据, 访问那一片内存仍然可以得到想要的结果(内容未被覆盖之前), 但这种行为仍然是危险的,因为被释放掉的内存随时可能被再次申请.)

传统做法是在拷贝进行之前判断,例如:

```C++
Widget& Widget::operator(const Widget& rhs) {
	if (this == &rhs) { return *this; }
	
	delete pb;
	pb = new Bitmap(*rhs.pb);
	return *this;
}
```
但是这种做法尽管解决了“自我赋值安全性”的问题, 还存在异常方面的问题. 比如, 如果 new Bitmap异常(内存不足或copy 构造函数抛出异常), Widget 最终会有一个指针指向一块被删除的Bitmap, 最终导致的结果就是无法删除也无法安全的读取.

比较好的做法在解决了异常安全性的问题的同时自动获得自我赋值安全.

```C++
Widget& Widget::operator=(const Widget& rhs) {
	Bitmap* pOrig = pb;
	pb = new Bitmap(*rhs.pb);
	delete pOrig;
	return *this;
}

``` 
如果关注效率将“证同测试”放在函数起始处, 需要考虑“自我赋值”的频率多大, 因为测试也需要成本, 会使代码变大(原始码, 目标码),还会引入一个控制流(control flow)分支,两者都会是运行变慢.

常见足够好的做法: 通过 copy and swap 技术 详见条款29.

```C++
class Widget {
...
	void swap(Widget& rhs) {
		...
		// 交换 *this 和 rhs 的数据
	}
};

Widget& Widget::operator= (const Widget& ths) {
	Widget temp(rhs);
	swap(temp);
	return *this;
}
```

另一种 pass by value 的方式(copy assignment 操作符被声明为 by value 的方式传参):

```C++
Widget& Widget::operator=(Widget rhs) { 
						// 特色: 将copy动作从函数本体移到函数参数构造阶段有时
						// 可令编译器生成更高效的代码
	swap(rhs);
	return *this;
}

``` 

**确保任何函数在操作一个以上的对象, 而其中多个对象是同一个对象时的行为依然正确**

+ 复制对象时勿忘其每一个成分

1. Copying 函数应确保复制“对象内所有的成员变量”, 以及“所有的base class成分”
2. 不要试图通过一个 copying 函数实现另一个 copying函数, 应该将 共同机能放进第三个函数中, 并由几个 copying 函数共同调用.


资源管理
--

+ 以对象管理资源

**目的: 防止因为一些原因导致的资源无法释放而造成内存泄露**

**做法: 将资源放进对象中,依赖C++的析构函数自动调用机制确保资源被释放**

以对象管理资源也被称为——资源获取即初始化, 也就是获得一笔资源后在同一语句内用它初始化或赋值给某个管理对象.

不管控制流如何离开区块, 一旦对象被销毁,析构函数就会自动调用释放资源. 如果资源释放动作会抛出异常, 也会在析构函数中被解决.

auto_ptr 的特性: 通过 copying构造函数或 copy assignment操作符复制它们, 它们会变成NULL, **复制所得的指针获得资源的唯一控制权**, 所以不必担心多个 auto_ptr 指向同一个对象, 也正因为这样, auto_ptr不适合管理动态分配资源. 例如: STL 容器要求其元素发挥“正常的复制行为”.  替代方案是使用引用计数技术的智能指针——shared_ptr.

但是由于两者都是在析构函数内做 delete 而不是 delete[], 所以在面对动态分配的数组时, 不能使用以上两种方式, 标准并没有特别针对 C++动态分配数组 而设计的类似于以上两种资源管理的方案, 因为 vector 和 string几乎可以取代动态分配而得的数组. 如果一定要有, 可以参考 boost::scoped_array 和 boost::shared_array classes.


+ 在资源管理类中小心 coping 行为

+ 在资源管理类中提供对原始资源的访问

+ 成对使用 new 和 delete 时要采取相同形式

+ 以独立语句将 newed 对象置入智能指针


























