#Item18:
当你要使用一个智能指针时，首先要想到的应该是`std::unique_ptr`.下面是一个很合理的假设:默认情况下，`std::unique_ptr`和原生指针同等大小，对于大多数操作(包括反引用)，它们执行的底层指令也一样。这就意味着，尽管在内存回收直来直往的情况下，`std::unique_ptr`也足以胜任原生指针轻巧快速的使用要求。

`std::unique_ptr`具现了独占(exclusive ownership)语义,一个非空的`std::unique_ptr`永远拥有它指向的对象，move一个`std::unique_ptr`会将所有权从源指针转向目的指针(源指针指向为null)。拷贝一个`std::unique_ptr`是不允许的，假如说真的可以允许拷贝`std::unique_ptr`,那么将会有两个`std::unique_ptr`指向同一块资源区域，每一个都认为它自己拥有且可以摧毁那块资源。因此，`std::unique_ptr`是一个move-only类型。当它面临析构时，一个非空的`std::unique_ptr`会摧毁它所拥有的资源。默认情况下，`std::unique_ptr`会使用delete来释放它所包裹的原生指针指向的空间。

`std::unique_ptr`的一个常见用法是作为一个工厂函数返回一个继承层级中的一个特定类型的对象。假设我们有一个投资类型的继承链。

[18-1.png]

```cpp
class Investment { ... };    
class Stock:public Investment { ... };
class Bond:public Investment { ... };
class RealEstate:public Investment { ... };
``` 

生产这种层级对象的工厂函数通常在堆上面分配一个对象并且返回一个指向它的指针。当不再需要使用时，调用者来决定是否删除这个对象。这是一个绝佳的`std::unique_ptr`的使用场景。因为调用者获得了由工厂函数分配的对象的所有权(并且是独占性的)，而且`std::unique_ptr`在自己即将被销毁时，自动销毁它所指向的空间。一个为Investment层级对象设计的工厂函数可以声明如下：

```cpp
template<typename... Ts> 
std::unique_ptr<Investment> makeInvestment(Ts&&... params);// return std::unique_ptr
    // to an object created
    // from the given args
```
调用者可以在一处代码块中使用返回的`std::unique_ptr`:

```cpp
{
	...
	auto pInvestment = makeInvestment( arguments ); 
	//pInvestment is of type std::unique_ptr<Investment>
	...
}//destroy *pInvestment
``` 
    
他们也可以使用在拥有权转移的场景中，例如当工厂函数返回的`std::unique_ptr`可以移动到一个容器中，这个容器随即被移动到一个对象的数据成员上，该对象随后即被销毁。当该对象被销毁后，该对象的`std::unique_ptr`数据成员也随即被销毁，它的析构会引发工厂返回的资源被销毁。如果拥有链因为异常或者其他的异常控制流(如，函数过早返回或者for循环中的break语句)中断，最终拥有资源的`std::unique_ptr`仍会调用它的析构函数(注解：这条规则仍有例外：大多数源自于程序的非正常中断。一个从一个线程主函数(如程序的初始线程的main函数)传递出来的异常，或者一个违背了noexpect规范(请看Item 14)的异常,本地对象不会得到析构，如果`std::abort`或者其他的exit函数(如`std::_Exit`, `std::exit`,或者`std::quick_exit`)被调用，那么它们肯定不会被析构)，`std::unique_ptr`管理的资源也因此得到释放。

默认情况下，析构函数会使用delete。但是，我们也可以在它的构造过程中指定特定的析构方法(custom deleters):当资源被回收时，传入的特定的析构方法(函数对象，或者是特定的lambda表达式)会被调用。对于我们的例子来说，如果被makeInvestment创建的对象不应该直接被deleted，而是首先要有一条log记录下来，我们就可以这样实现makeInvestment（当你看到意图不是很明显的代码时，请注意看注释）

```cpp
auto delInvmt = [](Investment* pInvestment){
	makeLogEntry(pInvestment);
	delete pInvestment;
};//custom deleter(a lambda expression)
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>//revised return type
makeInvestment(Ts&&... params)
{
	std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);//ptr to be returned
	if ( /* a Stock object should be created */ )
	{
       pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* a Bond object should be created */ )
    {
       pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* a RealEstate object should be created */ )
    {
       pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```
我之前说过，当使用默认的析构方法时(即，delete)，你可以假设`std::unique_ptr`对象的大小和原生指针一样。当`std::unique_ptr`用到了自定义的deleter时，情况可就不一样了。函数指针类型的deleter会使得`std::unique_ptr`的大小增长到一个字节到两个字节。对于deleters是函数对象的`std::unique_ptr`,大小的改变依赖于函数对象内部要存储多少状态。无状态的函数对象(如，没有captures的lambda expressions) 不会导致额外的大小开销。这就意味着当一个自定义的deleter既可以实现为一个函数对象或者一个无捕获状态的lambda表达式时，lambda是第一优先选择:

```cpp
auto delInvmt1 = [](Investment* pInvestment)
				{	
					makeLogEntry(pInvestment);
					delete pInvestment;
				}
//custom deleter as stateless lambda
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt1)>
makeInvestment(Ts&&.. args);//return type has size of Investment*

void delInvmt2(Investment* pInvestment)
{
	makeLogEntry(pInvestment);
	delete pInvestment;
}

template<typename... Ts>
std::unique_ptr<Investment,(void *)(Investment*)>
makeInvestment(Ts&&... params);//return type has size of Investment* plus at least size of function pointer!
```
带有过多状态的函数对象的deleters是使得`std::unique_ptr`的大小得到显著的增加。如果你发现一个自定义的deleter使得你的`std::unique_ptr`大到无法接受，请考虑重新改变你的设计。

`std::unique_ptr`会产生两种格式，一种是独立的对象(std::unique_ptr<T>)，另外一种是数组(`std::unique_ptr<T[]>`).因此，std::unique_ptr指向的内容从来不会产生任何歧义性。它的API是专门为了你使用的格式来设计的.例如，单对象格式中没有过索引操作符(操作符[]),数组格式则没有解引用操作符(操作符*和操作符->)

`std::unique_ptr`的数组格式对你来说可能是华而不实的东东，因为和原生的array相比，`std::array`,`std::vector`以及`std::string`几乎是更好的数据结构选择。我所想到的唯一的std::unique_ptr<T[]>有意义的使用场景是，你使用了C-like API来返回一个指向堆内分配的数组的原生指针，而且你像对之接管拥有权。

C++11使用`std::unique_ptr`来表述独占所有权。但是它的一项最引人注目的特性就是它可以轻易且有效的转化为`std::shared_ptr`:

```cpp
std::shared_ptr<Investment> sp = makeInvestment(arguments);//converts std::unique_ptr to std::shared_ptr
```

这就是`std::unique_ptr`很适合作为工厂函数返回值类型的原因。工厂函数不知道调用者想使用独占性的拥有语义还是共享式的拥有语义(即`std::share_ptr`).通过返回`std::unique_ptr`,工厂函数将选择权移交给了调用者,调用者在需要的时候可以将`std::unique_ptr`转化为它最富有灵活性的兄弟(如果想了解更多关于`std::shared_ptr`,请移步Item 19)

|要记住的东西|
|:--------- |
|`std::unique_ptr`是一个具有开销小，速度快，`move-only`特定的智能指针，使用独占拥有方式来管理资源。|
|默认情况下，释放资源由delete来完成，也可以指定自定义的析构函数来替代。但是具有丰富状态的deleters和以函数指针作为deleters增大了`std::unique_ptr`的存储开销|
|很容易将一个`std::unique_ptr`转化为`std::shared_ptr`|

