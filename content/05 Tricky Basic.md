> ## typename前缀
* C++默认用::访问的名称不是类，因此必须加上typename前缀，告诉编译器该名字是一个类型，否则会报错
```cpp
template<typename T>
void printcoll (const T& coll)
{
    typename T::const_iterator pos; // 必须加上typename前缀
    typename T::const_iterator end(coll.end());
    for (pos = coll.begin(); pos! = end; ++pos) std::cout << *pos << ' ';
}
```
* 这个模板使用T类型容器的迭代器，每个STL容器都声明了迭代器类型const_iterator
```cpp
class stlcontainer {
 public:
    using iterator = ...; // iterator for read/write access
    using const_iterator = ...; // iterator for read access
    ...
};
```
* 必须加typename前缀的原因，参考如下代码
```cpp
T::SubType* ptr;
```
* 如果没有加typename前缀，上式会被解析为一个乘法，而不是声明ptr为一个指针
```cpp
(T::SubType) * ptr;
```
> ## 零初始化（Zero Initialization）
* 使用模板时常希望模板类型的变量已经用默认值初始化，但内置类型无法满足要求
```cpp
template<typename T>
void f()
{
    T x; // T为内置类型则不会初始化  
}
```
* 解决方法是显式调用内置类型的默认构造函数，比如调用int()即可获得0
```cpp
template<typename T>
voidf()
{
    T x{}; // T为内置类型则x为0（或false）
    // C++11前的语法写为T x = T();
}
```
* 对于类模板则需要定义一个保证所有成员都初始化的默认构造函数
```cpp
template<typename T>
class A {
private:
    T x;
public:
    A() : x() {} // 确保x已被初始化，即使是内置类型
};
```
* C++11中可以写为
```cpp
template<typename T>
class A {
private:
    T x{};
};
```
* 但默认实参不能这样写
```cpp
template<typename T>
void f(T p{}) // 错误
{}
```
* 正确的写法是
```cpp
template<typename T>
void foo(T p = T{}) { // OK，如果是C++11前则用T() 
{}
```

> ## 派生类模板用this->调用基类同名函数
* 对于派生类模板，调用基类的同名函数时，并不一定是使用基类的此函数
```cpp
template<typename T>
class B {
public:
    void f();
};

template<typename T>
class D : B<T> {
public:
    void f2() { f(); } // 会调用外部的f或者出错
};
```
* 这里f2内部调用的f不会考虑基类的f，如果希望调用基类的，使用B<T>::或this->来指定
```cpp
template<typename T>
class B {
public:
    void f();
};

template<typename T>
class D : B<T> {
public:
    void f2() { B::<T>f(); } // 或this->f()
};
```

> ## 用于原始数组与字符串字面值（string literal）的模板
* 有时把原始数组或字符串字面值传递给函数模板的引用参数会出现问题
```cpp
template<typename T>
const T& max(const T& a, const T& b)
{
    return a < b ? b : a;
}

max("apple", "peach"); // OK
max("apple", "banana"); // 错误：类型不同，分别是const char[6]和const char[7]
``` 
* 模板参数为引用类型时，传递的原始数组不会退化为指针。将上述模板改为传值即可编译，非引用类型实参在推断过程中会出现数组到指针的转换，但这样比较的实际是指针的地址
```cpp
template<typename T>
T max(T a, T b)
{
    return a < b ? b : a;
}

std::cout << max("apple", "banana"); // apple
std::cout << max("cpple", "banana"); // cpple
```
* 因此需要为原始数组和字符串字面值提供特定处理的模板
```cpp
template<typename T, int N, int M>
bool less (T(&a)[N], T(&b)[M])
{
    for (int i = 0; i < N && i < M; ++i) {
        if (a[i]<b[i]) return true;
        if (b[i]<a[i]) return false;
    }
    return N < M;
}

int x[] = {1, 2, 3};
int y[] = {1, 2, 3, 4, 5};
std::cout << less(x,y); // true：T = int, N = 3，M = 5
std::cout << less("ab","abc"); // true：T = const char, N = 3，M = 4
```
* 如果只想支持字符串字面值，将模板参数T直接改为const char即可
```cpp
template<int N, int M>
bool less (const char(&a)[N], const char(&b)[M])
{
    for (int i = 0; i < N && i < M; ++i) {
        if (a[i]<b[i]) return true;
        if (b[i]<a[i]) return false;
    }
    return N < M;
}
```
* 对于边界未知的数组，有时必须重载或者局部特化
```cpp
#include <iostream>

template<typename T>
struct MyClass; // primary template

template<typename T, std::size_t SZ>
struct MyClass<T[SZ]> // 局部特化：用于已知边界的数组
{
    static void print() { std::cout << "print() for T[" << SZ << "]\n"; }
}; 

template<typename T, std::size_t SZ>
struct MyClass<T(&)[SZ]> // 局部特化：用于已知边界的数组的引用
{
    static void print() { std::cout << "print() for T(&)[" << SZ << "]\n"; }
}; 

template<typename T>
struct MyClass<T[]> // 局部特化：用于未知边界的数组
{
    static void print() { std::cout << "print() for T[]\n"; }
}; 

template<typename T>
struct MyClass<T(&)[]> // 局部特化：用于未知边界的数组的引用
{
    static void print() { std::cout << "print() for T(&)[]\n"; }
}; 

template<typename T>
struct MyClass<T*> // 局部特化：用于指针
{
    static void print() { std::cout << "print() for T*\n"; }
};

template<typename T1, typename T2, typename T3>
void foo(int a1[7], int a2[],    // pointers by language rules
         int (&a3)[42],          // reference to array of known bound
         int (&x0)[],            // reference to array of unknown bound
         T1 x1,                  // passing by value decays
         T2& x2, T3&& x3)        // passing by reference
{
    MyClass<decltype(a1)>::print();      // uses MyClass<T*>
    MyClass<decltype(a2)>::print();      // uses MyClass<T*>
    MyClass<decltype(a3)>::print();      // uses MyClass<T(&)[SZ]>
    MyClass<decltype(x0)>::print();      // uses MyClass<T(&)[]>
    MyClass<decltype(x1)>::print();      // uses MyClass<T*>
    MyClass<decltype(x2)>::print();      // uses MyClass<T(&)[]>
    MyClass<decltype(x3)>::print();      // uses MyClass<T(&)[]>
} 

int main()
{
    int a[42];
    MyClass<decltype(a)>::print();       // uses MyClass<T[SZ]> 
    extern int x[]; // 前置声明数组，x传引用时将变为int(&)[]
    MyClass<decltype(x)>::print();       // uses MyClass<T[]> 
    foo(a, a, a, x, x, x, x);
} 

int x[] = {0, 8, 15}; // 定义前置声明的数组
```
* 输出为
```cpp
print() for T[42]
print() for T[]
print() for T*
print() for T*
print() for T(&)[42]
print() for T(&)[]
print() for T*
print() for T(&)[]
print() for T(&)[]
```

> ## 成员模板（Member Template）
* 类的成员也可以是模板，嵌套类和成员函数都可以是模板
* 正常情况下不能用不同类型的类互相赋值
```cpp
Stack<int> intStack1, intStack2;
Stack<float> floatStack;
intStack1 = intStack2; // OK：类型相同
floatStack = intStack1; // 错误：类型不同
```
* 定义一个赋值运算符模板来实现不同类型的赋值
```cpp
template<typename T>
class Stack {
private:
    std::deque<T> elems;
public:
    void push(const T&);
    void pop();
    const T& top() const;
    bool empty() const {
        return elems.empty();
    } 
    // assign stack of elements of type T2
    template<typename T2>
    Stack& operator= (const Stack<T2>&);
};

template<typename T>
void Stack<T>::push(const T& elem)
{
    elems.push_back(elem);
}

template<typename T>
void Stack<T>::pop()
{
    assert(!elems.empty());
    elems.pop_back();
}

template<typename T>
const T& Stack<T>::top () const
{
    assert(!elems.empty());
    return elems.back();
}

template<typename T>
    template<typename T2>
Stack<T>& Stack<T>::operator= (const Stack<T2>& rhs)
{
    // 不能直接用elems = rhs.elems，因为内部的elems类型也不一样
    Stack<T2> tmp(rhs);
    elems.clear();
    while (!tmp.empty()) {
        elems.push_front(tmp.top());
        tmp.pop();
    }
    return *this;
}
```
* 为了获取用来赋值的源对象所有成员的访问权限，可以把其他的stack实例声明为友元
```cpp
template<typename T>
class Stack {
private:
    std::deque<T> elems;
public:
    void push(const T&);
    void pop();
    const T& top() const;
    bool empty() const {
        return elems.empty();
    }
    template<typename T2>
    Stack& operator= (Stack<T2> const&);
    // to get access to private members of Stack<T2> for any type T2
    template<typename> friend class Stack; // T2没被使用所以不用写为template<typename T2>
};
```
* 有了这个成员模板，就能把一个intStack赋值给floatStack，赋值后floatStack的元素仍为float
```cpp
Stack<int> intStack;
Stack<float> floatStack;
floatStack = intStack; // OK
```
* 不用担心可以给stack赋值任何类型，这行代码保证了类型检查
```cpp
elems.push_front(tmp.top());
```
* 如果把一个stringStack赋值给一个floatStack，就会出现错误
```cpp
Stack<std::string> stringStack;
Stack<float> floatStack;
floatStack = stringStack; // 错误：std::string不能转换为float
```

> ## 用成员模板参数化容器类型
```cpp
template<typename T, typename Cont = std::deque<T>>
class Stack {
private:
    Cont elems;
public:
    void push(const T&);
    void pop();
    const T& top() const;
    bool empty() const {
        return elems.empty();
    } 
    template<typename T2, typename Cont2>
    Stack& operator= (const Stack<T2, Cont2>&);
    // operator=中要访问begin、end等私有成员，必须声明友元
    template<typename, typename> friend class Stack;
};

template <typename T, typename Cont>
void Stack<T, Cont>::push(const T& elem)
{
    elems.push_back(elem);
}

template <typename T, typename Cont>
void Stack<T, Cont>::pop()
{
    assert(!elems.empty());
    elems.pop_back();
}

template <typename T, typename Cont>
const T& Stack<T, Cont>::top() const
{
    assert(!elems.empty());
    return elems.back();
}

template<typename T, typename Cont>
    template<typename T2, typename Cont2>
Stack<T, Cont>& Stack<T, Cont>::operator= (const Stack<T2, Cont2>& rhs)
{
    elems.clear();
    elems.insert(elems.begin(), rhs.elems.begin(), rhs.elems.end());
    return  *this;
}
```
* 这样实现更方便，但也可以按之前的写法实现
```cpp
template<typename T, typename Cont>
    template<typename T2, typename Cont2>
Stack<T, Cont>& Stack<T, Cont>::operator= (const Stack<T2, Cont2>& rhs)
{
    elems.clear();
    Stack<T2, Cont2> tmp(rhs);
    elems.clear();
    while (!tmp.empty()) {
        elems.push_front(tmp.top());
        tmp.pop();
    }
    return  *this;
}
```
* 如果使用这个实现，可以利用成员函数在被调用时才会被实例化的特性，来禁用赋值运算符。使用一个std::vector作为内部容器，因为赋值运算符中使用了push_front，而std::vector没有此成员函数，只要不使用赋值运算符，程序就能正常运行
```cpp
Stack<int, std::vector<int>> vStack;
vStack.push(42); vStack.push(7);
std::cout << vStack.top(); // 7
Stack<int> intStack;
vStack = intStack; // 错误：不能对vStack使用operator=
```

> ## 成员模板的特化
* 成员函数模板也能局部或全局特化
```cpp
#include <iostream>
#include <string>

class BoolString {
private:
    std::string value;
public:
    BoolString (std::string const& s) : value(s) {}
    template<typename T = std::string>
    T get() const { // get value (converted to T)
        return value;
    }
};

// bool类型的全局特化
template<>
inline bool BoolString::get<bool>() const {
    return value == "true" || value == "1" || value == "on";
}

int main()
{
    std::cout << std::boolalpha;
    BoolString s1("hello");
    std::cout << s1.get() << '\n'; // hello
    std::cout << s1.get<bool>() << '\n';  // false
    BoolString s2("on");
    std::cout << s2.get<bool>() << '\n';  // true
}
```

> ## 使用.template
* 有时调用一个成员模板，显式限定模板实参是有必要的，此时必须使用template关键字来确保<是模板实参列表的开始。下面这个例子中，如果没有template，编译器就不知道<是小于号还是模板实参列表的开始
```cpp
template<unsigned long N>
void printBitset (std::bitset<N> const& bs) {
    std::cout << bs.template to_string<char, std::char_traits<char>,
                                     std::allocator<char>>();
}
```
* .template只需要用于依赖于模板参数的名称之后，比如这里的依赖于模板参数N的bs

> ## 泛型lambda和成员模板
* lambda其实是成员模板的简写
```cpp
[] (auto x, auto y) {
  return x + y;
}
// 等价于下面这个类的一个默认构造对象的简写
class SomeCompilerSpecificName {
public:
    SomeCompilerSpecificName();  // constructor only callable by compiler
    template<typename T1, typename T2>
    auto operator() (T1 x, T2 y) const {
      return x + y;
    }
};
```

> ## 变量模板（Variable Template）
* C++14中，变量也能被参数化为一个具体类型。和所有模板一样，这个声明不应该出现在函数或局部作用域内
```cpp
template<typename T>
constexpr T pi{3.1415926535897932385};
```
* 使用一个变量模板必须指定类型
```cpp
std::cout << pi<double> << '\n';
std::cout << pi<float> << '\n';
```
* 可以在不同的编译单元中声明变量模板
```cpp
//== header.hpp:
template<typename T>
T val{};     // zero initialized value

//== translation unit 1:
#include "header.hpp"

int main()
{
    val<long> = 42;
    print();
} 

//== translation unit 2:
#include "header.hpp"

void print()
{
    std::cout << val<long> << '\n'; // OK: prints 42
}
```
* 变量模板也能有默认模板实参
```cpp
template<typename T = long double>
constexpr T pi = T{3.1415926535897932385};

std::cout << pi<> << '\n';       // outputs a long double
std::cout << pi<float> << '\n';  // outputs a float
```
* 注意必须有尖括号
```cpp
std::cout << pi << '\n'; // 错误
```
* 变量模板也能由非类型参数参数化
```cpp
#include <iostream>
#include <array> 

template<int N>
std::array<int, N> arr{}; // 零初始化N个int元素的array

template<auto N>
constexpr decltype(N) dval = N;  // dval的类型依赖于传递值的类型

int main()
{
    std::cout << dval<'c'> << '\n'; // N有char类型值'c'
    arr<10>[0] = 42; // 第一个元素设置为42（其他9个元素仍为0）
    for (std::size_t i = 0; i < arr<10>.size(); ++i)
        std::cout << arr<10>[i] << '\n';
}
```
* 变量模板的一个用法是为类模板成员定义变量
```cpp
template<typename T>
class MyClass {
public:
    static constexpr int max = 1000;
};

template<typename T>
int myMax = MyClass<T>::max;
```
* 使用时就可以直接写为
```cpp
auto i = myMax<std::string>;
// instead of
auto i = MyClass<std::string>::max;
```
* 另一个例子
```cpp
namespace std {
    template<typename T> class numeric_limits {
    public:
        ...
        static constexpr bool is_signed = false;
        ...
    };
}

template<typename T>
constexpr bool isSigned = std::numeric_limits<T>::is_signed;

isSigned<char>
// instead of
std::numeric_limits<char>::is_signed
```
* C++17开始，标准库使用变量模板来为所有产生一个值的type trait定义简写
```cpp
std::is_const_v<T>        // since C++17
// instead of
std::is_const<T>::value        //since C++11
// the standard library defines
namespace std {
    template<typename T> constexpr bool is_const_v = is_const<T>::value;
}
```

> ## 模板的模板参数（Template Template Parameter）
* 用模板的模板参数，能做到只指定容器类型而不需要指定元素类型
```cpp
Stack<int, std::vector<int>> vStack;
// 通过模板的模板参数可以写为
Stack<int, std::vector> vStack;
```
* 为此必须把第二个模板参数指定为模板的模板参数
```cpp
// basics/stack8decl.hpp
template<typename T,
    template<typename Elem> class Cont = std::deque>
class Stack {
private:
    Cont<T> elems;
public:
    void push(const T&);
    void pop();
    const T& top() const;
    bool empty() const {
        return elems.empty();
    }
};
```
* 因为Cont没有用到模板参数Elem，所以可以省略Elem
```cpp
template<typename T,
    template<typename> class Cont = std::deque>
```
* 对于模板的模板参数Cont，C++11之前只能用class关键字修饰，C++11之后可以用别名模板的名称来替代，C++17中可以用typename修饰
```cpp
// Since C++17
template<typename T,
    template<typename Elem> typename Cont = std::deque>
class Stack {
private:
    Cont<T> elems;
    ...
};
```

> ## 模板的模板实参（Template Template Argument）匹配
* 使用上面的类模板时可能会产生错误，原因是容器还有另一个参数，即内存分配器allocator，C++17之前要求模板的模板实参精确匹配模板的模板参数，即便allocator本身有一个默认值，也不会被考虑用于匹配
```cpp
template<typename T,
    template<typename Elem,
        typename Alloc = std::allocator<Elem>>
    class Cont = std::deque>
class Stack {
private:
    Cont<T> elems;
    ...
};
```
* 这里Alloc没被使用，因此也可以省略
```cpp
template<typename T,
    template<typename Elem,
        typename = std::allocator<Elem>>
    class Cont = std::deque>
class Stack {
private:
    Cont<T> elems;
    ...
};
```
* 最终版本的Stack模板如下
```cpp
#include <iostream>
#include <deque>
#include <cassert>
#include <memory>
#include <vector>

template<typename T,
    template<typename Elem,
        typename = std::allocator<Elem>>
    class Cont = std::deque>
class Stack {
private:
    Cont<T> elems;
public:
    void push(const T&);
    void pop();
    const T& top() const;
    bool empty() const {
        return elems.empty();
    }

    template<typename T2,
        template<typename Elem2,
            typename = std::allocator<Elem2>
                >class Cont2>
    Stack<T,Cont>& operator= (Stack<T2, Cont2> const&);
    // to get access to private members of any Stack with elements of type T2
    template<typename, template<typename, typename>class>
    friend class Stack;
};

template<typename T, template<typename,typename> class Cont>
void Stack<T,Cont>::push (const T& elem)
{
    elems.push_back(elem);
}

template<typename T, template<typename,typename> class Cont>
void Stack<T,Cont>::pop ()
{
    assert(!elems.empty());
    elems.pop_back();
}

template<typename T, template<typename,typename> class Cont>
const T& Stack<T,Cont>::top () const
{
    assert(!elems.empty());
    return elems.back();
}

template<typename T, template<typename,typename> class Cont>
    template<typename T2, template<typename,typename> class Cont2>
Stack<T,Cont>&
Stack<T,Cont>::operator= (Stack<T2,Cont2> const& rhs)
{
    elems.clear();                        // remove existing elements
    elems.insert(elems.begin(),           // insert at the beginning
                 rhs.elems.begin(),       // all elements from rhs
                 rhs.elems.end());
    return  *this;
}

int main()
{
    Stack<int> intStack;
    Stack<float> flaotStack;
    intStack.push(1);
    intStack.push(2);
    flaotStack.push(3.3);
    std::cout << flaotStack.top() << '\n'; // 3.3
    flaotStack = intStack;
    flaotStack.push(4.4);
    std::cout << flaotStack.top() << '\n'; // 4.4
    Stack<double, std::vector> vStack;
    vStack.push(5.5);
    std::cout << vStack.top() << '\n'; // 5.5
    vStack = flaotStack;
    while (!vStack.empty()) {
        std::cout << vStack.top() << ' '; // 4.4 2 1
        vStack.pop();
    }
}
```
