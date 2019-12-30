# 01 一个实例：累加一个序列
## 1.1 Fixed Traits

```cpp
#include <iostream>

template<typename T>
T accum (const T* beg, const T* end)
{
    T total{};
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}

int main()
{
    char name[] = "templates";
    int length = sizeof(name)-1;
    std::cout << accum(name, name+length) / length << '\n'; // -5
}
```
* 上述代码的问题是，对于char类型希望计算对应ASCII码的平均值，但结果却是-5（预期结果是108），原因在于模板基于char类型实例化，total类型为char导致结果出现了越界。多引入一个模板参数AccT来指定total的类型即可解决此问题，但这样做的缺点是每次调用都要显式指定这个类型。另一个方法是使用traits

```cpp
template<typename T>
struct Accumulationtraits;

template<>
struct Accumulationtraits<char> {
    using AccT = int;
};

template<typename T>
auto accum (const T* beg, const T* end)
{
    using AccT = typename Accumulationtraits<T>::AccT;
    AccT total{};
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}
```

## 1.2 Value Traits
* accum模板使用了默认构造函数的返回值初始化total，但类型AccT不一定有一个默认构造函数。为了解决这个问题，需要添加一个value traits

```cpp
template<typename T>
struct Accumulationtraits;

template<>
struct Accumulationtraits<char> {
    using AccT = int;
    static const AccT zero = 0;
};

template<typename T>
auto accum (const T* beg, const T* end)
{
    using AccT = typename Accumulationtraits<T>::AccT;
    AccT total = Accumulationtraits<T>::zero;
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}
```
* 但这种方法的缺点是，类内初始化的static成员变量只能是整型（int、long、unsigned）常量或枚举类型

```cpp
template<>
struct Accumulationtraits<float> {
    using Acct = float;
    static constexpr float zero = 0.0f; // ERROR: not an integral type
};

class BigInt {
  BigInt(long long);
  ...
};

template<>
struct Accumulationtraits<BigInt> {
    using AccT = BigInt;
    static constexpr BigInt zero = BigInt{0};  // ERROR: not a literal type
};
```
* 直接的解决方法是在类外定义value traits

```cpp
template<>
struct Accumulationtraits<BigInt> {
    using AccT = BigInt;
    static const BigInt zero; // 仅声明
};
// 在源文件中初始化
const BigInt Accumulationtraits<BigInt>::zero = BigInt{0};
```
* 但这个方法仍有缺点，编译器不知道其他文件的定义，也就不知道zero的值为0。C++17中可以使用inline变量解决此问题

```cpp
template<>
struct Accumulationtraits<BigInt> {
    using AccT = BigInt;
    inline static const BigInt zero = BigInt{0}; // OK since C++17
};
```
* C++17中的另一种优先做法是使用内联成员函数。如果返回一个literal type，可以将函数声明为constexpr

```cpp
template<typename T>
class Accumulationtraits;

template<>
struct Accumulationtraits<float> {
    using AccT = double;
    static constexpr AccT zero() {
        return 0;
    }
};

template<>
struct Accumulationtraits<BigInt> {
    using AccT = BigInt;
    static BigInt zero() {
        return BigInt{0};
    }
};
```
* 使用的区别只是由访问静态数据成员改为使用函数调用语法

```cpp
AccT total = Accumulationtraits<T>::zero();
```

## 1.3 Parameterized Traits
* 添加一个模板参数来参数化traits

```cpp
template<typename T, typename Traits = Accumulationtraits<T>>
auto accum (const T* beg, const T* end)
{
    typename Traits::AccT total = Traits::zero();
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
};
```

# 02 Traits versus Policies and Policy Classes
* 除了求和还有其他形式的累积问题，如求积、连接字符串，或者找出序列中的最大值。这些问题只需要修改`total += *beg`即可，把这个算法作为一个static函数模板提取到一个Policy类中

```cpp
class MultPolicy {
public:
    template<typename T1, typename T2>
    static void accumulate (T1& total, const T2& value) {
        total *= value;
    }
};

template <typename T,
    typename Policy = MultPolicy,
    typename Traits = AccumulationTraits<T>>
auto accum (const T* beg, const T* end)
{
    using AccT = typename Traits::AccT;
    AccT total = Traits::zero();
    while (beg != end) {
        Policy::accumulate(total, *beg);
        ++beg;
    }
    return total;
}

int main()
{
    int num[] = { 1, 2, 3, 4, 5 };
    std::cout << accum<int,MultPolicy>(num, num+5);
}
```
* 但输出结果却是0，原因是对于求积，0是错误的初值，应当把value traits值改为1

## 2.1 Member Templates versus Template Template Parameters
* 之前把policy实现为含成员模板的普通类，下面用类模板实现policy类，并将其用作模板的模板参数来修改Accum接口

```cpp
template <typename T1, typename T2>
class MultPolicy {
public:
    static void accumulate (T1& total, const T2& value) {
        total *= value;
    }
};

template<typename T,
    template<typename, typename> class Policy = MultPolicy,
    typename traits = Accumulationtraits<T>>
auto accum (const T* beg, const T* end)
{
    using AccT = typename traits::AccT;
    AccT total = traits::zero();
    while (beg != end) {
        Policy<AccT, T>::accumulate(total, *beg);
        ++beg;
    }
    return total;
}
```
* 用类模板实现policy的优点是，可以让policy类携带一些依赖于模板参数的状态信息（即static数据成员），缺点则是使用时需要定义模板参数的确切个数，使得traits的表达式更冗长

## 2.2 traits和policy的区别
* traits更注重于类型
  * traits可以不需要通过额外的模板参数来传递（fixed traits）
  * traits参数通常有十分自然的、很少或不能被改写的默认值
  * traits参数紧密依赖于一个或多个主参数
  * traits一般包含类型和常量
  * traits通常由traits模板实现
* policy更注重于行为
  * policy calss需要额外的模板参数来传递
  * policy参数不需要有默认值，通常都是显式指定（尽管许多泛型组件都配置了使用频率很高的缺省policy）
  * policy参数和模板的其他模板参数通常是正交的
  * policy类一般包含成员函数
  * policy既可以用普通类实现，也可以用类模板实现
* traits和policy可以控制模板参数个数，但是多个traits和policy的排序问题就出现了。一种简单的策略是按使用频率升序排序，即traits参数位于policy参数右边，因为policy参数通常会被改写

## 2.3 运用普通的迭代器进行累积
* 下面是一个新版本的的accum，它不仅支持指针，还支持普通的迭代器

```cpp
#include <iterator>

template<typename Iter>
auto accum (Iter beg, Iter end)
{
    using VT = typename std::iterator_traits<Iter>::value_type;
    VT total{};
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}
```
* [std::iterator_traits](https://en.cppreference.com/w/cpp/iterator/iterator_traits)封装了迭代器的相关属性

```cpp
namespace std {
    template<typename T>
    struct iterator_traits<T*> {
        using difference_type = ptrdiff_t;
        using value_type = T;
        using pointer = T*;
        using reference = T&;
        using iterator_category = random_access_iterator_tag ;
    };
}
```

# 03 类型函数（Type Function）

## 3.1 确定元素类型
* 下面用偏特化为给定容器指出元素类型

```cpp
#include <iostream>
#include <vector>
#include <list>

template<typename T>
struct ElementT; // primary template

template<typename T>
struct ElementT<std::vector<T>> { // partial specialization for std::vector
    using Type = T;
};

template<typename T>
struct ElementT<std::list<T>> { // partial specialization for std::list
    using Type = T;
};

template<typename T, std::size_t N>
struct ElementT<T[N]> { // partial specialization for arrays of known bounds
    using Type = T;
};

template<typename T>
struct ElementT<T[]> { // partial specialization for arrays of unknown bounds
    using Type = T;
};

template<typename T>
void printElementType (const T& c)
{
    std::cout << typeid(typename ElementT<T>::Type).name();
}

int main()
{
    std::vector<bool> s;
    printElementType(s); // bool
    int arr[42];
    printElementType(arr); // int
}
```
* 大多数情况下，类型函数是和容器类型一起实现的，实现能被简化。如果容器类型定义了一个成员类型value_type，可以编码如下

```cpp
template<typename C>
struct ElementT {
    using Type = typename C::value_type;
};
```
* 类型函数允许根据容器类型来参数化模板，而不需要指定元素类型的模板参数，比如

```cpp
template<typename T, typename C>
T f(const C& c);
```
* 可以更方便地写为

```cpp
template<typename C>
typename ElementT<C>::Type f(const C& c);
```
* 为类型函数引入别名模板可以进一步简化代码

```cpp
template<typename T>
using ElementType = typename ElementT<T>::Type;

template<typename C>
ElementType<C> f(const C& c);
```

## 3.2 Transformation traits
### 3.2.1 移除引用

```cpp
template<typename T>
struct RemoveReferenceT {
    using Type = T;
};

template<typename T>
struct RemoveReferenceT<T&> {
    using Type = T;
};

template<typename T>
struct RemoveReferenceT<T&&> {
    using Type = T;
};

template<typename T>
using RemoveReference = typename RemoveReferenceT<T>::Type;
```
* 标准库提供了对应的[std::remove_reference](https://en.cppreference.com/w/cpp/types/remove_reference)

### 3.2.2 添加引用

```cpp
template<typename T>
struct AddLValueReferenceT {
    using Type = T&;
};

template<typename T>
using AddLValueReference = typename AddLValueReferenceT<T>::Type;

template<typename T>
struct AddRValueReferenceT {
  using Type = T&&;
};

template<typename T>
using AddRValueReference = typename AddRValueReferenceT<T>::Type;
```
* 引用折叠在这里会生效，如`AddLValueReference<int&&>`将折叠为int&
* 如果不引入特化，可以将其简化为

```cpp
template<typename T>
using AddLValueReferenceT = T&;

template<typename T>
using AddRValueReferenceT = T&&;
```
* 这样可以不用实例化类模板，但对于void作为模板实参的情况，仍需要类模板特化（别名模板不能被特化）

```cpp
template<>
struct AddLValueReferenceT<void> {
    using Type = void;
};

template<>
struct AddLValueReferenceT<const void> {
    using Type = const void;
};

template<>
struct AddLValueReferenceT<volatile void> {
    using Type = volatile void;
};

template<>
struct AddLValueReferenceT<const volatile void> {
    using Type = const volatile void;
};
```
* 标准库提供了对应的[std::add_lvalue_reference](https://en.cppreference.com/w/cpp/types/add_reference)和[std::add_rvalue_reference](https://en.cppreference.com/w/cpp/types/add_reference)，标准模板包含了void类型的特化

### 3.2.3 移除限定符

```cpp
template<typename T>
struct RemoveConstT {
    using Type = T;
};

template<typename T>
struct RemoveConstT<const T> {
    using Type = T;
};

template<typename T>
using RemoveConst = typename RemoveConstT<T>::Type;
```
* 此外，transformation traits能被组合，比如创建一个RemoveCVT traits，能同时移除const和volatile

```cpp
template<typename T>
struct RemoveCVT : RemoveConstT<typename RemoveVolatileT<T>::Type> {};

template<typename T>
using RemoveCV = typename RemoveCVT<T>::Type;
```
* 如果不需要特化，RemoveCV可以直接用别名模板简化如下

```cpp
template<typename T>
using RemoveCV = RemoveConst<RemoveVolatile<T>>;
```
* 标准库提供了对应的[std::remove_volatile，std::remove_const和std::remove_cv](https://en.cppreference.com/w/cpp/types/remove_cv)

### 3.2.4 Decay
* 模仿传值时的类型转换，即把数组和函数转指针，并去除顶层cv或引用限定符

```cpp
#include <iostream>
#include <type_traits>

template<typename T>
void f(T) {}

template<typename A>
void printParameterType(void (*)(A))
{
    std::cout << "Parameter type: " << typeid(A).name() << '\n';
    std::cout << "- is int: " << std::is_same<A, int>::value << '\n';
    std::cout << "- is const: " << std::is_const<A>::value << '\n';
    std::cout << "- is pointer: " << std::is_pointer<A>::value << '\n';
}

int main()
{
    printParameterType(&f<int>); // 未改变
    printParameterType(&f<const int>); // 退化为int
    printParameterType(&f<int[7]>); // 退化为int*
    printParameterType(&f<int(int)>); // 退化为int(*)(int)
}
```
* 可以实现一个传值时产生相同的类型转换的traits

```cpp
#include <iostream>
#include <type_traits>

// 首先定义nonarray，nonfunction的情况，移除任何cv限定符
template<typename T>
struct DecayT : RemoveCVT<T> {
};

// 使用偏特化处理数组到指针的decay，要求识别任何数组类型
template<typename T>
struct DecayT<T[]> {
    using Type = T*;
};
template<typename T, std::size_t N>
struct DecayT<T[N]> {
    using Type = T*;
};

// 函数到指针的decay，必须匹配任何函数类型
template<typename R, typename... Args>
struct DecayT<R(Args...)> {
    using Type = R (*)(Args...);
};
template<typename R, typename... Args>
struct DecayT<R(Args..., ...)> { // 匹配使用C-style vararg的函数类型
    using Type = R (*)(Args..., ...);
};

template<typename T>
using Decay = typename DecayT<T>::Type;

template<typename T>
void printDecayedType()
{
    using A = Decay<T>;
    std::cout << "Parameter type: " << typeid(A).name() << '\n';
    std::cout << "- is int: " << std::is_same<A, int>::value << '\n';
    std::cout << "- is const: " << std::is_const<A>::value << '\n';
    std::cout << "- is pointer: " << std::is_pointer<A>::value << '\n';
}

int main()
{
    printDecayedType<int>();
    printDecayedType<const int>();
    printDecayedType<int[7]>();
    printDecayedType<int(int)>();
}
```
* 标准库提供了对应的[std::decay](https://en.cppreference.com/w/cpp/types/decay)

### 3.2.5 C++98中的处理

```cpp
#include <iostream>

template<typename T>
void apply (T& x, void (*f)(T))
{
    f(x);
}

void print (int a) { std::cout << a << std::endl; }
void incr (int& a) { ++a; }

int main()
{
    int x = 7;
    apply (x, print); // 1，OK
    apply (x, incr); // 2，error
}
```
* 1处int替换T，则apply的参数类型分别是`int&`和`void(*)(int)`，而2处要用`int&`替换T才能匹配，这样却导致第一个参数类型不匹配而出错。解决方法是创建一个类型函数，给定类型本身不是引用则添加引用限定符，此外还可以提供其他需要的转换

```cpp
template <typename T>
class TypeOp { // primary template
public:
    typedef T ArgT;
    typedef T BareT;
    typedef const T ConstT;
    typedef T & RefT;
    typedef T & RefBareT;
    typedef const T & RefConstT;
};

template <typename T>
class TypeOp <const T> { // partial specialization for const types
public:
    typedef const T ArgT;
    typedef T BareT;
    typedef const T ConstT;
    typedef const T & RefT;
    typedef T & RefBareT;
    typedef const T & RefConstT;
};

template <typename T>
class TypeOp <T&> { // partial specialization for references
public:
    typedef T & ArgT;
    typedef typename TypeOp<T>::BareT BareT;
    typedef const T ConstT;
    typedef T & RefT;
    typedef typename TypeOp<T>::BareT & RefBareT;
    typedef const T & RefConstT;
};

// 不允许指向void的引用，可以将其看成普通的void类型
template<>
class TypeOp <void> { // full specialization for void
public:
    typedef void ArgT;
    typedef void BareT;
    typedef void const ConstT;
    typedef void RefT;
    typedef void RefBareT;
    typedef void RefConstT;
};

template <typename T>
void apply (typename TypeOp<T>::RefT x, void (*f)(T))
{
    f(x);
}
```
* 注意T位于受限名称中，不能被第一个实参推断出来，因此只能根据第二个实参推断T，再根据结果生成第一个参数的实际类型

## 3.3 Predicate Traits
### 3.3.1 IsSameT
* 判断两个类型是否相等

```cpp
template<typename T1, typename T2>
struct IsSameT {
    static constexpr bool value = false;
};
template<typename T>
struct IsSameT<T, T> {
    static constexpr bool value = true;
};
```
* 检查模板参数是否为整型

```cpp
if (IsSameT<T, int>::value) ...
```
* 由于产生的不是类型而是常量值，因此简化时不能使用别名模板，而应该使用constexpr变量模板

```cpp
template<typename T1, typename T2>
constexpr bool isSame = IsSameT<T1, T2>::value;
```
* 标准库提供了对应的[std::is_same](https://en.cppreference.com/w/cpp/types/is_same)

### 3.3.2 true_type和false_type
* 可以为两种可能的输出提供不同类型来改进IsSameT的定义，如果声明一个类模板BoolConstant，它带有两种可能的实例化TrueType和FalseType，则可以让IsSameT派生自TrueType或FalseType，这个技术称为tag dispatching

```cpp
#include <iostream>

template<bool val>
struct BoolConstant {
    using Type = BoolConstant<val>;
    static constexpr bool value = val;
};

using TrueType = BoolConstant<true>;
using FalseType = BoolConstant<false>;

template<typename T1, typename T2>
struct IsSameT : FalseType
{};

template<typename T>
struct IsSameT<T, T> : TrueType
{};

template<typename T>
void fooImpl(T, TrueType)
{
    std::cout << "fooImpl(T, true) for int called\n";
}

template<typename T>
void fooImpl(T, FalseType)
{
    std::cout << "fooImpl(T, false) for other type called\n";
}

template<typename T>
void foo(T t)
{
    fooImpl(t, IsSameT<T, int>{}); // 根据T是否为int选择实现
}

int main()
{
    foo(42); // calls fooImpl(42, TrueType)
    foo(7.7); // calls fooImpl(42, FalseType)
}
```
* BoolConstant实现包含一个Type成员，这允许使用别名模板来简化

```cpp
template<typename T1, typename T2>
using IsSame = typename IsSameT<T1, T2>::Type;
```
* 为了支持泛型，应该分别只有一个类型代表true和false，C++11在[<type_traits>](https://en.cppreference.com/w/cpp/header/type_traits)中提供了std::true_type和std::false_type

```cpp
// C++11/14中的定义
namespace std {
    using true_type = integral_constant<bool, true>;
    using false_type = integral_constant<bool, false>;
}

// C++17中的定义
namespace std {
    using true_type = bool_constant<true>;
    using false_type = bool_constant<false>;
}
// 其中bool_constant的定义
template<bool B>
using bool_constant = integral_constant<bool, B>;
```

## 3.4 Result Type traits
* result type traits是另一个处理多种类型的类型函数，常用于操作符模板，如写一个对两个数组相加的模板声明时，返回类型的确定

```cpp
template<typename T1, typename T2>
Array<typename PlusResultT<T1, T2>::Type>
operator+ (const Array<T1>&, const Array<T2>&);
```
* 如果提供了对应的别名模板，可以简化为

```cpp
template<typename T1, typename T2>
Array<PlusResult<T1, T2>>
operator+ (const Array<T1>&, const Array<T2>&);
```
* PlusResultT决定两个类型可能不同的值相加后的类型

```cpp
template<typename T1, typename T2>
struct PlusResultT {
    using Type = decltype(T1() + T2());
};

template<typename T1, typename T2>
using PlusResult = typename PlusResultT<T1, T2>::Type;
```
* 这个traits使用decltype推断类型，但这有一些潜在的问题，比如PlusResultT可能产生一个引用类型，而这里的数组类模板可能没有设计对引用类型的处理，更实际的，重载的operator+还可能返回一个const class类型的值

```cpp
class X { ... };
const X operator+ (const X&, const X&);
```
* 于是需要做的就是移除限定符

```cpp
template<typename T1, typename T2>
Array<RemoveCV<RemoveReference<PlusResult<T1, T2>>>>
operator+ (const Array<T1>&, const Array<T2>&);
```
* 然而还有一个问题是，表达式`T1()+T2()`会尝试值初始化，需要元素类型为T1和T2的默认构造函数，但数组类本身可能不要求元素类型的值初始化

### 3.4.1 declval
* 标准库提供了[std::declval](https://en.cppreference.com/w/cpp/utility/declval)来产生值但不要求构造函数

```cpp
namespace std {
    template<typename T>
    add_rvalue_reference_t<T> declval() noexcept;
}
```
* 这个函数模板是特意未定义的，因为它只针对于在decltype、sizeof或其他不需要定义的上下文中使用。declval返回一个类型的右值引用，即使该类型没有默认构造函数或者不能创建对象，这使得declval甚至能处理不能从函数正常返回的类型，比如抽象类类型或数组类型。`declval<T>()`用作表达式时，从T到T&&的转换由于引用折叠对`declval<T>()`的行为没有影响。declval本身不会抛出异常，在noexcept运算符的上下文中使用时很有用

```cpp
#include <iostream>
#include <utility>

struct A
{
    A() = delete;
    int f() { return 42; }
};

int main()
{
    decltype(std::declval<A>().f()) x = 1; // OK：无需构造A对象
    std::cout << x; // 1
}
```
* 使用declval即可解决之前需要值初始化的问题

```cpp
#include <utility>

template<typename T1, typename T2>
struct PlusResultT {
    using Type = decltype(std::declval<T1>() + std::declval<T2>());
};

template<typename T1, typename T2>
using PlusResult = typename PlusResultT<T1, T2>::Type;
```

# 04 SFINAE-based traits

## 4.1 SFINAE Out函数重载
* 使用SFINAE以判断一个类型是否默认可构造

```cpp
template<typename T>
struct IsDefaultConstructibleT {
private:
    template<typename U, typename = decltype(U())>
    static char test(void*);
    
    template<typename>
    static long test(...);
public:
    static constexpr bool value
        = IsSame<decltype(test<T>(nullptr)), char>; // T可默认构造则匹配第一个函数
};
```
* 通常SFINAE-based traits就是声明两个返回类型不同的重载函数模板，第一个只在检查成功时匹配，第二个是fallback：可变参数总能匹配任何调用，但如果更好的匹配就会选择其他的。判断T()能否默认构造，首先把T作为U传递（注意不能在第一个test()中直接使用模板参数T），表达式仅当U()存在才有效

```cpp
IsDefaultConstructibleT<int>::value // true

struct A {
    A() = delete;
};
IsDefaultConstructibleT<A>::value // false
```

### 4.1.1 其他可选的SFINAE-based traits实现策略
* SFINAE-based traits从C++98开始就可以实现了，关键就是声明两个返回类型不同的重载函数模板

```cpp
template<...> static char test(void*);
template<...> static long test(...);
```
* 但最初的技术使用返回类型的大小来确定哪个重载被选择（仍使用0和enum，因为nullptr和constexpr还不可用）

```cpp
enum { value = sizeof(test<...>(0)) == 1 };
```
* 一些平台上可能发生`sizeof(char) == sizeof(long)`，比如在DSP或老的Cray机器上所有的整型都有同样的大小，为了确保在所有平台上有不同的大小，可以定义如下

```cpp
using Size1T = char;
using Size2T = struct { char a[2]; };
// 或者
using Size1T = char(&)[1];
using Size2T = char(&)[2];

template<...> static Size1T test(void*); // checking test()
template<...> static Size2T test(...); // fallback
```

### 4.1.2 Making SFINAE-based traits Predicate traits
* Predicate traits返回一个派生自std::true_type或std::false_type的布尔值，这个方法也可以解决一些平台上`sizeof(char) == sizeof(long)`的问题

```cpp
#include <type_traits>

template<typename T>
struct IsDefaultConstructibleHelper {
private:
    template<typename U, typename = decltype(U())>
    static std::true_type test(void*);

    template<typename>
    static std::false_type test(...);
public:
    using Type = decltype(test<T>(nullptr));
};

template<typename T>
struct IsDefaultConstructibleT :
IsDefaultConstructibleHelper<T>::Type
{};
```

## 4.2 SFINAE Out偏特化
* 第二个实现SFINAE-based traits的方法是使用偏特化，下面同样是一个确定T是否默认可构造的例子

```cpp
#include <iostream>
#include <type_traits>

template<typename...> using VoidT = void;

// primary template:
template<typename, typename = VoidT<>> // VoidT<>即void
struct IsDefaultConstructibleT : std::false_type
{};

// partial specialization (may be SFINAE'd away):
template<typename T>
struct IsDefaultConstructibleT<T, VoidT<decltype(T())>> : std::true_type
{};

struct A
{
    A() = delete;
};

int main()
{
    std::cout << std::boolalpha;
    std::cout << IsDefaultConstructibleT<A>::value; // false
    // 先匹配偏特化，decltype(T())构造无效，SFINAE丢弃偏特化，转到原始模板
}
```
* C++17引入了[std::void_t](https://en.cppreference.com/w/cpp/types/void_t)，对应于这里引入的VoidT，C++17前可以自行定义或直接定义在std中

```cpp
#include <type_traits>
#ifndef __cpp_lib_void_t
namespace std {
    template<typename...> using void_t = void;
}
#endif
```

## 4.3 为SFINAE使用泛型lambda
* 无论使用哪种技术，总需要一些公式化的套路来定义traits：重载和调用两个test()成员函数或实现多个偏特化。C++17中可以通过在一个泛型lambda中指定检查条件简化公式化的代码
* 首先引入一个由两个嵌套的泛型lambda构造的工具

```cpp
#include <iostream>
#include <utility>
// helper: checking validity of f (args...) for F f and Args... args:
template<typename F, typename... Args, // 下面的默认实参相当于对F调用Args...
    typename = decltype(std::declval<F>() (std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

// fallback if helper SFINAE'd out:
template<typename F, typename... Args>
std::false_type isValidImpl(...);

// 下面的lambda参数f也是一个lambda，用于检查对args调用f是否有效
inline constexpr
auto isValid = [](auto f) {
    return [](auto&&... args) {
        return decltype(isValidImpl<decltype(f), decltype(args)&&...>(nullptr)) {};
    };
};

template<typename T>
struct TypeT {
    using Type = T; // TypeT<T>就是T
};

// type<T>可以生成一个T类型的值
template<typename T>
constexpr auto type = TypeT<T>{};

// decltype(valueT(x))可以得到x的原始类型
template<typename T>
T valueT(TypeT<T>); // 不需要定义

struct A
{
    A() = delete;
};

constexpr auto isDefaultConstructible
    = isValid([](auto x) -> decltype((void)decltype(valueT(x))()) {});
// 如果x的类型不能被默认构造，则decltype(valueT(x))()无效
// 由此导致第一个isValidImpl()声明中的默认实参失败，被SFINAE out
// 于是匹配到fallback，结果为std::false_type

int main()
{
    std::cout << std::boolalpha;
    std::cout << isDefaultConstructible(type<A>); // false
    std::cout << isDefaultConstructible(type<int>) // true
}
```
* isDefaultConstructible比之前的traits更可读，并仍有办法使用之前的风格

```cpp
template<typename T>
using IsDefaultConstructibleT
    = decltype(isDefaultConstructible(std::declval<T>()));

std::cout << IsDefaultConstructibleT<A>::value;
```
* 这是一个传统的模板声明，然而它只能出现在namespace scope中，而isDefaulConstructible的定义可以在block scope中被引入
* 这个技术调用的表达式和使用风格十分复杂，一些编译器可能编译失败。但如果isValid可行，许多traits都能只用一个声明实现。比如检查是否存在一个名为first的成员

```cpp
constexpr auto hasFirst
    = isValid([](auto x) -> decltype((void)valueT(x).first) {});
```

## 4.4 SFINAE-Friendly traits
* 通常，一个type traits应该可以回应一个特殊的查询而不造成程序非法。SFINAE-based traits绕开了隐藏的问题，把错误变成了否定的结果。然而，一些traits在面对错误时就不能正常表现了，比如之前的PlusResultT

```cpp
#include <utility>

template<typename T1, typename T2>
struct PlusResultT {
    using Type = decltype(std::declval<T1>() + std::declval<T2>());
};

template<typename T1, typename T2>
using PlusResult = typename PlusResultT<T1, T2>::Type;
```
* 在这个定义中，+被使用于未被SFINAE保护的上下文中，因此如果程序尝试对没有合适的operator+的类型计算PlusResultT则会出错

```cpp
template<typename T>
class Array {
    ...
};

// declare + for arrays of different element types:
template<typename T1, typename T2>
Array<typename PlusResultT<T1, T2>::Type>
operator+ (const Array<T1>&, const Array<T2>&);
```
* 明显地，如果数组元素没有定义对应的operator，使用PlusResultT将造成错误

```cpp
class A {};
class B {};
void addAB(Array<A> arrayA, Array<B> arrayB) {
    auto sum = arrayA + arrayB; // 错误：初始化PlusResultT<A, B>失败
    ...
}
```
* 实际出现的问题不像这样明显，而是通常发生于operator+的模板实参推断过程，隐藏在`PlusResultT<A, B>`的实例化中。这将导致即使添加一个特定的重载用来相加A和B数组，程序也可能无法编译，因为C++不能确定如果另一个重载更好时，函数模板中的类型能否实例化

```cpp
// declare generic + for arrays of different element types:
template<typename T1, typename T2>
Array<typename PlusResultT<T1, T2>::Type>
operator+ (const Array<T1>&, const Array<T2>&);

// overload + for concrete types:
Array<A> operator+(const Array<A>& arrayA, const Array<B>& arrayB);
void addAB(const Array<A>& arrayA, const Array<B>& arrayB) {
    auto sum = arrayA + arrayB; // ERROR?取决于编译器是否实例化PlusResultT<A, B>
    ...
}
```
* 如果编译器能确定第二个`operator+`的声明是更好的匹配，则可以通过。推断和替换候选函数模板时，类模板定义的实例化期间发生的任何事都不是函数模板替换的即时上下文的一部分，于是在PlusResultT中对A和B类型的元素调用`operator+`就会出错

```cpp
template<typename T1, typename T2>
struct PlusResultT {
    using Type = decltype(std::declval<T1>() + std::declval<T2>());
};
```
* 为了解决这个问题，必须使PlusResultT SFINAE-friendly，即通过给它一个合适的定义使其更有弹性，即使它的decltype表达式是非法的
* 定义一个HasPlusT来检查给定的类型是否有合适的`operator+`

```cpp
#include <utility> // for declval
#include <type_traits> // for true_type, false_type, and void_t

// primary template:
template<typename, typename, typename = std::void_t<>>
struct HasPlusT : std::false_type
{};

// partial specialization (may be SFINAE'd away):
template<typename T1, typename T2>
struct HasPlusT<T1, T2,
    std::void_t<decltype(std::declval<T1>() + std::declval<T2>())>
    > : std::true_type
{};
```
* 如果它产生true，PlusResultT能使用已有实现，否则PlusResultT需要一个safe default，对于一个没有有意义的结果traits最好的default就是不提供任何成员Type，这样如果traits用在SFINAE 上下文（如上面的数组`operator+`模板的返回类型）中，丢失的成员Type将使模板实参推断失败，这正是数组`operator+`模板想要的结果。下面的实现提供了这个行为

```cpp
template<typename T1, typename T2, bool = HasPlusT<T1, T2>::value>
struct PlusResultT { // primary template, used when HasPlusT yields true
    using Type = decltype(std::declval<T1>() + std::declval<T2>());
};

template<typename T1, typename T2>
struct PlusResultT<T1, T2, false> {}; // partial specialization, used otherwise
```
* 再次考虑`Array<A>`和`Array<B>`的相加，在上面这个实现中，`PlusResultT<A, B>`的实例化将不会有Type成员，因此`operator+`模板无效，SFINAE将从考虑中除去函数模板，针对`Array<A>`和`Array<B>`重载的`operator+`将被选择
* 作为通用的设计原则，如果给出合理的模板实参作为输入，一个traits模板应该永远不会在实例化期间失败。通用的方法是执行两次对应的检查
  * 一次确定操作是否有效
  * 一次计算结果

# 05 IsConvertibleT
* 判断一个给定的类型能否转换为另一个给定的类型

```cpp
#include <iostream>
#include <type_traits> // for true_type and false_type
#include <utility> // for declval

template<typename FROM, typename TO>
struct IsConvertibleHelper {
private:
    static void aux(TO); // 调用时参数将转换为TO
    
    template<typename F, typename T,
        typename = decltype(aux(std::declval<F>()))> // 调用aux
    static std::true_type test(void*); // F能转换为TO则匹配，否则SFINAE
    
    template<typename, typename>
    static std::false_type test(...);
public:
    using Type = decltype(test<FROM, TO>(nullptr));
};

template<typename FROM, typename TO>
struct IsConvertibleT : IsConvertibleHelper<FROM, TO>::Type {};

template<typename FROM, typename TO>
using IsConvertible = typename IsConvertibleT<FROM, TO>::Type;

template<typename FROM, typename TO>
constexpr bool isConvertible = IsConvertibleT<FROM, TO>::value;

class A {};
class B : A {};

int main()
{
    std::cout << std::boolalpha;
    std::cout << IsConvertibleT<B, A>::value << '\n'; // true
    std::cout << IsConvertibleT<B*, A*>::value << '\n'; // true
    std::cout << IsConvertibleT<A*, B*>::value << '\n'; // false
    std::cout << IsConvertibleT<const char*, std::string>::value << '\n'; // true
}
```
* 有三种情况不能被IsConvertibleT正确处理
  * 转换为数组类型应该总是产生false，但是这里aux()声明中的类型TO参数会退化为指针类型，导致对一些FROM类型产生true的结果
  * 转换为函数类型应该总是产生false，但这里同数组的情况一样
  * 转换为（cv限定符）void类型应该总是产生true，但这里甚至不会成功实例化TO为void类型的情况，因为参数类型不能有类型void
* 对于这些情况需要额外的偏特化，但对每个可能的cv限定符组合添加特化很麻烦。可以如下给辅助类模板添加一个额外的模板参数

```cpp
template<typename FROM, typename TO, bool =
    IsVoidT<TO>::value || IsArrayT<TO>::value || IsFunctionT<TO>::value>
struct IsConvertibleHelper {
    using Type = std::integral_constant<bool,
        IsVoidT<TO>::value && IsVoidT<FROM>::value>;
};

template<typename FROM, typename TO>
struct IsConvertibleHelper<FROM,TO,false> {
    ... // previous implementation of IsConvertibleHelper here
};
```
* 标准库提供了对应的[std::is_convertible](https://en.cppreference.com/w/cpp/types/is_convertible)

# 06 检查成员
* SFINAE-based traits还能确定给定的类型T是否有名为X的成员

## 6.1 检查类型成员
* 判断给定的类型T是否有可访问的类型成员size_type

```cpp
#include <iostream> 
#include <type_traits> // defines true_type and false_type

// helper to ignore any number of template parameters:
template<typename...> using VoidT = void;

// primary template:
template<typename, typename = VoidT<>>
struct HasSizeTypeT : std::false_type {};

// partial specialization (may be SFINAE'd away):
template<typename T>
struct HasSizeTypeT<T, VoidT<typename T::size_type>> : std::true_type {};

struct A {
    using size_type = std::size_t;
};

class B {
    using size_type = std::size_t;
};

int main()
{
    std::cout << std::boolalpha;
    std::cout << HasSizeTypeT<A>::value; // true
    std::cout << HasSizeTypeT<B>::value; // false：traits无法访问private成员
}
```

### 6.1.1 处理引用类型
* 引用类型会引起一些问题

```cpp
struct X {
    using size_type = int;
};

std::cout << HasSizeTypeT<X>::value; // true
std::cout << HasSizeTypeT<X&>::value; // false
```
* 可以用RemoveReference移除引用

```cpp
template<typename T>
struct HasSizeTypeT<T, VoidT<typename RemoveReference<T>::size_type>>
: std::true_type {};
```

### 6.1.2 注入类名
* 注意HasSizeTypeT对注入类名也将产生一个true值

```cpp
struct size_type {};
struct A : size_type {};

std::cout << HasSizeTypeT<A>::value; // true
```

## 6.2 检查任意类型成员
* HasSizeTypeT只能检查size_type类型，现在要能对任意指定的类型进行检查。之前用泛型lambda实现过此功能，如果不使用泛型lambda只能通过宏实现

```cpp
#include <iostream>
#include <type_traits>
#include <vector>

#define DEFINE_HAS_TYPE(MemType) \
template<typename, typename = std::void_t<>> \
struct HasTypeT_##MemType \
: std::false_type {}; \
template<typename T> \
struct HasTypeT_##MemType<T, std::void_t<typename T::MemType>> \
: std::true_type {} // ; intentionally skipped

DEFINE_HAS_TYPE(value_type); // 定义HasTypeT_value_type，这个traits检查value_type成员
DEFINE_HAS_TYPE(char_type); // 定义HasTypeT_char_type，这个traits检查char_type成员

int main()
{
    std::cout << std::boolalpha;
    std::cout << HasTypeT_value_type<vector<int>>::value << '\n'; // true
    std::cout << HasTypeT_value_type<std::iostream>::value << '\n'; // false
    std::cout << HasTypeT_char_type<std::iostream>::value << '\n'; // true
}
```
* 注意之前的引用类型和注入类名的问题在这里仍会出现

```cpp
DEFINE_HAS_TYPE(size_type);

struct X {
    using size_type = int;
};

struct size_type {};
struct A : size_type {};

std::cout << HasTypeT_size_type<X>::value; // true
std::cout << HasTypeT_size_type<X&>::value; // false
std::cout << HasTypeT_size_type<A>::value; // true
```

## 6.3 检查非类型成员
* 检查可访问的非静态数据和非静态函数成员

```cpp
#include <iostream>
#include <type_traits>
#include <vector>

#define DEFINE_HAS_MEMBER(Member) \
template<typename, typename = std::void_t<>> \
struct HasMemberT_##Member \
: std::false_type {}; \
template<typename T> \
struct HasMemberT_##Member<T, std::void_t<decltype(&T::Member)>> \
: std::true_type { } // ; intentionally skipped

DEFINE_HAS_MEMBER(size); // &T::size无效时，SFINAE丢弃上面的偏特化
DEFINE_HAS_MEMBER(first); // 前缀&是为了要求成员必须是非类型、非枚举

int main()
{
    std::cout << std::boolalpha;
    std::cout << HasMemberT_size<std::vector<int>>::value << '\n'; // true
    std::cout << HasMemberT_first<std::pair<int, int>>::value << '\n'; // true
}
```

### 6.3.1 检查成员函数
* HasMember只检查单个成员的名称，如果存在多个成员，比如重载的成员函数，traits会失效

```cpp
DEFINE_HAS_MEMBER(begin);
std::cout << HasMemberT_begin<std::vector<int>>::value; // false
```
* 为此可以简化traits只检查函数，技巧在于构建一个判断条件的decltype表达式

```cpp
template<typename, typename = std::void_t<>>
struct HasBeginT : std::false_type {};

// partial specialization (may be SFINAE'd away):
template<typename T>
struct HasBeginT<T, std::void_t<decltype(std::declval<T>().begin())>>
: std::true_type {};

std::cout << HasBeginT<vector<int>>::value; // true
```

### 6.3.2 检查其他表达式
* 上述技术可用于其他表达式，比如检查两个类型之间是否有比较大小的运算符

```cpp
#include <iostream>
#include <utility> // for declval
#include <type_traits> // for true_type, false_type, and void_t
#include <complex>

// primary template:
template<typename, typename, typename = std::void_t<>>
struct HasLessT : std::false_type {};

// partial specialization (may be SFINAE'd away):
template<typename T1, typename T2>
struct HasLessT<T1, T2,
    std::void_t<decltype(std::declval<T1>() < std::declval<T2>())>>
: std::true_type {};

int main()
{
    std::cout << std::boolalpha;
    std::cout << HasLessT<int, char>::value << '\n'; // true
    std::cout << HasLessT<std::string, std::string>::value << '\n'; // true
    std::cout << HasLessT<std::string, int>::value << '\n'; // false
    std::cout << HasLessT<std::string, char*>::value << '\n'; // true
    std::cout << HasLessT<std::complex<double>,
        std::complex<double>>::value << '\n'; // false
}
```
* 这个traits可以用于要求模板参数是支持比较大小的类型

```cpp
template<typename T>
class X
{
    static_assert(HasLessT<T>::value,
        "Class X requires comparable elements");
    ...
};
```
* [std::void_t](https://en.cppreference.com/w/cpp/types/void_t)可接受任意数量的模板参数，由此可以一次组合多个表达式

```cpp
#include <utility> // for declval
#include <type_traits> // for true_type, false_type, and void_t

// primary template:
template<typename, typename = std::void_t<>>
struct HasVariousT : std::false_type {};

// partial specialization (may be SFINAE'd away):
template<typename T>
struct HasVariousT<T, std::void_t<decltype(std::declval<T>().begin()),
    typename T::difference_type, typename T::iterator>>
: std::true_type {};
```

## 6.4 使用泛型lambda检查成员
* 比起使用宏，泛型lambda能更紧凑地定义检查成员的traits

```cpp
#include <iostream>
#include <string>
#include <utility>

template<typename F, typename... Args,
    typename = decltype(std::declval<F>() (std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

template<typename F, typename... Args>
std::false_type isValidImpl(...);

inline constexpr
auto isValid = [](auto f) {
    return [](auto&&... args) {
        return decltype(isValidImpl<decltype(f), decltype(args) && ...>(nullptr)) {};
    };
};

template<typename T>
struct TypeT {
    using Type = T;
};

template<typename T>
constexpr auto type = TypeT<T>{};

template<typename T>
T valueT(TypeT<T>);

int main()
{
    std::cout << std::boolalpha;

    // 检查数据成员first
    constexpr auto hasFirst
        = isValid([](auto x) -> decltype((void)valueT(x).first) {});

    std::cout << hasFirst(type<std::pair<int, int>>) << '\n'; // true

    // 检查类型成员size_type
    constexpr auto hasSizeType
        = isValid([](auto x) -> typename std::decay_t<decltype(valueT(x))>::size_type{});

    struct X { using size_type = std::size_t; };
    std::cout << hasSizeType(type<X>) << '\n'; // true
    std::cout << hasSizeType(type<X&>) << '\n'; // true
    
    if constexpr(!hasSizeType(type<int>)) {
        std::cout << "int has no size_type\n";
    }

    // 检查类型之间是否支持operator<
    constexpr auto hasLess
        = isValid([](auto x, auto y) -> decltype(valueT(x) < valueT(y)) {});
        
    std::cout << hasLess(type<int>, type<char>) << '\n'; // true
    std::cout << hasLess(type<std::string>, type<std::string>) << '\n'; // true
    std::cout << hasLess(type<std::string>, type<int>) << '\n'; // false
    std::cout << hasLess(type<std::string>, type<const char*>) << '\n'; // true
}
```
* 如果要支持常用的traits语法，可定义如下

```cpp
#include <iostream>
#include <string>
#include <utility>

template<typename F, typename... Args,
    typename = decltype(std::declval<F>() (std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

template<typename F, typename... Args>
std::false_type isValidImpl(...);

inline constexpr
auto isValid = [](auto f) {
    return [](auto&&... args) {
        return decltype(isValidImpl<decltype(f), decltype(args) && ...>(nullptr)) {};
    };
};

constexpr auto hasFirst
    = isValid([](auto&& x) -> decltype((void)&x.first) {});

template<typename T>
using HasFirstT = decltype(hasFirst(std::declval<T>()));

constexpr auto hasSizeType
    = isValid([](auto&& x) -> typename std::decay_t<decltype(x)>::size_type {});

template<typename T>
using HasSizeTypeT = decltype(hasSizeType(std::declval<T>()));

constexpr auto hasLess
    = isValid([](auto&& x, auto&& y) -> decltype(x < y) {});

template<typename T1, typename T2>
using HasLessT = decltype(hasLess(std::declval<T1>(), std::declval<T2>()));

int main()
{
    std::cout << std::boolalpha;
    std::cout << HasFirstT<std::pair<int, int>>::value << '\n'; // true
    struct X { using size_type = std::size_t; };
    std::cout << HasSizeTypeT<X>::value << '\n'; // true
    std::cout << HasSizeTypeT<X&>::value << '\n'; // true
    std::cout << HasLessT<int, char>::value << '\n'; // true
    std::cout << HasLessT<std::string, std::string>::value << '\n'; // true
    std::cout << HasLessT<std::string, int>::value << '\n'; // false
    std::cout << HasLessT<std::string, const char*>::value << '\n'; // true
}
```

# 07 其他traits技术

## 7.1 If-Then-Else
* IfThenElse通过一个布尔参数在两个类型参数进行选择

```cpp
// traits/ifthenelse.hpp

#ifndef IFTHENELSE_HPP
#define IFTHENELSE_HPP

// primary template: yield the second argument by default and rely on
// a partial specialization to yield the third argument if COND is false
template<bool COND, typename TrueType, typename FalseType>
struct IfThenElseT {
    using Type = TrueType;
};

// partial specialization: false yields third argument
template<typename TrueType, typename FalseType>
struct IfThenElseT<false, TrueType, FalseType> {
    using Type = FalseType;
};

template<bool COND, typename TrueType, typename FalseType>
using IfThenElse =
    typename IfThenElseT<COND, TrueType, FalseType>::Type;

#endif // IFTHENELSE_HPP
```
* 下面的类型函数能确定某个值的最低级别整型

```cpp
// traits/smallestint.hpp

#include <limits>
#include "ifthenelse.hpp"

template<auto N>
struct SmallestIntT {
    using Type =
        typename IfThenElseT<N <= std::numeric_limits<char> ::max(), char,
            typename IfThenElseT<N <= std::numeric_limits<short> ::max(), short,
                typename IfThenElseT<N <= std::numeric_limits<int> ::max(), int,
                    typename IfThenElseT<N <= std::numeric_limits<long>::max(), long,
                        typename IfThenElseT<N <= std::numeric_limits<long long>::max(),
                            long long, // then
                            void // fallback
                    >::Type
                >::Type
            >::Type
        >::Type
    >::Type;
};
```
* 不同于常规的if-then-else语句，这里所有分支的模板实参在被选择前都会被计算，所以不能有非法的代码

```cpp
// T是bool或非整型将产生未定义行为
template<typename T>
struct UnsignedT {
    using Type = IfThenElse<std::is_integral_v<T> && !std::is_same_v<T, bool>,
        std::make_unsigned_t<T>, T>; // 无论是否被选择，所有分支都会被计算
};
```
* 添加一个类型函数作为中间层即可解决此问题

```cpp
// yield T when using member Type:
template<typename T>
struct IdentityT {
    using Type = T;
};

// to make unsigned after IfThenElse was evaluated:
template<typename T>
struct MakeUnsignedT {
    using Type = std::make_unsigned_t<T>;
};

template<typename T>
struct UnsignedT {
    using Type = IfThenElse<std::is_integral_v<T> && !std::is_same_v<T, bool>,
        MakeUnsignedT<T>, IdentityT<T>>;
};
```
* 类型函数在需要计算::Type时会实例化，所以别名模板并不能有效地用于IfThenElse的分支

```cpp
template<typename T>
using MakeUnsigned = typename MakeUnsignedT<T>::Type;

template<typename T>
struct UnsignedT {
    using Type = IfThenElse<std::is_integral_v<T> && !std::is_same_v<T, bool>,
        MakeUnsigned<T>, T>; // 仍会实例化，未解决之前的问题
};
```
* 标准库提供了对应的[std::conditional](https://en.cppreference.com/w/cpp/types/conditional)

```cpp
template<typename T>
struct UnsignedT {
    using Type = std::conditional_t<std::is_integral_v<T> && !std::is_same_v<T, bool>,
        MakeUnsignedT<T>, IdentityT<T>>;
};
```

## 7.2 检查不抛出异常的操作
* 可以通过[noexcept运算符](https://en.cppreference.com/w/cpp/language/noexcept)直接实现

```cpp
#include <utility> // for declval
#include <type_traits> // for bool_constant

template<typename T>
struct IsNothrowMoveConstructibleT
: std::bool_constant<noexcept(T(std::declval<T>()))> {};
```
* 但这个实现对无法移动构造的类型将报错而非产生false，因为T(std::declval<T&&>())无效

```cpp
class A {
public:
    A(A&&) = delete;
};

std::cout << IsNothrowMoveConstructibleT<A>::value; // 编译期错误
```
* 因此在使用[noexcept运算符](https://en.cppreference.com/w/cpp/language/noexcept)前必须确保移动构造函数有效

```cpp
#include <utility> // for declval
#include <type_traits> // for true_type, false_type, and bool_constant<>

// primary template:
template<typename T, typename = std::void_t<>>
struct IsNothrowMoveConstructibleT : std::false_type {};

// partial specialization (may be SFINAE'd away):
template<typename T>
struct IsNothrowMoveConstructibleT<T, std::void_t<decltype(T(std::declval<T>()))>>
: std::bool_constant<noexcept(T(std::declval<T>()))> {}; // 在noexcept前确保移动构造函数有效
```
* 如果不能直接访问移动构造函数则无法检查它是否抛异常，此外对应的类型不能是抽象类（但可以是抽象类的引用或指针），因此traits名为IsNothrowMoveConstructible而不是HasNothrowMoveConstructor
* 标准库提供了对应的[std::is_move_constructible](https://en.cppreference.com/w/cpp/types/is_move_constructible)

## 7.3 简化traits
* 用别名模板简化产生类型的traits

```cpp
template<typename T>
using RemoveReference = typename RemoveReferenceT<T>::Type;
```
* 别名模板可以简化代码，但使用别名模板也有一些缺点
  * 别名模板不能被特化，traits的许多技术依赖于特化，此时只能把别名模板改回类模板
  * 一些traits有意让用户特化，大量调用别名模板时就容易和类模板特化混淆
  * 使用别名模板总会实例化类型，将使得对给定类型难以避免无意义的实例化traits。换句话说，别名模板不能用于元函数转发
* 用constexpr变量模板简化产生值的traits

```cpp
template<typename T1, typename T2>
constexpr bool IsSame = IsSameT<T1, T2>::value;

template<typename FROM, typename TO>
constexpr bool IsConvertible = IsConvertibleT<FROM, TO>::value;
```

# 08 类型分类（Type Classification）

## 8.1 判断基本类型

```cpp
#include <iostream>
#include <cstddef> // for nullptr_t
#include <type_traits> // for true_type, false_type, and bool_constant<>

// primary template: in general T is not a fundamental type
template<typename T>
struct IsFundaT : std::false_type {};

// macro to specialize for fundamental types
#define MK_FUNDA_TYPE(T) \
template<> struct IsFundaT<T> : std::true_type {};

MK_FUNDA_TYPE(void)

MK_FUNDA_TYPE(bool)
MK_FUNDA_TYPE(char)
MK_FUNDA_TYPE(signed char)
MK_FUNDA_TYPE(unsigned char)
MK_FUNDA_TYPE(wchar_t)
MK_FUNDA_TYPE(char16_t)
MK_FUNDA_TYPE(char32_t)

MK_FUNDA_TYPE(signed short)
MK_FUNDA_TYPE(unsigned short)
MK_FUNDA_TYPE(signed int)
MK_FUNDA_TYPE(unsigned int)
MK_FUNDA_TYPE(signed long)
MK_FUNDA_TYPE(unsigned long)
MK_FUNDA_TYPE(signed long long)
MK_FUNDA_TYPE(unsigned long long)

MK_FUNDA_TYPE(float)
MK_FUNDA_TYPE(double)
MK_FUNDA_TYPE(long double)

MK_FUNDA_TYPE(std::nullptr_t)

#undef MK_FUNDA_TYPE

template<typename T>
void test (const T&)
{
    std::cout << IsFundaT<T>::value;
}

int main()
{
    test(7); // true
    test("hello"); // false
}
```
* 标准库提供了对应的[std::is_fundamental](https://en.cppreference.com/w/cpp/types/is_fundamental)

## 8.2 判断复合类型
* 指针类型：标准库提供了[std::is_pointer](https://en.cppreference.com/w/cpp/types/is_pointer)

```cpp
template<typename T>
struct IsPointerT : std::false_type {};

template<typename T>
struct IsPointerT<T*> : std::true_type {
    using BaseT = T;
};
```
* 引用类型：标准库提供了[std::is_lvalue_reference](https://en.cppreference.com/w/cpp/types/is_lvalue_reference)、[std::is_rvalue_reference](https://en.cppreference.com/w/cpp/types/is_rvalue_reference)、[std::is_reference](https://en.cppreference.com/w/cpp/types/is_reference)

```cpp
// 判断左值引用
template<typename T>
struct IsLValueReferenceT : std::false_type {};

template<typename T>
struct IsLValueReferenceT<T&> : std::true_type {
    using BaseT = T;
};

// 判断右值引用
template<typename T>
struct IsRValueReferenceT : std::false_type {};

template<typename T>
struct IsRValueReferenceT<T&&> : std::true_type {
    using BaseT = T;
};

// 判断引用
template<typename T>
class IsReferenceT: public IfThenElse<IsLValueReferenceT<T>::value,
    IsLValueReferenceT<T>, IsRValueReferenceT<T>> {};
```
* 数组类型：标准库提供了[std::is_array](https://en.cppreference.com/w/cpp/types/is_array)，同时提供了[std::rank](https://en.cppreference.com/w/cpp/types/rank)和[std::extent](https://en.cppreference.com/w/cpp/types/extent)来允许查询大小

```cpp
#include <cstddef>

template<typename T>
struct IsArrayT : std::false_type {};

template<typename T, std::size_t N>
struct IsArrayT<T[N]> : std::true_type {
    using BaseT = T;
    static constexpr std::size_t size = N;
};

template<typename T>
struct IsArrayT<T[]> : std::true_type {
    using BaseT = T;
    static constexpr std::size_t size = 0;
};
```
* 类成员指针类型：标准库提供了[std::is_member_object_pointer](https://en.cppreference.com/w/cpp/types/is_member_object_pointer)、[std::is_member_function_pointer](https://en.cppreference.com/w/cpp/types/is_member_function_pointer)、[std::is_member_pointer](https://en.cppreference.com/w/cpp/types/is_member_pointer)

```cpp
template<typename T>
struct IsPointerToMemberT : std::false_type {};

template<typename T, typename C>
struct IsPointerToMemberT<T C::*> : std::true_type {
    using MemberT = T;
    using ClassT = C;
};
```

## 8.3 判断函数类型
* 函数类型有任意数量的参数影响结果，因此在匹配函数类型的偏特化中，借用一个参数包来捕获所有的参数类型

```cpp
template<typename... Elements>
class Typelist {};

template<typename T>
struct IsFunctionT : std::false_type {};

template<typename R, typename... Params>
struct IsFunctionT<R (Params...)> : std::true_type
{ //functions
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = false;
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params..., ...)> : std::true_type
{ // variadic functions
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = true;
};
```
* IsFunctionT不能处理所有函数类型，因为函数类型还涉及cv限定符、左值右值引用限定符，以及C++17的noexcept限定符，比如

```cpp
using MyFuncType = void (int&) const;
```
* 标记为const的函数类型不是真的const类型，无法用RemoveConst去除。为了识别带限定符的函数类型，需要额外引入大量的偏特化来覆盖所有的限定符组合。这里只阐述其中五个

```cpp
template<typename R, typename... Params>
struct IsFunctionT<R (Params...) const> : std::true_type {
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = false;
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params..., ...) volatile> : std::true_type {
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = true;
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params..., ...) const volatile> :
std::true_type {
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = true;
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params..., ...) &> : std::true_type {
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = true;
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params..., ...) const&> : std::true_type {
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = true;
};
```
* 标准库提供了对应的[std::is_function](https://en.cppreference.com/w/cpp/types/is_function)

## 8.4 判断类类型
* 表达式`T::*`中的T只能为类类型，对其使用SFINAE即可

```cpp
#include <type_traits>

template<typename T, typename = std::void_t<>>
struct IsClassT : std::false_type {};

template<typename T>
struct IsClassT<T, std::void_t<int T::*>> : std::true_type {};
```
* lambda表达式是一个匿名类，因此检查lambda表达式也将产生true

```cpp
auto l = []{};
static_assert<IsClassT<decltype(l)>::value, "">; // succeeds
```
* union也是一种类类型，所以检查union也将产生true
* 标准库提供了[std::is_class](https://en.cppreference.com/w/cpp/types/is_class)和[std::is_union](https://en.cppreference.com/w/cpp/types/is_union)
* 下面是C++98中确定类类型的方法

```cpp
#include <iostream>

template<typename T>
class IsClassT {
private:
    typedef char One;
    typedef struct { char a[2]; } Two;
    template<typename C> static One test(int C::*);
    template<typename C> static Two test(...);
public:
    enum { Yes = sizeof(IsClassT<T>::test<T>(0)) == 1 };
    enum { No = !Yes };
};

template<typename T>
void check()
{
    if (IsClassT<T>::Yes) std::cout << "Y" << std::endl;
    else std::cout << "N" << std::endl;
}

template<typename T>
void checkT (T)
{
    check<T>();
}

class MyClass {};
struct MyStruct {};
union MyUnion {};
void myfunc() {}
enum E{e1}e;

int main()
{
    check<int>(); // N
    check<MyClass>(); // Y
    MyStruct s;
    checkT(s); // Y
    check<MyUnion>(); // Y
    checkT(e); // N
    checkT(myfunc); // N
}
```

## 8.5 判断枚举类型
* 对于枚举类型，直接用SFINAE-based traits检查一个到int的显式转换，排除其他能转到int的类型就是枚举类型

```cpp
template<typename T>
struct IsEnumT {
    static constexpr bool value =
        !IsFundaT<T>::value &&
        !IsPointerT<T>::value &&
        !IsReferenceT<T>::value &&
        !IsArrayT<T>::value &&
        !IsPointerToMemberT<T>::value &&
        !IsFunctionT<T>::value &&
        !IsClassT<T>::value;
};
```
* 标准库提供了对应的[std::is_enum](https://en.cppreference.com/w/cpp/types/is_enum)

# 09 Policy traits
* 目前给出的traits都是用于去确定模板参数的一些属性，这些traits称为property traits。另外还存在其他类型的traits，定义了如何对待这些类型，这类traits称为policy traits，policy traits针对的是模板参数相关的更加独有的属性。通常把property traits实现为类型函数，而policy traits通常把policy封装在成员函数内部

## 9.1 只读的参数类型
* 一个小的结构也可能有高开销的拷贝构造函数，这时应该以const引用方式传递只读参数。使用policy traits模板可以根据类型大小把实参类型T映射为T或const T&

```cpp
template<typename T>
class RParam {
public:
    using Type = IfThenElse<sizeof(T) <= 2*sizeof(void*), T, const T&>;
};
```
* 另一方面，对容器类型，即使sizeof很小也可能涉及昂贵的拷贝构造函数，因此需要偏特化

```cpp
template<typename T>
struct RParam<Array<T>> {
    using Type = const Array<T>&;
};
```
* 对某些性能要求严格的类，可以选择性地设置类为传值方式

```cpp
template<typename T>
struct RParam {
    using Type = IfThenElse<(sizeof(T) <= 2*sizeof(void*)
        && std::is_trivially_copy_constructible<T>::value
        && std::is_trivially_move_constructible<T>::value), T, const T&>;
};

class A {};
class B {};

// 对B类型对象按值传递
template<>
class RParam<B> {
public:
    using Type = B;
};

// 允许按值传递和按引用传递的函数
template<typename T1, typename T2>
void f(typename RParam<T1>::Type p1, typename RParam<T2>::Type p2)
{}

int main()
{
    A a;
    B b;
    f<A, B>(a,b); // 对a按引用传递，对b按值传递
}
```
* 这种做法的缺点是，函数声明变得格外复杂，且无法使用实参推断，调用时必须显式指定模板实参。一个解决方法是使用一个内联的提供完美转发的包裹函数模板，但该方案要求内联函数会被编译器移除，即编译器直接调用位于内联函数中的函数

```cpp
template<typename T1, typename T2>
void f(typename RParam<T1>::Type p1, typename RParam<T2>::Type p2)
{}

// wrapper to avoid explicit template parameter passing
template<typename T1, typename T2>
void g(T1&& p1, T2&& p2)
{
    f<T1,T2>(std::forward<T1>(p1),std::forward<T2>(p2));
}

int main()
{
    A a;
    B b;
    g(a, b); // same as f<A, B>(a, b)
}
```

# 10 标准库中的type traits
* C++11中，[type traits](https://en.cppreference.com/w/cpp/header/type_traits)变成了标准库的内置部分。他们或多或少包含上述所有的类型函数和type traits，但对于其中一些，比如trivial operation detection traits和[std::is_union](https://en.cppreference.com/w/cpp/types/is_union)，没有已知的in-language solution，但编译器对这些traits提供了内置支持。因此如果需要type traits，只要可行就推荐使用标准库
* 标准库也定义了一些policy和property traits
  * 类模板[std::char_traits](https://en.cppreference.com/w/cpp/string/char_traits)被string和I/O stream类用作一个policy traits参数
  * 为了更容易给标准迭代器配置算法，提供了一个非常简单的property traits模板[std::iterator_traits](https://en.cppreference.com/w/cpp/iterator/iterator_traits)
  * 模板[std::numeric_limits](https://en.cppreference.com/w/cpp/types/numeric_limits)也能被用作property traits模板
  * 标准容器类型的内存分配器使用了policy traits类进行处理，C++98提供了[std::allocator](https://en.cppreference.com/w/cpp/memory/allocator)作为此目的的标准组件，C++11引入了[std::allocator_traits](https://en.cppreference.com/w/cpp/memory/allocator_traits)用于改变分配器的行为
* 改写[std::char_traits](https://en.cppreference.com/w/cpp/string/char_traits)即可实现自定义行为的string，比如让string不区分大小写

```cpp
#include <iostream>

struct ci_char_traits : public std::char_traits<char>
{
    static bool eq(char c1, char c2)
    {
        return toupper(c1) == toupper(c2);
    }
    static bool ne(char c1, char c2)
    {
        return toupper(c1) != toupper(c2);
    }
    static bool lt(char c1, char c2)
    {
        return toupper(c1) < toupper(c2);
    }
    static int compare(const char* s1, const char* s2, std::size_t n)
    {
        for (; n; ++s1, ++s2, --n) {
            const char diff (toupper(*s1) - toupper(*s2));
            if (diff < 0) { return -1; }
            else if (diff > 0) { return +1; }
        }
        return 0;
    }
    static const char* find(const char* p, int n, char a)
    {
        for (; n != 0; --n, ++p)
        {
            if (toupper(*p) == toupper(a)) return p;
        }
        return nullptr;
    }
};

// std::string其实是std::basic_string<char, char_traits<char>, std::allocator<char>>
using ci_string = std::basic_string<char, ci_char_traits>;

ostream& operator<<(ostream& os, const ci_string& str) {
    return os.write(str.data(), str.size());
}

int main()
{
    ci_string s1 = "hello";
    ci_string s2 = "HeLLO";
    if (s1 == s2) std::cout << s1;
}
```
