<font face="微软雅黑">

# Effective C++

---

[toc]

---

## 1、让自己习惯 C++

<br>

### 条款 01：视 C++ 为一个语言联邦

- C++ 高效编程守则视状况而变化，取决于你使用 C++ 的那一部分。

<br>

### 条款 02：尽量以 const，enum，inline 替换 #define

- 对于单纯常量，最好以 const 对象或 enums 替换 #defines。
- 对于形似函数的宏（macros），最好改用 inline 函数替换 #defines。

``` cpp {.line-numbers}
/* 以一个常量替换宏（#define） */
#define ASPECT_RATIO 1.653

const double AspectRatio = 1.653;   // 大写名称通常用于宏
                                    // 因此这里改变名称写法





/* class 专属常量 */
// 为确保此常量至多只有一份实体，你必须让它成为一个 static 成员
class GamePlayer {
private:
    static const int NumTurns = 5;      // 常量声明式
    int scores[NumTurns];               // 使用该常量
    ...
};

// 如果你取某个 class 专属常量的地址，或编译器坚持要看到一个定义式，
// 你就必须另外提供定义式如下
const int GamePlayer::NumTurns;     // NumTurns 的定义；
                                    // 为什么没有给予数值
                                    // 由于 class 常量已在声明时获得初值
                                    //（例如先前声明 NumTurns 时为它设初值 5）
                                    // 因此定义时不可以再设初值

// 旧式编译器也许不支持上述语法，它们不允许 static 成员在其声明式上获得初值
class CostEstimate {
private:
    static const double FudgeFactor;    // static class 常量声明
    ...                                 // 位于头文件内
};
const double CostEstimate::FudgeFactor = 1.35;  // static class 常量定义
                                                // 位于实现文件内





/* 枚举类型 */
class GamePlayer {
private:
    enum { NumTurns = 5 };      // "the enum hack"—— 令 NumTurns
                                // 成为 5 的一个记号名称
    int scores[NumTurns];       // 这就没问题了
    ...
};





/* 函数调用 */
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a : b);
}
```

<br>

### 条款 03：尽可能使用 const

- 将某些东西声明为 const 可帮助编译器侦测出错误用法。const 可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
- 编译器强制实施 bitwise constness，但你编写程序时应该使用“概念上的常量性”（conceptual constness）。
- 当 const 和 non-const 成员函数有着实质等价的实现时，令 non-const 版本调用 const 版本可避免代码重复。

``` cpp {.line-numbers}
// 如果关键字 const 出现在星号左边，表示被指物是常量
// 如果出现在星号右边，表示指针自身是常量
// 如果出现在星号两边，表示被指物和指针两者都是常量
char greeting[] = "Hello";
char* p = greeting;                 // non-const pointer, non-const data
const char* p = greeting;           // non-const pointer, const data
char* const p = greeting;           // const pointer, non-const data
const char* const p = greeting;     // const pointer, const data





void f1(const Widget* pw);      // f1 获得一个指针，指向一个
                                // 常量的（不变的）Widget 对象
void f2(Widget const * pw);     // f2 也是





std::vector<int> vec;
...
const std::vector<int>::iterator iter = vec.begin();  // iter 的作用像个 T* const
*iter = 10;     // 没问题，改变 iter 所指物
++iter;         // 错误！iter 是 const
std::vector<int>::const_iterator cIter = vec.begin(); // cIter 的作用像个 const T*
*cIter = 10;    // 错误！*cIter 是 const
++cIter;        // 没问题，改变 cIter





class Rational { ... };
const Rational operator* (const Rational& lhs, const Rational& rhs);

if (a * b = c) ...      // 其实是想做一个比较（comparison）动作！
                        // 将 operator* 的回传值声明为 const
                        // 可以预防那个“没意思的赋值动作”





class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const     // operator[] for
    { return text[position]; }                              // const 对象
    char& operator[] (std::size_t position)                 // operator[] for
    { return text[position]; }                              // non-const 对象
private:
    std::string text;
};

TextBlock tb("Hello");
std::cout << tb[0];     // 调用 non-const TextBlock::operator[]
const TextBlock ctb("World");
std::cout << ctb[0];    // 调用 const TextBlock::operator[]

std::cout << tb[0];     // 没问题 —— 读一个 non-const TextBlock
tb[0] = 'x';            // 没问题 —— 写一个 non-const TextBlock
std::cout << ctb[0];    // 没问题 —— 读一个 const TextBlock
ctb[0] = 'x';           // 错误！ —— 写一个 const TextBlock





class CTextBlock {
public:
    ...
    std::size_t length() const;
private:
    char* pText;
    mutable std::size_t textLength;         // 这些成员变量可能总是
    mutable bool lengthIsValid;             // 会被更改，即使在
};                                          // const 成员函数内。
std::size_t CTextBlock::length() const
{
    if (!lengthIsValid) {
        textLength = std::strlen(pText);    // 现在，可以这样，
        lengthIsValid = true;               // 也可以这样。
    }
    return textLength;
}





/* 在 const 和 non-const 成员函数中避免重复 */
// 以下重复了一些代码，例如函数调用、两次 return 语句等等
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const
    {
        ...             // 边界检验 （bounds checking）
        ...             // 志记数据访问 （log access data）
        ...             // 检验数据完整性（verify data integrity）
        return text[position];
    }
    char& operator[] (std::size_t position)
    {
        ...             // 边界检验 （bounds checking）
        ...             // 志记数据访问 （log access data）
        ...             // 检验数据完整性（verify data integrity）
        return text[position];
    }
private:
    std::string text;
};

// 避免代码重复的安全做法 —— 即使过程中需要一个转型动作
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const // 一如既往
    {
        ...
        ...
        ...
        return text[position];
    }
    char& operator[] (std::size_t position)
    {
        return const_cast<char&> (                  // 将 op[] 返回值的 const 转除
            static_cast<const TextBlock&>(*this)    // 为 *this 加上 const
                [position]                          // 调用 const op[]
        );
    }
...
};
```

<br>

### 条款 04：确定对象被使用前已先被初始化

- 为内置型对象进行手工初始化，因为 C++ 不保证初始化它们。
- 构造函数最好使用成员初值列（member initialization list），而不要在构造函数本体使用赋值操作（assignment）。初值列列出的成员变量，其排列次序应该和它们在 class 中的声明次序相同。
- 为免除“跨编译单元之初始化次序”问题，请以 local static 对象替换 non-local static 对象。

``` cpp {.line-numbers}
int x = 0;                                  // 对 int 进行手工初始化
const char* text = "A C-style string";      // 对指针进行手工初始化（亦见条款 3）

double d;
std::cin >> d;      // 以读取 input stream 的方式完成初始化





// 重要的是别混淆了赋值（assignment）和初始化（initialization）
class PhoneNumber { ... };
class ABEntry {     // ABEntry = "Address Book Entry"
public:
    ABEntry(const std::string& name, const std::string& address,
            const std::list<PhoneNumber>& phones);
private:
    std::string theName;
    std::string theAddress;
    std::lits<PhoneNumber> thePhones;
    int numTimesConsulted;
};
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
{
    theName = name;             // 这些都是赋值（assignment）
    theAddress = address;       // 而非初始化（initialization）
    thePhones = phones;
    numTimesConsulted = 0;
}

// 使用 member initialization list（成员初值列）替换赋值动作：
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
    : theName(name),
      theAddress(address),          // 现在，这些都是初始化（initialization）
      thePhones(phones),
      numTimesConsulted(0)
{  }                                // 现在，构造函数本体不必有任何动作

// 假设 ABEntry 有一个无参数构造函数，我们可将它实现如下：
ABEntry::ABEntry()
    : theName(),                // 调用 theName 的 default 构造函数；
      theAddress(),             // 为 theAddress 做类似动作；
      thePhones(),              // 为 thePhones 做类似动作；
      numTimesConsulted(0)      // 记得将 numTimesConsulted 显示初始化为 0
{  }

// 为避免需要记住成员变量何时必须在成员初值列中初始化，何时不需要，最简单的做法就是：
// 总是使用成员初值列。这样做有时候绝对必要，且又往往比赋值更高效。





// C++ 有着十分固定的“成员初始化次序”。
// base classes 更早于其 derived classes 被初始化，
// 而 class 的成员变量总是以其声明次序被初始化。
// 回头看看 ABEntry，其 theName 成员永远最先被初始化，
// 然后是 theAddress，再来是 thePhones，最后是 numTimesConsulted。
// 为避免你或你的检阅者迷惑，并避免某些可能存在的晦涩错误，
// 当你在成员初值列中条列各个成员时，最好总是以其声明次序为次序。
// 上述所谓晦涩错误，指的是两个成员变量的初始化带有次序性。
// 例如初始化 array 时需要指定大小，因此代表大小的那个成员变量必须先有初值。





// “不同编译单元内定义之 non-local static 对象”的初始化次序
class FileSystem {          // 来自你的程序库
public:
    ...
    std::size_t numDisks() const;       // 众多成员函数之一
    ...
};
extern FileSystem tfs;      // 预备给客户使用的对象；
                            // tfs 代表 “the file system”

class Directory {
public:
    Directory( params );
    ...
};
Directory::Directory( params )
{
    ...
    std::size_t disks = tfs.numDisks();     // 使用 tfs 对象
    ...
}

Directory tempDir( params );        // 为临时文件而做出的目录

// 初始化次序的重要性显现出来了：除非 tfs 在 tempDir 之前先被初始化，
// 否则 tempDir 的构造函数会用到尚未初始化的 tfs。
// C++ 对“定义于不同的编译单元内的 non-local static 对象”的初始化相对次序并无明确定义

// 一个设计便可完全消除这个问题。
// 将每个 non-local static 对象搬到自己的专属函数内
// （该对象在此函数内被声明为 static）。
// 这些函数返回一个 reference 指向它所含的对象。
// 然后用户调用这些函数，而不直接指涉这些对象。
// 换句话说，non-local static 对象被 local static 对象替换了，
// 这是 Singleton 模式的一个常见实现手法。

// 这个手法的基础在于：C++ 保证，函数内的 local static 对象会在
// “该函数被调用期间”“首次遇上该对象之定义式”时被初始化。
// 所以如果你以“函数调用”（返回一个 reference 指向 local static 对象）
// 替换“直接访问 non-local static 对象”，就获得了保证，
// 保证你所获得的那个 reference 将指向一个历经初始化的对象。

class FileSystem { ... };       // 同前
FileSystem& tfs()               // 这个函数用来替换 tfs 对象；
{                               // 它在 FileSystem class 中可能是个 static。
    static FileSystem fs;       // 定义并初始化一个 local static 对象。
    return fs;                  // 返回一个 reference 指向上述对象。
}
class Directory { ... };        // 同前
Directory::Directory( params )  // 同前，但原本的 reference to tfs
{                               // 现在改为 tfs()
    ...
    std::size_t disks = tfs().numDisks();
    ...
}
Directory& tempDir()            // 这个函数用来替换 tempDir 对象
{                               // 它在 Directory class 中可能是个 static。
    static Directory td;        // 定义并初始化 local static 对象，
    return td;                  // 返回一个 reference 指向上述对象。
}

// 但是从另一个角度看，这些函数“内含 static 对象”的事实使它们
// 在多线程系统中带有不确定性。再说一次，任何一种 non-const static
// 对象，不论它是 local 或 non-local，在多线程环境下“等待某事发生”都会有麻烦。
// 处理这个麻烦的一种做法是：在程序的单线程启动阶段
// （single-threaded startup portion）手工调用所有 reference-returning 函数，
// 这可消除与初始化相关的“竞速形势（race conditions）”。
```
<br>

## 2、构造/析构/赋值运算

<br>

### 条款 05：了解 C++ 默默编写并调用哪些函数

- 编译器可以暗自为 class 创建 default 构造函数、copy 构造函数、copy assignment 操作符，以及析构函数。

``` cpp {.line-numbers}
// empty class（空类）不再是个 empty class，如果你自己没声明，
// 编译器就会为它声明（编译器版本的）一个 copy 构造函数，一个 
// copy assignment 操作符和一个析构函数。
// 此外如果你没有声明任何构造函数，编译器也会为你声明一个 default 构造函数。
// 所有这些函数都是 public 且 inline。

// 如果你写下：
class Empty {  };

// 这就好像你写下这样的代码：
class Empty {
public:
    Empty() { ... }
    Empty(const Empty& rhs) { ... }
    ~Empty() { ... }

    Empty& operator= (const Empty& rhs) { ... }
};

Empty e1;           // default 构造函数
                    // 析构函数
Empty e2(e1);       // copy 构造函数
e2 = e1;            // copy assignment 操作符





template<typename T>
class NamedObject {
public:
    NamedObject(const char* name, const T& value);
    NamedObject(const std::string& name, const T& value);
    ...
private:
    std::string nameValue;
    T objectValue;
};

NamedObject<int> no1("Smallest Prime Number", 2);
NamedObject<int> no2(no1);      // 调用 copy 构造函数

template<class T>
class NamedObject {
public:
    // 以下构造函数如今不再接受一个 const 名称，因为 nameValue
    // 如今是个 reference-to-non-const string。先前那个 char* 构造函数
    // 已经过去了，因为必须有个 string 可供指涉。
    NamedObject(std::string name, const T& value);
    ...         // 如前，假设并未声明 operator=
private:
    std::string& nameValue;     // 这如今是个 reference
    const T objectValue;        // 这如今是个 const
};

std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2);      // 当初撰写至此，我们的狗 Persephone
                                    // 即将度过其第二个生日。
NamedObject<int> s(oldDog, 36);     // 我小时候养的狗 Satch 则是 36 岁，
                                    // —— 如果她还活着

p = s;                              // 现在 p 的成员变量该发生什么事？

// C++ 的响应是拒绝编译那一行赋值动作。
// 如果你打算在一个“内含 reference 成员”的 class 内支持赋值操作（assignment），
// 你必须自己定义 copy assignment 操作符。
```

<br>

### 条款 06：若不想使用编译器自动生成的函数，就该明确拒绝

- 为驳回编译器自动（暗自）提供的机能，可将相应的成员函数声明为 private 并且不予实现。使用像 Uncopyable 这样的 base class 也是一种做法。

``` cpp {.line-numbers}
class HomeForSale { ... };

HomeForSale h1;
HomeForSale h2;
HomeForSale h3(h1);     // 企图拷贝 h1 —— 不该通过编译
h1 = h2;                // 企图拷贝 h2 —— 也不该通过编译

class HomeForSale {
public:
    ...
private:
    ...
    HomeForSale(const HomeForSale&);                // 只有声明
    HomeForSale& operator= (const HomeForSale&);
}





// 如果你不慎在 member 函数或 friend 函数之内那么做，轮到连接器发出抱怨。
// 将连接期错误移至编译期是可能的（而且那是好事，毕竟愈早侦测出错误愈好）
class Uncopyable {
protected:                              // 允许 derived 对象构造和析构
    Uncopyable() {}
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable&);      // 但阻止 copying
    Uncopyable& operator= (const Uncopyable&);

// 为求阻止 HomeForSale 对象拷贝，我们唯一需要做的就是继承 Uncopyable：
class HomeForSale : private Uncopyable {        // class 不再声明
    ...                                         // copy 构造函数或
};                                              // copy assign. 操作符
```

<br>

### 条款 07：为多态基类声明 virtual 析构函数

- polymorphic（带多态性质的）base classes 应该声明一个 virtual 析构函数。如果 class 带有任何 virtual 函数，它就应该拥有一个 virtual 析构函数。
- Classes 的设计目的如果不是作为 base classes 使用，或不是为了具备多态性（polymorphically），就不该声明 virtual 析构函数。

``` cpp {.line-numbers}
class  TimeKeeper {
public:
    TimeKeeper();
    ~TimeKeeper();
    ...
};
class AtomicClock : public TimeKeeper { ... };   // 原子钟
class WaterClock : public TimeKeeper { ... };    // 水钟
class WristWatch : public TimeKeeper { ... };    // 腕表

TimeKeeper* getTimeKeeper();        // 返回一个指针，指向一个
                                    // TimeKeeper 派生类的动态分配对象

TimeKeeper* ptk = getTimeKeeper();      // 从 TimeKeeper 继承体系
                                        // 获得一个动态分配对象。
...                                     // 运用它...
delete ptk;                             // 释放它，避免资源泄露

// 问题出在 getTimeKeeper 返回的指针指向一个 derived class 对象（例如 AtomicClock），
// 而那个对象却经由一个 base class 指针（例如一个 TimeKeeper* 指针）被删除，
// 而目前的 base class（TimeKeeper）有个 non-virtual 析构函数。
// 这是以一个引来灾难的秘诀，因为 C++ 明白指出，
// 当 derived class 对象经由一个 base class 指针被删除，
// 而该 base class 带着一个 non-virtual 析构函数，
// 其结果未有定义——实际执行时通常发生的是对象的 derived 成分没被销毁。
// 如果 getTimeKeeper 返回指针指向一个 AtomicClock 对象，
// 其内的 AtomicClock 成分（也就是声明于 AtomicClock class 内的成员变量）很可能没被销毁，
// 而 AtomicClock 的析构函数也未能执行起来。

// 消除这个问题的做法很简单：给 base class 一个 virtual 析构函数。
// 此后删除 derived class 对象就会如你想要的那般。
// 是的，它会销毁整个对象，包括所有 derived class 成分：
class TimeKeeper {
public:
    TimeKeeper();
    virtual ~TimeKeeper();
    ...
};
TimeKeeper* ptk = getTimeKeeper();
...
delete ptk;         // 现在，行为正确。





// 如果 class 不含 virtual 函数，通常表示它并不意图被用作一个 base class。
// 当 class 不企图被当作 base class，令其析构函数为 virtual 往往是个馊主意。
class Point {               // 一个二维空间点（2D point）
public:
    Point(int xCoord, int yCoord);
    ~Point();
private:
    int x, y;
};

// 如果 Point class 内含 virtual 函数，其对象体积会增加
// 因此，无端地将所有 classes 的析构函数声明为 virtual，
// 就像从未声明它们为 virtual 一样，都是错误的。

class SpecialString : public std::string {   // 馊主意！std::string 有个
    ...                                     // non-virtual 析构函数
};

SpecialString* pss = new SpecialString("Impending Doom");
std::string* ps;
...
ps = pss;           // SpecialString* => std::string*
...
delete ps;          // 未有定义！现实中 *ps 的 SpecialString 资源会泄漏，
                    // 因为 SpecialString 析构函数没被调用。

// 相同的分析适用于任何不带 virtual 析构函数的 class，
// 包括所有 STL 容器如 vector，list，set，tr1::unordered_map 等等。





// pure virtual 函数导致 abstract（抽象）classes —— 也就是
// 不能被实体化（instantiated）的 class。
class AWOV {                // AWOV = "Abstract w/o Virtuals"
public:
    virtual ~AWOV() = 0;    // 声明 pure virtual 析构函数
};

// 这个 class 有一个 pure virtual 函数，所以它是个抽象 class，
// 又由于它有个 virtual 析构函数，所以你不需要担心析构函数的问题。
// 然而这里有个窍门：你必须为这个 pure virtual 析构函数提供一份定义：
AWOV::~AWOV {  }            // pure virtual 析构函数的定义

// 析构函数的运作方式是，最深层派生（most derived）的那个 class 其析构函数最先被调用，
// 然后是其每一个 base class 的析构函数被调用。
// 编译器会在 AWOV 的 derived classes 的析构函数中创建一个对 ~AWOV 的调用动作，
// 所以你必须为这个函数提供一份定义。





// “给 base classes 一个 virtual 析构函数”，
// 这个规则只适用于 polymorphic（带多态性质的）base classes 身上。
// 这种 base classes 的设计目的是
// 为了用来“通过 base class 接口处理 derived class 对象”。
```

<br>

### 条款 08：别让异常逃离析构函数

- 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们（不传播）或结束程序。
- 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么 class 应该提供一个普通函数（而非在析构函数中）执行该操作。

``` cpp {.line-numbers}
class Widget {
public:
    ...
    ~Widget() { ... }       // 假设这个可能吐出一个异常
};
void doSomething()
{
    std::vector<Widget> v;
    ...
}                           // v 在这里被自动销毁

// 当 vector v 被销毁，它有责任销毁其内含的所有 Widgets。
// 假设 v 内含十个 Widgets，而在析构第一个元素期间，有个异常被抛出。
// 其他九个 Widgets 还是应该被销毁（否则它们保存的任何资源都会发生泄漏），
// 因此 v 应该调用它们各个析构函数。但假设在那些调用期间，
// 第二个 Widget 析构函数又抛出异常。现在有两个同时作用的异常，这对 C++ 而言太多了。
// 在两个异常同时存在的情况下，程序若不是结束执行就是导致不明确行为。

// 如果你的析构函数必须执行一个动作，而该动作这可能会在失败时抛出异常，该怎么办？
class DBConnection {
public:
    ...
    static DBConnection create();       // 这个函数返回
                                        // DBConnection 对象；
                                        // 为求简化暂略参数。
    void close();                       // 关闭联机；失败则抛出异常
};

// 为确保客户不忘记在 DBConnection 对象身上调用 close()，
// 一个合理的想法是创建一个用来管理 DBConnection 资源的 class，
// 并在其析构函数中调用 close。
class DBConn {          // 这个 class 用来管理 DBConnection 对象
public:
    ...
    ~DBConn()           // 确保数据库连接总是会被关闭
    {
        db.close();
    }
private:
    DBConnection db;
};

// 这便允许客户写出这样的代码：
{                                           // 开启一个区块（block）。
    DBConn dbc(DBConnection::create());     // 建立 DBConnection 对象并
                                            // 交给 DBConn 对象以便管理。
    ...                                     // 通过 DBConn 的接口
                                            // 使用 DBConnection 对象。
}                                           // 在区块结束点，DBConn 对象
                                            // 被销毁，因而自动
                                            // 为 DBConnection 对象调用 close

// 只要调用 close 成功，一切都美好。但如果该调用导致异常，DBConn 析构函数会传播该异常，
// 也就是允许它离开这个析构函数。

// 两个办法可以避免这一问题。

// 如果 close 抛出异常就结束程序。通常通过调用 abort 完成：
DBConn::~DBConn()
{
    try { db.close(); }
    catch (...) {
        // 制作运转记录，记下对 close 的调用失败；
        std::abort();
    }
}
// 如果程序遭遇一个“于析构期间发生的错误”后无法继续执行，
// “强迫结束程序”是个合理选项。

// 吞下因调用 close 而发生的异常：
DBConn::~DBConn()
{
    try { db.close(); }
    catch (...) {
        // 制作运转记录，记下对 close 的调用失败；
    }
}
// 一般而言，将异常吞掉是个坏主意，因为它压制了“某些动作失败”的重要信息！
// 然而有时候吞下异常也比负担“草率结束程序”或“不明确行为带来的风险”好。

// 一个较佳的策略是重新设计 DBConn 接口，使其客户有机会对可能出现的问题作出反应。
class DBconn {
public:
    ...
    void close()                // 供客户使用的新函数
    {
        db.close();
        closed = true;
    }
    ~DBConn()
    {
        if (!closed) {
            try {               // 关闭连接（如果客户不那么做的话）
                db.close();
            }
            catch (...) {                                // 如果关闭动作失败，
                // 制作运转记录，记下对 close 的调用失败；  // 记下来并结束程序
                ...                                      // 或吞下异常。
            }
        }
    }
private:
    DBConnection db;
    bool closed;
};
// 由客户自己调用 close 并不会对它们带来负担，
// 而是给他们一个处理错误的机会，否则他们没有机会响应。
// 如果他们不认为这个机会有用（或许他们坚信不会有错误发生），
// 可以忽略它，依赖 DBConn 析构函数去调用 close。
// 如果真有错误发生——如果 close 的确抛出异常——而且 DBConn 吞下该异常或结束程序，
// 客户没有立场抱怨，毕竟他们曾有机会第一手处理问题，而他们选择了放弃。
```

<br>

### 条款 09：绝不在析构和构造过程中调用 virtual 函数

- 在构造和析构期间不要调用 virtual 函数，因为这类调用从不下降至 derived class（比起当前执行构造函数和析构函数的那层）。

``` cpp {.line-numbers}
class Transaction {                             // 所有交易的 base class
public:
    Transaction();
    virtual void logTransaction() const = 0;    // 做出一份因类型不同而不同
                                                // 的日志记录（log entry）
    ...
};

Transaction::Transaction()      // base class 构造函数之实现
{
    ...
    logTransaction();           // 最后动作是志记这笔交易
}

class BuyTransaction : public Transaction {     // derived class
public:
    virtual void logTransaction() const;        // 志记（log）此型交易
    ...
};

class SellTransaction : public Transaction {    // derived class
public:
    virtual void logTransaction() const;        // 志记（log）此型交易
    ...
};

BuyTransaction b;
// 这时候被调用的 logTransaction 是 Transaction 内的版本，
// 不是 BuyTransaction 内的版本 —— 即使目前即将建立的对象类型是 BuyTransaction。
// base class 构造期间 virtual 函数绝不会下降到 derived classes 阶层。
// 在 base class 构造期间，virtual 函数不是 virtual 函数。

// 由于 base class 构造函数的执行更早于 derived class 构造函数，
// 当 base class 构造函数执行时 derived class 的成员变量尚未初始化。
// 如果此期间调用的 virtual 函数下降至 derived classes 阶层，
// 要知道 derived class 的函数几乎必然取用 local 成员变量，
// 而那些成员变量尚未初始化。

// 对象在 derived class 构造函数开始执行前不会成为一个 derived class 对象。





// 如果 Transaction 有多个构造函数，每个都需执行某些相同的工作，
// 那么避免代码重复的一个优秀做法是把共同的初始化代码
// （其中包括对 logTransaction 的调用）放进一个初始化函数如 init 内：
class Transaction {
public:
    Transaction()
    { init(); }                                 // 调用 non-virtual
    virtual void logTransaction() const = 0;
    ...
private:
    void init()
    {
        ...
        logTransaction();                       // 这里调用 virtual！
    }
};

// 但你如何确保每次一有 Transaction 继承体系上的对象被创建，
// 就会有适当版本的 logTransaction 被调用呢？很显然，在 Transaction
// 构造函数（s）内对着对象调用 virtual 函数是一种错误做法。

// 解决这个问题一种做法是：
// 在 class Transaction 内将 logTransaction 函数改为 non-virtual，
// 然后要求 derived class 构造函数传递必要信息给 Transaction 构造函数，
// 而后那个构造函数便可安全地调用 non-virtual logTransaction。像这样：
class Transaction {
public:
    explicit Transaction(const std::string& logInfo);
    void logTransaction(const std::string& logInfo) const;  // 如今是个
                                                            // non-virtual 函数
    ...
};

Transaction::Transaction(const std::string& logInfo)
{
    ...
    logTransaction(logInfo);        // 如今是个
                                    // non-virtual 函数
}

class BuyTransaction : public Transaction {
public:
    BuyTransaction( parameters )
        : Transaction(createLogString( parameters ))    // 将 log 信息
    { ... }                                     // 传给 base class 构造函数
    ...
private:
    static std::string createLogString( parameters );
};

// 换句话说由于你无法使用 virtual 函数从 base classes 向下调用，
// 在构造期间，你可以藉由“令 derived classes 将必要的构造信息
// 向上传递至 base class 构造函数” 替换之而加以弥补。

// 令 createLogString 函数为 static，也就不可能意外指向“初期未成熟之
// BuyTransaction 对象内部未初始化的成员变量”。
```

<br>

### 条款 10：令 operator= 返回一个 reference to *this

-  令赋值（assignment）操作符返回一个 reference to *this。

``` cpp {.line-numbers}
int x, y, z;
x = y = z = 15;         // 赋值连锁形式

// 赋值采用右结合律，所以上述连锁赋值被解析为：
x = (y = (z = 15));




// 为了实现“连锁赋值”，赋值操作符必须返回一个 reference 指向操作符的左侧实参
class Widget {
public:
    ...
    Widget& operator= (const Widget& rhs)       // 返回类型是个 reference
    {                                           // 指向当前对象
        ...
        return *this;                           // 返回左侧对象
    }
    ...
};

// 这个协议不仅适用于以上的标准赋值形式，也适用于所有赋值相关运算，例如：
class Widget {
public:
    ...
    Widget& operator+= (const Widget& rhs)      // 这个协议适用于
    {                                           // +=，-=，*=， 等等。
        ...
        return *this;
    }
    Widget& operator= (int rhs)     // 此函数也适用，即使
    {                               // 此一操作符的参数类型
        ...                         // 不符协定。
        return *this;
    }
    ...
};
```

<br>

### 条款 11：在 operator= 中处理“自我赋值”

- 确保当对象自我赋值时 operator= 有良好行为。其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序、以及 copy-and-swap。

- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

``` cpp {.line-numbers}
class Widget { ... };
Widget w;
...
w = w;              // 赋值给自己

a[i] = a[j];        // 潜在的自我赋值
// 如果 i 和 j 有相同的值，这便是个自我赋值

*px = *py;          // 潜在的自我赋值
// 如果 px 和 py 恰巧指向同一个东西，这也是自我赋值。

// 一个 base class 的 reference 或 pointer 可以指向一个 derived class 对象：
class Base { ... };
class Derived : public Base { ... };
void doSomething(const Base& rb, Derived* pd);  // rb 和 *pd 有可能其实是同一个对象





class Bitmap { ... };
class Widget {
    ...
private:
    Bitmap* pb;         // 指针，指向一个从 heap 分配而得的对象
}

// 下面是 operator= 实现代码，表面上看起来合理，但自我赋值出现时并不安全
// （它也不具备异常安全性，但我们稍后才讨论这个主题）。
Widget& Widget::operator= (const Widget& rhs)   // 一份不安全的 operator= 实现版本
{
    delete pd;                                  // 停止使用当前的 bitmap，
    pb = new Bitmap(*rhs.pb);                   // 使用 rhs's bitmap 的副本（复件）
    return *this;                               // 见条款 10。
}

// 这里的自我赋值问题是，operator= 函数内的 *this（赋值的目的端）和 rhs 有
// 可能是同一个对象。果真如此 delete 就不只是销毁当前对象的 bitmap，
// 它也销毁 rhs 的 bitmap。在函数末尾，Widget——它原本不该被自我赋值动作
// 改变的——发现自己持有一个指针指向一个已被删除的对象！

// 欲阻止这种错误，传统做法是藉由 operator= 最前面的一个“证同测试（identity test）”
// 达到“自我赋值”的检验目的：
Widget& Widget::operator= (const Widget& rhs)
{
    if (this == &rhs) return *this;     // 证同测试（identity test）：
                                        // 如果是自我赋值，就不做任何事。
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
// 这个新版本仍然存在异常方面的麻烦。
// 如果“new Bitmap”导致异常（不论是因为分配时内存不足或因为 Bitmap 的
// copy 构造函数抛出异常），Widget 最终会持有一个指针指向一块被删除的 Bitmap。
// 你无法安全地删除它们，甚至无法安全地读取它们。

// 我们只需注意在赋值 pb 所指东西之前别删除 pb：
Widget& Widget::operator= (const Widget& rhs)
{
    Bitmap* pOrig = pb;             // 记住原先的 pb
    pb = new Bitmap(*rhs.pb);       // 令 pb 指向 *pb 的一个附件（副本）
    delete pOrig;                   // 删除原先的 pb
    return *this;
}
// 我们对原 bitmap 做了一份复件、删除原 bitmap、然后指向新制造的那个复件。





// 如果你很关心效率，可以把“证同测试”（identity test）再次放回函数起始处，
// 然而这样做之前先问问自己，你估计“自我赋值”的发生频率有多高？因为这项测试
// 也需要成本。
// 在 operator= 函数内手工排列的语句（确保代码不但“异常安全”而且“”自我赋值安全）
// 的一个替代方案是，使用所谓的 copy and swap 技术。
class Widget {
    ...
    void swap(Widget& rhs);         // 交换 *this 和 rhs 的数据；详见条款 29
    ...
};
Widget& Widget::operator= (const Widget& rhs)
{
    Widget temp(rhs);       // 为 rhs 数据制作一份复件（副本）
    swap(temp);             // 将 *this 数据和上述复件的数据交换，
    return *this;
}

// 利用以下事实：
// （1）某 class 的 copy assignment 操作符可能被声明为“以 by value 方式接受实参”；
// （2）以 by value 方式传递东西会造成一份复件/副本（见条款 20）：
Widget& Widget::operator= (Widget rhs)      // rhs 是被传对象的一份复件（副本）
{                                           // 注意这里是 pass by value.
    swap(this);                             // 将 *this 的数据和复件/副本的数据互换
    return *this;
}
// 我个人比较忧虑这种做法，我认为它为了伶俐巧妙的修补而牺牲了清晰性。
// 然而将“copying 动作”从函数本体内移至“函数参数构造阶段”却可令编译
// 器有时生成更高效的代码
```

<br>

### 条款 12：复制对象时勿忘其每一个成分

- Copying 函数应该确保赋值“对象内的所有成员变量”及“所有 base class 成分”。
- 不要尝试以某个 copying 函数实现另一个 copying 函数。应该将共同机能放进第三个函数中，并有两个 copying 函数共同调用。

``` cpp {.line-numbers}
void logCall(const std::string& funcName);      // 制造一个 log entry
class Customer {
public:
    ...
    Customer(const Customer& rhs);
    Customer& operator= (const Customer& rhs);
    ...
private:
    std::string name;
};

Customer::Customer(const Customer& rhs)
    : name(this.name)                           // 复制 rhs 的数据
{
    logCall("Customer copy constructor");
}

Customer& Customer::operator= (const Customer& rhs)
{
    logCall("Customer copy assignment operator");
    name = rhs.name;                            // 复制 rhs 的数据
    return *this;                               // 见条款 10
}

// 这里的每一件事情看起来都很好，而实际上每件事情也的确都好，直到另一个成员变量加入战局：
class Date { ... };                 // 日期
class Customer {
public:
    ...                             // 同前
private:
    std::string name;
    Date lastTransaction;
};
// 如果你为 class 添加一个成员变量，你必须同时修改 copying 函数。
// （你也需要修改 class 的所有构造函数以及任何非标准形式的 operator=）





class PriorityCustomer : public Customer {      // 一个 derived class
public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator= (const PriorityCustomer& rhs);
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
    : priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer& PriorityCustomer::operator= (const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
// PriorityCustomer 的 copying 函数复制了 PriorityCustomer 声明的成员变量，
// 但每个 PriorityCustomer 还内含它所继承的 Customer 成员变量复件（副本），
// 而那些成员变量却未被复制。

// 你应该让 derived class 的 copying 函数调用相应的 base class 函数：
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
    : Customer(rhs),        // 调用 base class 的 copy 构造函数
      priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator= (const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs);       // 对 base class 成分进行赋值动作
    priority = rhs.priority;
    return *this;
}

// 当你编写一个 copying 函数，请确保（1）复制所有 local 成员变量，
// （2）调用所有 base classes 内的适当的 copying 函数。

// 令 copy assignment 操作符调用 copy 构造函数是不合理的
// 令 copy 构造函数调用 copy assignment 操作符同样无意义

// 如果你发现你的 copy 构造函数和 copy assignment 操作符有相近的代码，
// 消除重复代码的做法是，建立一个新的成员函数给两者调用。
// 这样的函数往往是 private 而且常被命名为 init。
// 这个策略可以安全消除 copy 构造函数和 copy assignment 操作符
// 之间的代码重复。
```

<br>

## 3、资源管理

<br>

### 条款 13：以对象管理资源

- 为防止资源泄露，请使用 RAII 对象，它们在构造函数中获得资源并在析构函数中释放资源。
- 两个常被使用的 RAII classes 分别是 tr1::shared_ptr 和 auto_ptr。前者通常是较佳选择，因为其 copy 行为比较直观。若选择 auto_ptr，复制动作会使它（被复制物）指向 null。

``` cpp {.line-numbers}
class Investment { ... };       // “投资类型”继承体系中的 root class

Investment* createInvestment();     // 返回指针，指向 Investment 继承
                                    // 体系内的动态分配对象。调用者有责任删除它。
                                    // 这里为了简化，刻意不写参数。

void f()
{
    Investment* pInv = createInvestment();      // 调用 factory 函数
    ...
    delete pInv;                                // 释放 pInv 所指对象
}
// 这看起来妥当，但若干情况下 f 可能无法删除它得自 createInvestment 的投资对象——
// 或许因为“...”区域内的一个过早的 return 语句。如果这样一个 return 被执行起来，
// 控制流就绝不会触及 delete 语句。

// 为确保 createInvestment 返回的资源总是被释放，我们需要将资源放进对象内，
// 当控制流离开 f，该对象的析构函数会自动释放那些资源。





// 下面示范如何使用 auto_ptr 以避免 f 函数潜在的资源泄漏可能性：
void f()
{
    std::auto_ptr<Investment> pInv(createInvestment());
                        // 调用 factory 函数
    ...                 // 一如以往地使用 pInv
}                       // 经由 auto_ptr 的析构函数自动删除 pInv





// 如果对象会被删除一次以上，为了预防这个问题，auto_ptrs 有一个不寻常的性质，
// 若通过 copy 构造函数或 copy assignment 操作符复制它们，它们会变成 null，
// 而复制所得的指针将取得资源的唯一拥有权！
std::auto_ptr<Investment> pInv1(createInvestment())   // pInv1 指向
                                                      // createInvestment 返回物。
std::auto_ptr<Investment> pInv2(pInv1);     // 现在 pInv2 指向对象，
                                            // pInv1 被设为 null，
pInv1 = pInv2;                              // 现在 pInv1 指向对象，
                                            // pInv2 被设为 null





// TR1 的 tr1::shared_ptr（见条款 54）就是个 RCSP，所以你可以这么写 f：
void f()
{
    ...
    std::tr1::shared_ptr<Investment> 
    pInv(createInvestment());               // 调用 factory 函数
    ...                                     // 使用 pInv 一如以往
}                               // 经由 shared_ptr 析构函数自动删除 pInv 

void f()
{
    ...
    std::tr1::shared_ptr<Investment>
    pInv1(createInvestment());          // pInv1 指向
                                        // createInvestment 返回物。
    std::tr1::shared_ptr<Investment>
    pInv2(pInv1);                       // pInv1 和 pInv2 指向同一个对象
    pInv1 = pInv2;                      // 同上，无任何改变。
}                                       // pInv1 和 pInv2 被销毁，
                                        // 它们所指的对象也就被自动销毁





// auto_ptr 和 tr1::shared_ptr 两者都在其析构函数内做 delete
// 而不是 delete[] 动作（条款 16 对两者的不同有些描述）。那意味
// 在动态分配而得的 array 身上使用 auto_ptr 或 tr1::shared_ptr 是个馊主意。
std::auto_ptr<std::string>                      // 馊主意！会用上错误的
    aps(new std::string[10]);                   // delete 形式。
std::tr1::shared_ptr<int> spi(new int[1024]);   // 相同问题。
```

<br>

### 条款 14：在资源管理类中小心 copying 行为

- 复制 RAII 对象必须一并复制它所管理的资源，所以资源的 copying 行为决定 RAII 对象的 copying 行为。
- 普遍而常见的 RAII class copying 行为是：抑制 copying、施行引用计数法（reference counting）。不过其他行为也都可能被实现。

``` cpp {.line-numbers}
void lock(Mutex* pm);           // 锁定 pm 所指的互斥器
void unlock(Mutex* pm);         // 将互斥器解除锁定

// 为确保绝不会忘记将一个被锁住的 Mutex 解锁，你可能会希望建立一个
// class 用来管理机锁。这样的 class 的基本结构有 RAII 守则支配，
// 也就是“资源在构造源期间获得，在析构期间释放”。
class Lock {
public:
    explicit Lock(Mutex* pm)
        :mutexPtr(pm)
    { lock(mutexPtr); }                 // 获得资源
    ~Lock() { unlock(mutexPtr); }       // 释放资源
private:
    Mutex* mutexPtr;
};

// 客户对 Lock 的用法符合 RAII 方式：
Mutex m;            // 定义你需要的互斥器
...
{                   // 建立一个区块用来定义 critical section
    Lock m1(&m);    // 锁定互斥器
    ...             // 执行 critical section 内的操作
}                   // 在区块最末尾，自动解除互斥器锁定

// 这很好，但如果 Lock 对象被复制。这会发生什么事？
Lock ml1(&m);       // 锁定 m
Lock ml2(m11);      // 将 ml1 复制到 ml2 身上。这会发生什么事？





// 当一个 RAII 对象被复制，会发什么事？
// 以下两种可能选择：

// 1. 禁止复制
class Lock : private Uncopyable {       // 禁止复制。见条款 6
public:
    ...                                 // 如前
};

// 2. 对底层资源祭出“引用计数法”（reference-count）
class Lock {
public:
    explicit Lock(Mutex* pm)            // 以某个 Mutex 初始化 shared_ptr
        : mutexPtr(pm, unlock)          // 并以 unlock 函数为删除器
    {
        lock(mutexPtr.get());           // 条款 15 谈到“get”
    }
private:
    std::tr1::shared_ptr<Mutex> mutexPtr;   // 使用 shared_ptr
                                            // 替换 raw pointer
};
```
<br>

### 条款 15：在资源管理类中提供对原始资源的访问

- APIs 往往要求访问原始资源（raw resource），所以每一个 RAII class 应该提供一个“取得其所管理之资源”的办法。
- 对原始资源的访问可能经由显式转换或隐式转换。一般而言显式转换比较安全，但隐式转换对客户比较方便。

``` cpp {.line-numbers}
std::tr1::shared_ptr<Investment> pInv(createInvestment());      // 见条款 13

int daysHeld(const Investment* pi);     // 返回投资天数

int days = daysHeld(pInv);              // 错误！

int days = daysHeld(pInv.get());        // 很好，将 pInv 内的原始指针
                                        // 传给 daysHeld





class Investment {                      // investment 继承体系的根类
public:
    bool isTaxFree() const;
    ...
};

Investment* createInvestment();         // factory 函数

std::tr1::shared_ptr<Investment>        // 令 tr1::shared_ptr
    pi1(createInvestment());            // 管理一笔资源。

bool taxable1 = !(pi1->isTaxFree());    // 经由 operator-> 访问资源。
...
std::auto_ptr<Investment> pi2(createInvestment());      // 令 auto_ptr
                                                        // 管理一笔资源。
bool taxable = !((*pi2).isTaxFree());   // 经由 operator* 访问资源
...





FontHandle getFont();                   // 这是个 C API。为求简化暂略参数。

void releaseFont(FontHandle fh);        // 来自同一组 C API

class Font {                            // RAII class
public:
    explicit Font(FontHandle fh)        // 获得资源；
        : f(fh)                         // 采用 pass-by-value
    {  }                                // 因为 C API 这样做。
    ~Font() { releaseFont(f); }         // 释放资源
private:
    Fonthandle f;                       // 原始（raw）字体资源
};

// 假设有大量与字体相关的 C API，它们处理的是 FontHandles，那
// 么“将 Font 对象转换为 FontHandle” 会是一种很频繁的需求。
// Font class 可为此提供一个显示转换函数，像 get 那样：
class Font {
public:
    ...
    FontHandle get() const { return f; }        // 显示转换函数
    ...
};

// 不幸的是这使得客户每当想要使用 API 时就必须调用 get：
void changeFontSize(FontHandle f, int newSize);     // C API
Font f(getFont());
int newFontSize;
...
changeFontSize(f.get(), newFontSize);       // 明白地将 Font 转换为 FontHandle

// 某些程序员可能会认为，如此这般地到处要求显示转换，足以使人们
// 倒尽胃口，不再愿意使用这个 class，从而增加了 泄漏字体的可能
// 性，而 Font class 的主要设计目的就是为了防止资源（字体）泄漏。

// 另一个办法是令 Font 提示隐式转换函数，转型为 FontHandle
class Font {
public:
    ...
    operator FontHandle() const     // 隐式转换函数
    { return f; }
    ...
};

// 这使得客户调用 C API 时比较轻松且自然：
Font f(getFont());
int newFontSize;
...
changeFontSize(f, neFontSize);      // 将 Font 隐式转换为 FontHandle

// 但是这个隐式转换会增加错误发生机会。例如客户可能会在需要
// Font 时意外创建一个 FontHandle：
Font f1(getFont());
...
FontHandle f2 = f1;                 // 原意识要拷贝一个 Font 对象
                                    // 却反而将 f1 隐式转换为其底部的 FontHandle
                                    // 然后才复制它。
```

<br>

### 条款 16：成对使用 new 和 delete 时要采取相同形式

- 如果你在 new 表达式中使用 []，必须在相应的 delete 表达式中也使用 []。如果你在 new 表达式中不使用 []，一定不要在相应的 delete 表达式中使用 []。

``` cpp {.line-numbers}
// 以下动作有什么错？
std::string* stringArray = new std::string[100];
...
delete stringArray;
// 使用了 new，也搭配了对应的 delete。但还是有某样东西完全错误：
// 你的程序行为不明确（未有定义）。最低限度，stringArray 所含的
// 100 个 string 对象中的 99 个不太可能被适当删除，因为它们的析
// 构函数很可能没被调用。





std::string* stringPtr1 = new std::string;
std::string* stringPtr2 = new std::string[100];
...
delete stringPtr1;          // 删除一个对象
delete[] stringPtr2;        // 删除一个由对象组成的数组

// 如果你调用 new 时使用 []，你必须在对应调用 delete 时也使用 []。
// 如果你调用 new 时没有使用 []，那么也不该在对应调用 delete 时使用 []。





// 这个规则对于喜欢使用 typedef 的人也很重要，因为它意味 typedef 作者
// 必须说清楚，当程序员以 new 创建这种 typedef 类型对象时，该以
// 哪一种 delete 形式删除之。考虑下面这个 typedef：
typedef std::string AddressLines[4];        // 每个人的地址有 4 行，
                                            // 每行是一个 string

// 由于 AddressLines 是个数组，如果这样使用 new：
std::string* pal = new AddressLines;        // 注意，“new AddressLines” 返回
                                            // 一个 string*，就像
                                            // “new string[4]”一样。
// 那么就必须匹配“数组形式”的 delete：
delete pal;         // 行为未有定义！
delete[] pal;       // 很好。

```

<br>

### 条款 17：以独立语句将 newed 对象置入智能指针

- 以独立语句将 newed 对象存储于（置入）智能指针内。如果不这样做，一旦异常被抛出，有可能导致难以察觉的资源泄漏。

``` cpp {.line-numbers}
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);

// 现在考虑调用 processWidget：
processWidget(new Widget, priority());
// 它不能通过编译。tr1::shared_ptr 构造函数需要一个原始指针（raw pointer），
// 但该构造函数是个 explicit 构造函数，无法进行隐式转换，将得自“newWidget”的
// 原始指针转换为 processWidget 所要求的 tr1::shared_ptr。
// 如果写成这样就可以通过编译：
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
// 上述调用可能存在泄漏资源。
// 在调用 processWidget 之前，编译器必须创建代码，做以下三件事：
// 调用 priority
// 执行“new Widget”
// 调用 tr1::shared_ptr 构造函数

// 万一对 priority 的调用导致异常，会发生什么事？
// 在此情况下“new Widget”返回的指针将会遗失，因为它尚未被置入
// tr1::shared_ptr 内，后者是我们期盼用来防卫资源泄漏的武器。
// 是的，在对 processWidget 的调用过程中可能引发资源泄漏，
// 因为在“资源被创建（经由“new Widget”）”和“资源被转换为资源
// 管理对象”两个时间点之间有可能发生异常干扰。

// 避免这类问题的方法很简单：使用分离语句，分别写出（1）创建 Widget，
// （2）将它置入一个智能指针内，然后再把那个智能指针传给 processWidget：
std::tr1::shared_ptr<Widget> pw(new Widget);    // 在单独语句内以
                                                // 智能指针存储
                                                // newed 所得对象。
processWidget(pw, priority());
```

<br>

## 4、设计与声明

<br>

### 条款 18：让接口容易被正确使用，不易被误用

- 好的接口很容易被正确使用，不容易被误用。你应该在你的所有接口中努力达成这些性质。
- “促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。
- “阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。
- tr1::shared_ptr 支持定制型删除器（custom deleter）。这可防范 DLL 问题，可被用来自动解除互斥锁（mutexes；见条款 14）等等。

``` cpp {.line-numbers}
class Date {
public:
    Date(int month, int day, int year);
    ...
};

Date d(30, 3, 1995);        // 应该是“3,30”而不是“30,3”

Date d(2, 30, 1995);        // 应该是“3,30”而不是“2,30”





struct Day {
    explicit Day(int d) : val(d) {  }
    int val;
};

struct Month {
    explicit Month(int m) : val(m) {  }
    int val;
};

struct Year {
    explicit Year(int y) : val(y) {  }
    int val;
};

class Date {
public:
    Date(const Month& m, const Day& d, const Year& y);
    ...
};
Date d(30, 3, 1995);                        // 错误！不正确的类型
Date d(Day(30), Month(3), Year(1995));      // 错误！不正确的类型
Date d(Month(3), Day(30), Year(1995));      // OK，类型正确！





// 预先定义所有有效的 Months：
class Month {
public:
    static Month Jan() { return Month(1); }     // 函数，返回有效月份。
    static Month Feb() { return Month(2); }     // 稍后解释为什么。
    ...                                         // 这些是函数而非对象。
    static Month Dec() { return Month(12); }
    ...                                         // 其他成员函数
private:
    explicit Month(int m);                      // 阻止生成新的月份。
    ...                                         // 这是月份专属数据。
};

Date d(Month::Mar(), Day(30), Year(1995));





Investment* createInvestment();     // 来自条款 13：为求简化暂略参数。

std::tr1::shared_ptr<Investment> createInvestment();

// tr1::shared_ptr 提供的某个构造函数接受两个参数：一个是被管
// 的指针，另一个是引用次数变成 0 时将被调用的“删除器”。这启发
// 我们创建一个 null tr1::shared_ptr 并以 getRidOfInvestment
// 作为其删除器，像这样：
std::tr1::shared_ptr<Investment>        // 企图创建一个 null shared_ptr
    pInv(0, getRidOfInvestment);        // 并携带一个自定的删除器。
                                        // 此式无法通过编译。

// tr1::shared_ptr 构造函数坚持其第一参数必须是个指针，而 0 不是指针，是个 int。
// 转型（cast）可以解决这个问题：
std::tr1::shared_ptr<Investment>        // 建立一个 null shared_ptr 并以
    pInv(static_cast<Investment*>(0),   // getRidOfInvestment 为删除器
        getRidOfInvestment);            // 条款 27 提到 static_cast

// 如果我们要实现 createInvestment 使它返回一个 tr1::shared_ptr
// 并夹带 getRidOfInvestment() 函数作为删除器，代码看起来像这样：
std::tr1::shared_ptr<Investment> createInvestment()
{
    std::tr1::shared_ptr<Investment> retVal(static_cast<Investment*>(0),
                                            getRidOfInvestment);
    retVal = ...;       // 令 retVal 指向正确对象
    return retVal;
}





// tr1::shared_ptr 有一个特别好的性质是：它会自动使用它的“每个
// 指针专属的删除器”，因为消除另一个潜在的客户错误：所谓的“cross-DLL problem”
std::tr1::shared_ptr<Investment> createInvestment()
{
    return std::tr1::shared_ptr<Investment>(new Stock);
}
// 返回的那个 tr1::shared_ptr 可被传递给任何其他 DLLs，无需在意 “cross-DLL problem”。
// 这个指向 Stock 的 tr1::shared_ptrs 会追踪记录“当 Stock 的引用次数变成 0 时该调
// 用的那个 DLL's delete”。
```
<br>

### 条款 19：设计 class 犹如设计 type

- Class 的设计就是 type 的设计。在定义一个新 type 之前，请确定你已经考虑过本条款覆盖的所有讨论主题。
    - 新 type 的对象应该如何被创建和销毁？
    - 对象的初始化和对象的赋值该有什么样的差别？
    - 新 type 的对象如果被 passed by value（以值传递），意味着什么？
    - 什么是新 type 的“合法值”？
    - 你的新 type 需要配合某个继承图系（inheritance graph）吗？
    - 你的新 type 需要什么样的转换？
    - 什么样的操作符和函数对此新 type 而言是合理的？
    - 什么样的标准函数应该驳回？
    - 谁该取用新 type 的成员？
    - 什么是新 type 的“未声明接口”（undeclared interface）？
    - 你的新 type 有多么一般化？
    - 你真的需要一个新 type 吗？

<br>

### 条款 20：宁以 pass-by-reference-to-const 替换 pass-by-value

- 尽量以 pass-by-reference-to-const 替换 pass-by-value。前者通常比较高效，并可避免切割问题（slicing problem）。
- 以上规则并不适用于内置类型，以及 STL 的迭代器和函数对象。对它们而言，pass-by-value 往往比较适当。

``` cpp {.line-numbers}
class Person {
public:
    Person();                   // 为求简化，省略参数
    virtual ~Person();          // 条款 7 告诉你为什么它是 virtual
    ...
private:
    std::string name;
    std::string address;
};

class Student : public Person {
public:
    Student();                  // 再次省略参数
    ~Student();
    ...
private:
    std::string schoolName;
    std::string schoolAddress;
};

bool validateStudent(Student s);            // 函数以 by value 方式接受学生
Student plato;                              // 柏拉图，苏格拉底的学生
bool platoIsOK = validateStudent(plato);    // 调用函数

// 无疑地 Student 的 copy 构造函数会被调用，以 plato 为蓝本将 s 初始化。
// 同样明显地，当 validateStudent 返回 s 会被销毁。
// 因此，对此函数而言，参数的传递成本是“一次 Student copy 构造函数调用，
// 加上一次 Student 析构函数调用”。
// 但那还不是整个故事喔。Student 对象内有两个 string 对象，所以每次构造一个
// Student 对象也就构造了两个 string 对象。此外 Student 对象继承自 Person 对象，
// 所以每次构造 Student 对象也必须构造出一个 Person 对象。
// 一个 Person 对象又有两个 string 对象在其中，因此每一次 Person 构造动作
// 又需承担两个 string 构造动作。
// 最终结果是，以 by value 方式床底一个 Student 对象会导致调用一次
// Student copy 构造函数、一次 Person copy 构造函数、四次 string copy 构造函数。
// 当函数内的那个 Student 复件被销毁，每一个构造函数调用动作都需要一个对应
// 的析构函数调用动作。因此，以 by value 方式传递一个 Student 对象，
// 总体成本是“六次构造函数和六次析构函数”！

// 如果有什么方法可以回避所有那些构造和析构动作就太好了。
// 有的，就是 pass by reference-to-const：
bool validateStudent(const Student& s);
// 这种传递方式的效率高得多：没有任何构造函数或析构函数被调用，因为没有任何新对象被创建。





// 以 by reference 方式传递参数也可以避免 slicing（对象切割）问题。
class Window {
public:
    ...
    std::string name() const;           // 返回窗口名称
    virtual void display() const;       // 显示窗口和其内容
};

class WindowWithScrollBars : public Window {
public:
    ...
    virtual void display() const;
};

// 现在假设你希望写个函数打印窗口名称，然后显示该窗口。下面是错误示范：
void printNameAndDisplay(Window w)      // 不正确！参数可能被切割
{
    std::cout << w.name();
    w.display();
}

// 当你调用上述函数并交给它一个 WindowWithScrollBars 对象，会发生什么事呢？
WindowWithScrollBars wwsb;
printNameAndDisplay(wwsb);
// 参数 w 会被构造成为一个 Window 对象；它是 passed by value，还记得吗？
// 而造成 wwsb “之所以是个 WindowWithScrollBars 对象”的所有特化信息都会被切除。
// 在 printNameAndDisplay 函数内不论传递过来的对象原本是什么类型，
// 参数 w 就像一个 Window 对象（因为其类型是 Window）。
// 因此在 printNameAndDisplay 内调用 display 调用的总是 Window::display，
// 绝不会是 WindowWithScrollBars::display。

// 解决切割（slicing）问题的办法，就是以 by reference-to-const 的方式传递 w：
void printNameAndDisplay(const Window& w)       // 很好，参数不会被切割
{
    std::cout << w.name();
    w.display();
}

// 如果窥视 C++ 编译器的底层，你会发现，references 往往以指针实现出来，
// 因此 pass by reference 通常意味真正传递的是指针。
// 因此如果你有个对象属于内置类型（例如 int），pass by value
// 往往比 pass by reference 的效率高些。
```

<br>

### 条款 21：必须返回对象时，别妄想返回其 reference

- 绝不要返回 pointer 或 reference 指向一个 local stack 对象，或返回 reference 指向一个 heap-allocated 对象，或返回 pointer 或 reference 指向一个 local static 对象而有可能同时需要多个这样的对象。条款 4 已经为“在单线程环境中合理返回 reference 指向一个 local static 对象”提供了一份设计实例。

``` cpp {.line-numbers}
class Rational {
public:
    Rational(int number = 0,            // 条款 24 说明为什么这个构造函数
             int denominator = 1);      // 不声明为 explicit
    ...
private:
    int n, d;                   // 分子（numerator）和分母（denominator）
    friend const Rational       // 条款 3 说明为什么返回类型是 const
        operator* (const Rational& lhs, const Rational& rhs);
};

Rational a(1, 2);           // a = 1 / 2
Rational b(3, 5);           // b = 3 / 5
Rational c = a * b;         // c 应该是 3 / 10

// 如果 operator* 要返回一个 reference 指向如此数值，
// 它必须自己创建那个 Rational 对象。





// 函数创建新对象的途径有二：在 stack 空间或在 heap 空间创建之。
// 如果定义一个 local 变量，就是在 stack 空间创建对象。
// 根据这个策略试写 operator* 如下：
const Rational& operator* (const Rational& lhs, const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);      // 警告！糟糕的代码！
    return result;
}

// 这个函数返回一个 reference 指向 result，但 result 是个 local 对象，
// 而 local 对象在函数退出前被销毁了。
// 因此，这个版本的 operator* 并未返回 reference 指向某个 Rational，
// 它返回的 reference 指向一个“从前的” Rational；一个旧时的 Rational；
// 一个曾经被当做 Rational 但如今已经成空、发臭、败坏的残骸，因为它已经被销毁了。

// 任何函数如果返回一个 reference 指向某个 local 对象，都将一败涂地。
// （如果函数返回指针指向一个 local 对象，也是一样。）





// 于是，让我们考虑在 heap 内构造一个对象，并返回 reference 指向它。
// Heap-based 对象由 new 创建，所以你得写一个 heap-based operator* 如下：
const Rational& operator* (const Rational& lhs, const Rational& rhs)
{
    Rational* result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return *result;
}
// 你还是必须付出一个“构造函数调用”代价，因为分配所得的内存将以一个适当的
// 构造函数完成初始化动作。但此外你现在又有另一个问题：
// 谁该对着被你 new 出来的对象实施 delete？

// 即使调用者诚实谨慎，并且处于良好意识，他们还是不太能够在这样
// 合情合理的用法下阻止内存泄漏：
Rational w, x, y, z;
w = x * y * z;              // 与 operator* (operator* (x, y), z) 相同

// 这里，同一个语句内调用了两次 operator*，因而两次使用 new，
// 也就需要两次 delete。但却没有合理的办法让 operator* 使用
// 者进行那些 delete 调用，因为没有合理的办法让他们取得 operator*
// 返回的 references 背后隐藏的那个指针。这绝对导致资源泄漏。





// 让 operator* 返回的是 reference 指向一个被定义与函数内部
// 的 static Rantional 对象：
const Rational& operator* (const Rational& lhs, const Rational& rhs)
{                               // 警告，又一堆烂代码。
    static Rational result;     // static 对象，此函数将返回
                                // 其 reference。
    result = ...;               // 将 lhs 乘以 rhs，并将结果
                                // 置于 result 内。
    return result;
}
// 就像所有用上 static 对象的设计一样，这一个也立刻造成我们对多线程安全性的疑虑。

bool operator== (const Rational& lhs,       // 一个针对 Rationals
                 const Rational& rhs);      // 而写的 operator==
Rational a, b, c, d;
...
if ((a * b) == (c * d)) {
    当乘积相等时，做适当的相应动作；
} else {
    当乘积不等时，做适当的相应动作；
}
// 猜想怎么着？表达式 ((a * b) == (c * d)) 总是
// 被核算为 true，不论 a, b, c 和 d 的值是什么！
if (operator==(operator*(a, b), operator*(c, d)))
// 注意，在 operator== 被调用前，已有两个 operator* 调用式起作用，
// 每一个都返回 reference 指向 operator* 内部定义的 static Rational 对象。
// 因此 operator== 被要求将“operator* 内部定义的 static Rational 对象值”拿来
// 和“operator* 内的 static Rational 对象值”比较，如果比较结果不相等，那才奇怪呢。





// 一个“必须返回新对象”的函数的正确写法是：就让那个函数返回一个新对象呗。
// 对 Rational 的 operator* 而言意味以下写法（或其本质上等价的代码）：
inline const Rational operator* (const Rational& lhs,
                                 const Rational& rhs)
{
    return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

<br>

### 条款 22：将成员变量声明为 private

- 切记将成员变量声明为 private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供 class 作者以充分的实现弹性。
- protected 并不比 public 更具封装性。

``` cpp {.line-numbers}
class AccessLevels {
public:
    ...
    int getReadOnly() const { return readOnly; }
    void setReadWrite(int value) { readWrite = value; }
    int getReadWrite() const { return readWrite; }
    void setWriteOnly(int value) { writeOnly = value; }
private:
    int noAccess;       // 对此 int 无任何访问动作
    int readOnly;       // 对此 int 做只读访问（read-only access）
    int readWrite;      // 对此 int 做读写访问（read-write access）
    int writeOnly;      // 对此 int 做惟写访问（write-only access）
};
// 如此细微地划分访问控制颇有必要，因为许多成员变量应该被隐藏起来。
// 每个成员变量都需要一个 getter 函数和 setter 函数毕竟罕见。





class SpeedDataCollection {
    ...
public:
    void addValue(int speed);           // 添加一笔新数据
    double averageSoFar() const;        // 返回平均速度
    ...
};
// 第一种做法（随时保存平均值）会使得每一个 SpeedDataCollection 对象变大，
// 因为你必须为用来存放目前平均值、累积总量、数据点数的每一个成员变量分配空间。
// 然而 averageSoFar 却可因此而十分高效；它可以只是一个返回目前平均值的 inline 函数。
// 相反地，“被询问才计算平均值”会使得 averageSoFar 执行较慢，
// 但每一个 SpeedDataCollection 对象比较小。

// 重点是，由于通过成员函数来访问平均值（也就是封装了它），你得以替换
// 不同的实现方式（以及其他你可能想到的东西），客户最多只需重新编译。
// （如果遵循条款 31 所描述的技术，你甚至可以消除重新编译的不便性。）





// 一旦你将一个成员变量声明为 public 或 protected 而客户开始使用它，
// 就很难改变那个成员变量所涉及的一切。太多代码需要重写、重新测试、
// 重新编写文档、重新编译。从封装的角度观之，其实只有两种访问权限：
// private（提供封装）和其他（不提供封装）。
```

<br>

### 条款 23：宁以 non-member、non-friend 替换 member 函数

- 宁可拿 non-member non-friend 函数替换 member 函数。这样做可以增加封装性、包裹弹性（packaging flexibility）和机能扩充性。

``` cpp {.line-numbers}
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
    ...
};

// 许多用户会想一整个执行所有这些动作，因此 WebBrowser 也提供这样一个函数：
class WebBrowser {
public:
    ...
    void clearEverything();         // 调用 clearCache，clearHistory，
                                    // 和 removeCookies
    ...
};

// 当然，这一机能也可由一个 non-member 函数调用适当的 member 函数而提供出来：
void clearBrowser(WebBrowser& wb)
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}

// 面向对象守则要求，数据以及操作数据的那些函数应该被捆绑在一块，
// 这意味它建议 member 函数是较好的选择。不幸的是这个建议不正确。
// 这是基于对面向对象真实意义的一个误解。面向对象守则要求数据应该尽可能被封装，
// 然而与直观相反地，member 函数 clearEverything 带来的封装性比
// non-member 函数 clearBrowser 低。此外，提供 non-member 函数
// 可允许对 WebBrowser 相关机能有较大的包裹弹性（packaging flexibility），
// 而那最终导致较低的编译相依度，增加 WebBrowser 的可延伸性。
// 因此在许多方面 non-member 做法比 member 做法好。重要的是，
// 我们必须了解其原因。





// 例如我们可以令 clearBrowser 成为某工具类（utility class）的一个
// static member 函数。只要它不是 WebBrowser 的一部分（或成为其 friend），
// 就不会影响 WebBrowser 的 private 成员封装性。

// 在 C++，比较自然的做法是让 clearBrowser 成为一个 non-member 函数并且位于
// WebBrowser 所在的同一个 namespace（命名空间）内：
namespace WebBrowserStuff {
    class WebBrowser { ... };
    void clearBrowser(WebBrowser& wb);
    ...
}
// 然而这不只是为了看起来自然而已。要知道，namespace 和 classes 不同，
// 前者可跨越多个源码文件而后者不能。这很重要，因为像 clearBrowser
// 这样的函数是个“提供便利的函数”，如果它既不是 members 也不是 friends，
// 就没有对 WebBrowser 的特殊访问权利，也就不能提供
// “WebBrowser 客户无法以其他方式取得”的机能。
// 举个例子，如果 clearBrowser 不存在，客户端就只好自行调用
// clearCache，clearHistory 和 removeCookies。





// 分离它们的最直接做法就是将书签相关便利函数声明于一个头文件，
// 将 cookie 相关便利函数声明于另一个头文件，再将打印相关便利
// 函数声明于第三个头文件，依次类推：

// 头文件“webbrowser.h”——这个头文件针对 class WebBrowser 自身
// 及 WebBrowser 核心机能。
namespace WebBrowserStuff {
class WebBrowser { ... };
    ...                     // 核心机能，例如几乎所有客户都需要的
                            // non-member 函数。
}

// 头文件“webbrowserbookmarks.h”
namespace WebBrowserStuff {
    ...                     // 与书签相关的便利函数
}

// 头文件“webbrowsercookies.h”
namespace WebBrowserStuff {
    ...                     // 与 cookie 相关的便利函数
}
...





// 将所有便利函数放在多个头文件内但隶属同一个命名空间，
// 意味客户可以轻松扩展这一组便利函数。他们需要做的就是添加
// 更多 non-member non-friend 函数到此命名空间内。
```

<br>

### 条款 24：若所有参数皆需类型转换，请为此采用 non-member 函数

- 如果你需要为某个函数的所有参数（包括被 this 指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个 non-member。

``` cpp {.line-numbers}
class Rational {
public:
    Rational(int numerator = 0,         // 构造函数刻意不为 explicit；
             int denominator = 1);      // 允许 int-to-Rational 隐式转换。
    int numerator() const;              // 分子（numerator）和分母（denominator）
    int denominator() const;            // 的访问函数（accessors）—— 见条款 22。
private:
    ...
};





class Rational {
public:
    ...
    const Rational operator* (const Rational& rhs) const;
};

// 这个设计使你能够将两个有理数以最轻松自在的方式相乘：
Rational oneEighth(1, 8);
Rational oneHalf(1, 2);
Rational result = oneHalf * oneEighth;      // 很好
result = result * oneEighth                 // 很好

// 然而当你尝试混合式算术，你发现只有一半行得通：
result = oneHalf * 2;           // 很好
result = 2 * oneHalf;           // 错误！

// 当你以对应的函数形式重写上述两个式子，问题所在便一目了然了：
result = oneHalf.operator*(2);      // 很好
result = 2.operator*(oneHalf);      // 错误！

// 是的，oneHalf 是一个内含 operator* 函数的 class 的对象，
// 所以编译器调用该函数。然而整数 2 并没有相应的 class，也就
// 没有 operator* 成员函数。编译器也会尝试寻找可被以下这般调
// 用的 non-member operator*（也就是在命名空间内或在 global 作用域内）：
result = operator*(2, oneHalf);     // 错误！
// 但本例并不存在这样一个接受 int 和 Rational 作为参数的 non-member
// operator*，因此查找失败。





// 再次看看先前成功的那个调用。注意其第二参数是整数 2，但
// Rational::operator* 需要的实参却是个 Rational 对象。
const Rational temp(2);     // 根据 2 建立一个暂时性的 Rational 对象。
result = oneHalf * temp;    // 等同于 oneHalf.operator*(temp)

// 当然，只因为涉及 non-explicit 构造函数，编译器才会这么做。
// 如果 Rational 构造函数是 explicit，以下语句没有一个可通过编译：
result = oneHalf * 2;       // 错误！（在 explicit 构造函数的情况下）
                            // 无法将 2 转换为一个 Rational。
result = 2 * oneHalf;       // 一样的错误，一样的问题。





result = oneHalf * 2;       // 没问题（在 non-explicit 构造函数的情况下）
result = 2 * oneHalf;       // 错误！（甚至在 non-explicit 构造函数的情况下）
// 结论是，只有当参数被列于参数列（parameter list）内，
// 这个参数才是隐式类型转换的合格参与者。





// 让 operator* 成为一个 non-member 函数，便允许编译器在
// 每一个实参身上执行隐式类型转换：
class Rational {
    ...                         // 不包括 operator*
};

const Rational operator* (const Rational& lhs,      // 现在成了一个
                          const Rational& rhs)      // non-member 函数
{
    return Rational(lhs.numerator() * rhs.numerator(),
                    lhs.denominator() * rhs.denominator());
}

Rational oneFourth(1, 4);
Rational result;
result = oneFourth * 2;     // 没问题
result = 2 * oneFourth;     // 万岁，通过编译了！

// 这当然是个快乐的结局，不过还有一点必须操心：operator* 是否应
// 该成为 Rational class 的一个 friend 函数呢？
// 就本例而言答案是否定的，因为 operator* 可以完全藉由 Rational
// 的 public 接口完成任务，上面代码已表明此种做法。

```

<br>

### 条款 25：考虑写出一个不抛异常的 swap 函数

- 当 std::swap 对你的类型效率不高时，提供一个 swap 成员函数，并确定这个函数不抛出异常。
- 如果你提供一个 member swap，也该提供一个 non-member swap 用来调用其前者。对于 classes（而非 templates），也请特化 std::swap。
- 调用 swap 时应针对 std::swap 使用 using 声明式，然后调用 swap 并且不带任何“命名空间资格修饰”。
- 为“用户定义类型”进行 std templates 全特化是好的，但千万不要尝试在 std 内加入某些对 std 而言全新的东西。

``` cpp {.line-numbers}
namespace std {
    template<typename T>        // std::swap 的典型实现；
    void swap(T& a, T& b)       // 置换 a 和 b 的值。
    {
        T temp(a);
        a = b;
        b = temp;
    }
};


// 其中最主要的就是“以指针指向一个对象，内含真正数据”那种类型。
// 这种设计的常见表现形式是所谓“pimpl 手法”
// （pimpl 是 “pointer to implementation”的缩写，见条款 31）。
// 如果以这种手法设计 Widget class，看起来会像这样：
class WidgetImpl {              // 针对 Widget 数据而设计的 class；
public:                         // 细节不重要。
    ...
private:
    int a, b, c;                // 可能有许多数据，
    std::vector<double> v;      // 意味复制时间很长。
    ...
};

class Widget {                                  // 这个 class 使用 pimpl 手法
public:
    Widget(const Widget& rhs);
    Widget& operator= (const Widget& rhs)       // 复制 Widget 时，令它赋值其
    {                                           // WidgetImpl 对象。
        ...                                     // 关于 operator= 的一般性实现
        *pImpl = *(rhs.pImpl);                  // 细节，见条款 10，11 和 12.
        ...
    }
    ...
private:
    WidgetImpl* pImpl;                          // 指针，所指对象内含
                                                // Widget 数据。
};

// 一旦要置换两个 Widget 对象值，我们唯一需要做的就是置换其
// pImpl 指针，但缺省的 swap 算法不知道这一点。它不只复制三
// 个 Widgets，还复制三个 WidgetImpl 对象。非常缺乏效率！

// 我们希望能够告诉 std::swap：当 Widgets 被置换时真正该做的是
// 置换其内部的 pImpl 指针。确切实践这个思路的一个做法是：将 std::swap
// 针对 Widget 特化。下面是基本构想，但目前这个形式无法通过编译：
namespace std {
    template<>                          // 这是 std::swap 针对
    void swap<Widget> (Widget& a,       // “T 是 Widget”的特化版本。
                       Widget& b)       // 目前还不能通过编译。
    {
        swap(a.pImpl, b.pImpl);         // 置换 Widgets 时只要置换它们的
                                        // pImpl 指针就好。
    }
}
// 这个函数一开始的“template<>”表示它是 std::swap 的一个全特化
// （total template specialization）版本，函数名称之后的
// “<Widget>”表示这一特化版本系针对“T 是 Widget”而设计。

// 这个函数无法通过编译。因为它企图访问 a 和 b 内的 pImpl 指针，
// 而那却是 private。我们可以将这个特化版本声明为 friend，
// 但和以往的规矩不太一样：我们令 Widget 声明一个名为 swap 的
// public 成员函数做真正的置换工作，然后将 std::swap 特化，
// 令它调用该成员函数：
class Widget {                          // 与前同，唯一差别是增加 swap 函数
public:
    ...
    void swap(Widget& other)
    {
        using std::swap;                // 这个声明之所以必要，稍后解释。
        swap(pImpl, other.pImpl);       // 若要置换 Widgets 就置换其 pImpl 指针。
    }
    ...
};

namespace std {
    template<>                          // 修订后的 std::swap 特化版本
    void swap<Widget>(Widget& a,
                      Widget& b)
    {
        a.swap(b);                      // 若要置换 Widgets，调用其
    }                                   // swap 成员函数。
}
// 这种做法不只能够通过编译，还与 STL 容器有一致性，因为所有 STL
// 容器也都提供有 public swap 成员函数和 std::swap 特化版本（用以调用前者）。





// 然而假设 Widget 和 WidgetImpl 都是 class templates 而非 classes，
// 也许我们可以试试将 WidgetImpl 内的数据类型加以参数化：
template<typename T>
class WidgetImpl { ... };

template<typename T>
class Widget { ... };

// 在 Widget 内（以及 WidgetImpl 内，如果需要的话）放个 swap
// 成员函数就像以往一样简单，但我们却在特化 std::swap 时遇上乱流。
// 我们想写成这样：
namespace std {
    template<typename T>
    void swap<Widget<T>>(Widget<T>& a,      // 错误！不合法！
                         Widget<T>& b)
    { a.swap(b); }
}

// 看起来合情合理，却不合法。是这样的，我们企图偏特化（partially specialize）
// 一个 function template（std::swap），但 C++ 只允许对 class templates
// 偏特化，在 function templates 身上偏特化是行不通的。
// 这段代码不该通过编译（虽然有些编译器错误地接受了它）。





// 当你打算偏特化一个 function template 时，惯常做法是简单
// 地为它添加一个重载版本，像这样：
namespace std {
    template<typename T>        // std::swap 的一个重载版本
    void swap(Widget<T>& a,     // （注意“swap”之后没有“<...>”）
              Widget<T>& b)     // 稍后我会告诉你，这也不合法。
    { a.swap(b); }
}
// 客户可以全特化 std 内的 templates，但不可以添加新的 templates
// （或 classes 或 functions 或其他任何东西）到 std 里头。





// 我们还是声明一个 non-member swap 让它调用 member swap，
// 但不再将那个 non-member swap 声明为 std::swap 的特化版本
// 或重载版本。
namespace WidgetStuff {
    ...                             // 模板化的 WidgetImpl 等等，
    template<typename T>            // 同前，内含 swap 成员函数
    class Widget { ... };
    ...
    template<typename T>            // non-member swap 函数
    void swap(Widget<T>& a,         // 这里并不属于 std 命名空间。
              Widget<T>& b)
    {
        a.swap(b);
    }
}





template<typename T>
void doSomething(T& obj1, T& obj2)
{
    ...
    swap(obj1, obj2);
    ...
}
// 你希望的应该是调用 T 专属版本，并在该版本不存在的情况下调用 std
// 内的一般化版本。下面是你希望发生的事：
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    using std::swap;        // 令 std::swap 在此函数内可用
    ...
    swap(obj1, obj2);       // 为 T 型对象调用最佳 swap 版本
    ...
}
// 一旦编译器看到对 swap 的调用，它们便查找适当的 swap 并调用之。
// C++ 的名称查找法则（name lookup rules）确保将找到 global
// 作用域或 T 所在之命名空间内的任何 T 专属的 swap。如果 T 是
// Widget 并位于命名空间 WidgetStuff 内，编译器会使用“实参取
// 决之查找规则”（argument-dependent lookup）找出 WidgetStuff 内
// 的 swap。如果没有 T 专属之 swap 存在，编译器就使用 std 内的 swap，
// 这得感谢 using 声明式让 std::swap 在函数内曝光。然而即便如此
// 编译器还是比较喜欢 std::swap 的 T 专属特化版，而非一般化的
// 那个 template，所以如果你已针对 T 将 std::swap 特化，特化版
// 会被编译器挑中。

std::swap(obj1, obj2);          // 这是错误的 swap 调用方式
// 这便强迫编译器只认 std 内的 swap（包括其任何 template 特
// 化），因而不再可能调用一个定义于它处的较适当 T 专属版本。
```

<br>

## 5、实现

<br>

### 条款 26：尽可能延后变量定义式的出现时间

- 尽可能延后变量定义式的出现。这样做可增加程序的清晰度并改善程序效率。

``` cpp {.line-numbers}
// 只要你定义了一个变量而其类型有一个构造函数或析构函数，
// 那么当程序的控制流（control flow）到达这个变量定义式时，你便得承受构造成本；
// 当这个变量离开其作用域时，你便得承受析构成本。
// 即使这个变量最终并未被使用，仍需耗费这些成本，所以你应该尽可能避免这种情形。

// 这个函数过早定义变量“encrypted”
std::string encryptPassword(cons std::string& password)
{
    using namespace std;
    string encrypted;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ...             // 必要动作，俾能将一个加密后的密码
                    // 置入变量 encrypted 内
    return encrypted;
}

// 对象 encrypted 在此函数中并非完全未被使用，但如果有个异常被丢出，
// 它就真的没被使用。也就是说如果函数 encryptedPassword 丢出异常，
// 你仍得付出 encrypted 的构造成本和析构成本。所以最好延后 encrypted 的定义式，
// 直到确实需要它：

// 这个函数延后“encrypted”的定义，直到真正需要它
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    string encrypted;
    ...                 // 必要动作，俾能将一个加密后的密码
                        // 置入变量 encrypted 内
    return encrypted;
}





// 但这段代码仍然不够秾纤和度，因为 encrypted 虽然定义却无任何实参作为初值。
void encrypt(std::string& s);       // 在其中的适当地点对 s 加密

// 于是 encryptPassword 可实现如下，虽然还不算是最好的做法：

// 这个函数延后“encrypted”的定义，直到需要它为止。
// 但此函数仍然有着不该有的效率低落。
std::string encryptPassword(const std::string& password)
{
    ...                         // 检查 length，如前。
    std::string encrypted;      // default-construct encrypted
    encrypted = password;       // 赋值给 encrypted
    encrypt(encrypted);
    return encrypted;
}

// 更受欢迎的做法是以 password 作为 encrypted 的初值，
// 跳过毫无意义的 default 构造过程：

// 终于，这是定义并初始化 encrypted 的最佳做法
std::string encryptPassword(const std::string& password)
{
    ...                                 // 检查长度
    std::string encrypted(password);    // 通过 copy 构造函数
                                        // 定义并初始化。
    encrypt(encrypted);
    return encrypted;
}





// 方法 A：定义于循环外
Widget w;
for (int i = 0; i < n; ++i) {
    w = 取决于 i 的某个值；
    ...
}

// 方法 B：定义于循环内
for (int i = 0; i < n; ++i) {
    Widget w(取决于 i 的某个值);
    ...
}

// 这里我把对象的类型从 string 改为 Widget，
// 以免造成读者对于“对象执行构造、析构、或赋值动作所需的成本” 有任何特殊偏见。
// 在 Widget 函数内部，以上两种写法的成本如下：
// 做法 A：1 个构造函数 + 1 个析构函数 + n 个赋值操作
// 做法 B：n 个构造函数 + n 个析构函数

// 如果 classes 的一个赋值成本低于一组构造 + 析构成本，做法 A 大体而言比较高效。
// 尤其当 n 值很大的时候。否则做法 B 或许比较好。此外做法 A 造成名称 w 的作用域
// （覆盖整个循环）比做法 B 更大，有时那对程序的可理解性和易维护性造成冲突。
// 因此除非（1）你知道赋值成本比“构造 + 析构”成本低，
// （2）你正在处理代码中效率高度敏感（performance-sensitive）的部分，
// 否则你应该使用做法 B。
```

<br>

### 条款 27：尽量少做转型动作

- 如果可以，尽量避免转型，特别是在注重效率的代码中避免 dynamic_casts。如果有个设计需要转型动作，试着发展无需转型的替代设计。
- 如果转型是必要的，试着将它隐藏于某个函数背后。客户随后可以调用该函数，而不需要将转型放进他们自己的代码内。
- 宁可使用 C++-style（新式）转型，不要使用旧式转型。前者很容易辨识出来，而且也比较有着分门别类的职掌。

``` cpp {.line-numbers}
// C 风格的转型动作看起来像这样；
(T)expression       // 将 expression 转型为 T

// 函数风格的转型动作看起来像这样：
T(expression)       // 将 expression 转型为 T

// 两种形式并无差别，纯粹只是小括号的摆放位置不同而已。
// 我称此二种形式为“旧式转型”（old-style casts）。

// C++ 还提供四种新式转型（常常被称为 new-style 或 C++-style casts）：
const_cast<T>( expression )
dynamic_cast<T>( expression )
reinterpret_cast<T>( expression )
static_cast<T>( expression )

// 各有不同的目的：

// const_cast 通常被用来将对象的常量性转除（cast away the constness）。
// 它也是唯一有此能力的 C++-style 转型操作符。

// dynamic_cast 主要用来执行“安全向下转型”（safe downcasting），也就是用来
// 决定某对象是否归属继承体系中的某个类型。它是唯一无法由旧式语法执行的动作，
// 也是唯一可能耗费重大运行成本的转型动作（稍后细谈）。

// reinterpret_cast 意图执行低级转型，实际动作（及结果）可能取决于编译器，
// 这也就表示它不可移植。例如将一个 pointer to int 转型为一个 int。这一类转型
// 在低级代码以外很少见。本书只使用一次，那是在讨论如何针对原始内存（raw memory）
// 写出一个调试用的分配器（debugging allocator）时，见条款 50。

// static_cast 用来强迫隐式转换（implicit conversions），例如将 non-const
// 对象转为 const 对象（就像条款 3 所为），或将 int 转为 double 等等。它也可以
// 用来执行上述多种转换的反向转换，例如将 void* 指针转为 typed 指针，
// 将 pointer-to-base 转为 pointer-to-derived。
// 但它无法将 const 转为 non-const —— 这个只有 const_cast 才办得到。





// 旧式转型仍然合法，但新式转型较受欢迎。原因是：第一，它们很容易在代码中被辨识出来
// （不论是人工辨识或使用工具如 grep），因此得以简化“找出类型系统在哪个地点被破坏”
// 的过程。第二，各转型动作的目标愈窄化，编译器愈可能诊断出错误的运用。举个例子，
// 果你打算将常量性（constness）去掉，除非使用新式转型中的 const_cast，否则无法
// 通过编译。





class Widget {
public:
    explicit Widget(int size);
    ...
};

void doSomeWork(const Widget& w);
doSomeWork(Widget(15));                 // 以一个 int 加上“函数风格”的
                                        // 转型动作创建一个 Widget。
doSomeWork(static _cast<Widget>(15));   // 以一个 int 加上“C++ 风格”的
                                        // 转型动作创建一个 Widget。





int x, y;
...
double d = static_cast<double>(x) / y;      // x 除以 y，使用浮点数除法

// 将 int x 转型为 double 几乎肯定会产生一些代码，因为在大部分计算器体系结构中，
// int 的底层表述不同于 double 的底层表述。这或许不会让你惊讶，
// 但下面这个例子就有可能让你稍微睁大眼睛了：
class Base { ... };
class Derived : public Base { ... };
Derived d;
Base* pb = &d;                              // 隐喻地将 Derived* 转换为 Base*





// 另一件关于转型的有趣事情是：我们很容易写出某些似是而非的代码
// （在其他语言中也许真是对的）。例如许多应用框架（application frameworks）
// 都要求 derived classes 内的 virtual 函数代码的第一个动作
// 就先调用 base class 的对应函数。假设我们有个 Window base class
// 和一个 SpecialWindow derived class，两者都定义了 virtual 函数
// onResize。进一步假设 SpecialWindow 的 onResize 函数被要求首先调用 Window
// 的 onResize。下面是实现方式之一，它看起来对，但实际上错：
class Window {                                  // base class
public:
    virtual void onResize() { ... }             // base onResize 实现代码
    ...
};

class SpecialWindow : public Window {           // derived class
public:
    virtual void onResize() {                   // derived onResize 实现代码
        static_cast<Window>(*this).onResize();  // 将 *this 转型为 Window，
                                                // 然后调用其 onResize；
                                                // 这可不行！
        ... // 这里进行 SpecialWindow 专属行为。
    }
    ...
};

// 上述代码并非在当前对象身上调用 Window::onResize 之后又在该对象身上
// 执行 SpecialWindow 专属动作。不，它是在“当前对象之 base class 成分”的副本上
// 调用 Window::onResize，然后在当前对象身上执行 SpecialWindow 专属动作。
// 如果 Window::onResize 修改了对象内容（不能说没有可能性，因为
// onResize 是个 non-const 成员函数），当前对象其实没被改动，改动的是副本。
// 然而 SpecialWindow::onResize 内如果也修改对象，当前对象真的会被改动。
// 这使当前对象进入一种“伤残”转状态：其 base class 成分的更改没有落实，
// 而 derived class 成分的更改倒是落实了。

// 解决之道是拿掉转型动作，你只是想调用 base class 版本的 onResize 函数，
// 令它作用于当前对象身上，所以请这么写：
class SpecialWindow : public Window {
public:
    virtual void onResize() {
        Window::onResize();         // 调用 Window::onResize 作用于 *this 身上
        ...
    }
    ...
};





// 之所以需要 dynamic_cast，通常是因为你想在一个你认定为 derived class 对象身上执行
// derived class 操作函数，但你的手上却只有一个“指向 base”的 pointer 或 reference，
// 你只能靠它们来处理对象。有两个一般性做法可以避免这个问题。

// 第一，使用容器并在其中存储直接指向 derived class 对象的指针
// （通常是智能指针，见条款 13），如此便消除了“通过 base class 接口处理对象”的需要。
// 假设先前的 Window / SpecialWindow 继承体系中只有 SpecialWindows 才支持闪烁效果，
// 试着不要这样做：
class Window { ... };
class SpecialWindow : public Window {
public:
    void blink();
    ...
};

typedef                                         // 关于 tr1::shared_ptr
std::vector<std::tr1::shared_ptr<Window>> VPW;  // 见条款 13。
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin();      // 不希望使用
     iter != winPtrs.end(); ++iter) {            // dynamic_cast
    if (SpecialWindow* psw = dynamic_cast<SpecialWindow*>(iter->get()))
        psw->blink();
}

// 应该改而这样做：
typedef std::vector<std::tr1::shared_ptr<SpecialWindow>> VPSW;
VPSW winPtrs;
...
for (VPSW::iterator iter = winPtrs.begin();     // 这样写比较好
     iter != winPtrs.end();                     // 不使用 dynamic_cast
     ++iter)
     (*iter)->blink();

// 当然啦，这种做法使你无法在同一个容器内存储指针“指向所有可能之各种 Window 派生类”。
// 如果真要处理多种窗口类型，你可能需要多个容器，它们都必须具备类型安全性（type-safe）。

// 另一种做法可让你通过 base class 接口处理“所有可能之各种 window 派生类”，
// 那就是在 base class 内提供 virtual 函数做你想对各个 Window 派生类做的事。
// 举个例子，虽然只有 SpecialWindows 可以闪烁，但或许将闪烁函数声明于 base class 内
// 并提供一份“什么也没做”的缺省实现码是有意义的：
class Window {
public:
    virtual void blink() {  }       // 缺省实现代码“什么也没做”；
    ...                             // 条款 34 告诉你为什么
};                                  // 缺省实现代码可能是个馊主意。

class SpecialWindow : public Window {
public:
    virtual void blink() { ... }    // 在此 class 内，
    ...                             // blink 做某些事。
};

typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
VPW winPtrs;                        // 容器，内含指针，指向
...                                 // 所有可能的 Window 类型。
for (VPW::iterator iter = winPtrs.begin();
     iter != winPtrs.end();
     ++iter)                        // 注意，这里没有
    (*iter)->blink();               // dynamic_cast。

// 不论哪一种写法——“使用类型安全容器”或“将 virtual 函数往继承体系上方移动”——
// 都并非放之四海皆准，但在许多情况下它们都提供一个可行的 dynamic_cast 替代方案。





// 绝对必须避免的一件事是所谓的“连串（cascading）dynamic_casts”，
// 也就是看起来像这样的东西：
class Window { ... };
...                                 // derived classes 定义在这里
typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin();
     iter != winPtrs.end(); ++iter)
{
    if (SpecialWindow* psw1 = 
       dynamic_cast<SpecialWindow1*>(iter->get())) { ... }
    else if (SpecialWindow* psw2 = 
       dynamic_cast<SpecialWindow2*>(iter->get())) { ... }
    else if (SpecialWindow* psw3 = 
       dynamic_cast<SpecialWindow3*>(iter->get())) { ... }
    ...
}
```

<br>

### 条款 28：避免返回 handles 指向对象内部成分

- 避免返回 handles（包括 references、指针、迭代器）指向对象内部。遵守这个条款，帮助 const 成员函数的行为像个 const，并将发生“虚吊号码牌”（dangling handles）的可能性降至最低。

``` cpp {.line-numbers}
// 假设你的程序设计矩形。每个矩形由其左上角和右下角表示。
// 为了让一个 Rectangle 对象尽可能小，你可能会决定不把定义矩形的这些点存放在
// Rectangle 对象内，而是放在一个辅助的 struct 内再让 Rectangle 去指它：
class Point {           // 这个 class 用来表述“点”
public:
    Point(int x, int y);
    ...
    void setX(int newVal);
    void setY(int newVal);
    ...
};

struct RectDatea {          // 这些“点”数据用来表现一个矩形
    Point ulhc;             // ulhc="upper left-hand corner"（左上角）
    Point lrhc;             // lrhc="lower right-hand corner"（右下角）
};

class Rectangle {
    ...
private:
    std::tr1::shared_ptr<RectData> pData;       // 关于 tr1::shared_ptr，
};                                              // 见条款 13

// Rectangle 的客户必须能够计算 Rectangle 的范围，
// 所以这个 class 提供 upperLeft 函数和 lowerRight 函数。
class Rectangle {
public:
    ...
    Point& upperLeft() const { return pData->ulhc; }
    Point& lowerRight() const { return pData->lrhc; }
    ...
};

// 这样的设计可通过编译，但却是错误的。
// 一方面 upperLeft 和 lowerRight 被声明为 const 成员函数，因为它们的
// 目的只是为了提供客户一个得知 Rectangle 相关坐标点的方法，而不是让客户
// 修改 Rectangle（见条款 3）。另一方面两个函数却都返回 references
// 指向 private 内部数据，调用者于是可通过这些 references 更改内部数据！例如：
Point coord1(0, 0);
Point coord2(100, 100);
const Rectangle rec(coord1, coord2);        // rec 是个 const 矩形，
                                            // 从 (0, 0) 到 (100, 100)
rec.upperLeft().setX(50);                   // 现在 rec 却变成
                                            // 从 (50, 0) 带 (100, 100)
// 这里请注意，upperLeft 的调用者能够使用被返回的 reference
// （指向 rec 内部的 Point 成员变量）来更改成员。但 rec 其实应该是不可变的（const）！





// 第一，成员变量的封装性最多只等于“返回其 reference”的函数的访问级别。
// 本例之中虽然 ulhc 和 lrhc 都被声明为 private，它们实际上却是 public，
// 因为 public 函数 upperLeft 和 lowerRight 传出了它们的 references。
// 第二，如果 const 成员函数传出一个 reference，后者所指数据与对象自身有关联，
// 而它又被存储于对象之外，那么这个函数的调用者可以修改那笔数据。
// 这正是 bitwise constness 的一个附带结果，见条款 3。

// 如果它们返回的是指针或迭代器，相同的情况还是发生，原因也相同。
// References、指针和迭代器统统都是所谓的 handles（号码牌，用来取得某个对象），
// 而返回一个“代表对象内部数据”的 handle，随之而来的便是“降低对象封装性”的风险。
// 同时，它也可能导致“虽然调用 const 成员函数却造成对象状态被更改”。





class Rectangle {
public:
    ...
    const Point& upperLeft() const { return pData->ulhc; }
    const Point& lowerRight() const { return pData->lrhc; }
    ...
};

// 有了这样的改变，客户可以读取矩形的 Points，但不能涂写它们。
// 这些函数只让渡读取权，涂写权仍然是被禁止的。





// 它可能导致 dangling handles（空悬的号码牌）：这种 handles 所指东西（的所属对象）
// 不复存在。这种“不复存在的对象”最常见的来源就是函数返回值。
class GUIObject { ... };
const Rectangle                             // 以 by value 方式返回一个矩形
    boundingBox(const GUIObject& obj);      // 条款 3 谈过为什么返回类型是 const

GUIObject* pgo;                 // 让 pgo 指向某个 GUIObject
...
const Point* pUpperLeft =       // 取得一个指针指向外框左上点
    &(boundingBox(*pgo).upperLeft());

// 对 boundingBox 的调用获得一个新的、暂时的 Rectangle 对象。
// 这个对象没有名称，所以我们权且称它为 temp。随后 upperLeft 作用于 temp 身上，
// 返回一个 reference 指向 temp 的一个内部成分，更具体地说是
// 指向一个用以标示 temp 的 Points。于是 pUpperLeft 指向那个
// Point 对象。目前为止一切还好，但故事尚未结束，因为在那个语
// 句之后，boundingBox 的返回值，也就是我们所说的 temp，将被
// 销毁，而那间接导致 temp 内的 Points 析构。最终导致 pUpperLeft
// 指向一个不再存在的对象；也就是说一旦产出 pUpperLeft 的那个
// 语句结束，pUpperLeft 也就变成悬空、虚吊（dangling）！
```

<br>

### 条款 29：为“异常安全”而努力是值得的

- 异常安全函数（Exception-safe functions）即使发生异常也不会泄漏资源或允许任何数据结构败坏。这样的函数区分为三种可能的保证：基本型、强烈型、不抛异常型。
- “强烈保证”往往能够以 copy-and-swap 实现出来，但“强烈保证”并非对所有函数都可实现或具备现实意义。
- 函数提供的“异常安全保证”通常最高只等于其所调用之各个函数的“异常安全保证”中的最弱者。

``` cpp {.line-numbers}
class PrettyMenu {
public:
    ...
    void changeBackground(std::istream& imgSrc);        // 改变背景图像
    ...
private:
    Mutex mutex;                    // 互斥器
    Image* bgImage;                 // 目前的背景图像
    int imageChanges;               // 背景图像被改变的次数
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    lock(&mutex);                   // 取得互斥器（见条款 14）
    delete bgImage;                 // 摆脱旧的背景图像
    ++imageChanges;                 // 修改图像变更次数
    bgImage = new Image(imgSrc);    // 安装新的背景图像
    unlock(&mutex);                 // 释放互斥器
}

// 从“异常安全性”的观点来看，这个函数很糟。
// “异常安全”有两个条件，而这个函数没有满足其中任何一个条件。





// 当异常被抛出时，带有异常安全性的函数会：

// （1）：不泄漏任何资源。上述代码没有做到这一点，因为一旦“new Image(imgSrc)”导致异常，
// 对 unlock 的调用就绝不会执行，于是互斥器就永远被把持住了。

// （2）：不允许数据败坏。如果“new Image(imgSrc)”抛出异常，
// bgImage 就是指向一个已被删除的对象，imageChanges 也已被累加，
// 而其实并没有新的图像被成功安装起来。（但从另一个角度说，旧图像已被消除，
// 所以你可能会争辩说图像还是“改变了”）。





// 解决资源泄漏的问题很容易，因为条款 13 讨论过如何以对象管理资源，
// 而条款 14 也导入了 Lock class 作为一种“确保互斥器被及时释放”的方法：
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock m1(&mutex);        // 来自条款 14：获得互斥器并确保它稍后被释放
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}





// 把资源泄漏抛诸脑后，现在我们可以专注解决数据的败坏了。

// 异常安全函数（Exception-safe functions）提供一下三个保证之一：

// （1）基本承诺：如果异常被抛出，程序内的任何事物仍然保持在有效状态下。
// 没有任何对象或数据结构会因此而败坏，所有对象都处于一种内部前后一致的状态
// （例如所有的 class 约束条件都继续获得满足）。
// 然而程序的现实状态（exact state）恐怕不可预测。
// 举个例子，我们可以撰写 changeBackground 使得一旦有异常被抛出时，
// PrettyMenu 对象可以继续拥有原背景图像，或是令它拥有某个缺省背景图像，
// 但客户无法预期哪一种情况。如果想知道，他们恐怕必须调用某个成员函数
// 以得知当时的背景图像是什么。

// （2）强烈保证：如果异常被抛出，程序状态不改变。
// 调用这样的函数需有这样的认知：如果函数成功，就是完全成功，
// 如果函数失败，程序会回复到“调用函数之前”的状态。
// 和这种提供强烈保证的函数公事，比刚才说的那种只是提供基本承诺的函数共事，容易多了，
// 因为在调用一个提供强烈保证的函数后，程序状态只有两种可能：
// 如预期般地到达函数成功执行后的函数，或回到函数被调用前的状态。
// 与此成对比的是，如果调用一个只提供基本承诺的函数，而真的出现异常，
// 程序有可能处于任何状态——只要那是个合法状态。

// （3）不抛掷（nothrow）保证：承诺绝不抛出异常，因为它们总是能够
// 完成它们原先承诺的功能。作用于内置类型（例如 ints，指针等等）
// 身上的所有操作都提供 nothrow 保证。这是异常安全码中一个必不可少的关键基础材料。
// 如果我们假设，函数带着“空白的异常明细”（empty exception specification）者
// 必为 nothrow 函数，似乎合情合理，其实不尽然。举个例子，考虑以下函数：
int doSomething() throw();      // 注意“空白的异常明细”
                                // （empty exception spec
// 这并不是说 doSomething 绝不会抛出异常，而是说如果 doSomething 抛出异常，
// 将是严重错误，会有你意想不到的函数被调用。





// 我们重新排列 changeBackground 内的语句次序，使得在更换图像
// 在之后才累加 imageChanges。一般而言这是个好策略：不要为了表
// 示某件事情发生而改变对象状态，除非那件事情真的发生了。
class PrettyMenu {
    ...
    std::tr1::shared_ptr<Image> bgImage;
    ...
};
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock m1(&mutex);
    bgImage.reset(new Image(imgSrc));       // 以“new Image”的执行结果
                                            // 设定 bgImage 内部指针
    ++imageChanges;
}

// 这两个改变几乎足够让 changeBackground 提供强烈保证的异常安全保证。
// 美中不足的是参数 imgSrc。如果 Image 构造函数抛出异常，有可能输入流
// （input stream）的读取记号（read marker）已被移走，而这样的搬移对
// 程序其余部分是一种可见状态的改变。所以 changeBackground 在解决这
// 个问题之前只提供基本的异常安全保证。





// 有个一般化的设计策略很典型地会导致强烈保证，很值得熟悉它。
// 这个策略被称为 copy and swap。原则很简单：为你打算修改的
// 对象（原件）做出一份副本，然后在那副本上做一切必要修改。
// 若有任何修改动作抛出异常，原对象仍保持为改变状态。待所有改
// 变都成功后，再将修改过的那个副本和原对象在一个不抛出异常的
// 操作中置换（swap）。

// 实现上通常是将所有“隶属对象的数据”从原对象放进另一个对象内，
// 然后赋予原对象一个指针，指向那个所谓的实现对象（implementation object，即副本）。
// 这种手法常被称为 pimpl idiom，条款 31 详细描述了它。
// 对 PrettyMenu 而言，典型手法如下：
struct PMImpl {                                 // PMImpl = "PrettyMenu Impl";
    std::tr1::shared_ptr<Image> bgImage;        // 稍后说明为什么它是个 struct
    int imageChanges;
};

class PrettyMenu {
    ...
private:
    Mutex mutex;
    std::tr1::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    using std::swap;                            // 见条款 25
    Lock m1(&mutex);                            // 获得 mutex 的副本数据
    std::tr1::shared_ptr<PMImpl>
        pNew(new PMImpl(*pImpl));
    pNew->bgImage.reset(new Image(imgSrc));     // 修改副本
    ++pNew->imageChanges;
    swap(pImpl, pNew);                          // 置换（swap）数据，释放 mutex
}

// 此例之中我选择让 PMImpl 成为一个 struct 而不是一个 class，
// 这是因为 PrettyMenu 的数据封装性已经由于“pImpl 是 private”
// 而获得了保证。如果令 PMImpl 为一个 class，虽然一样好，有时候
// 却不太方便（但也保持了面向对象纯度）。如果你要，也可以将 PMImpl
// 嵌套于 PrettyMenu 内，但打包问题（packaging，例如“独立撰写
// 异常安全码”）是我们这里所挂虑的事。





// “copy-and-swap”策略是对对象状态做出“全有或全无”改变的一个很好办法，
// 但一般而言它并不保证整个函数有强烈的异常安全性。
// 为了了解原因，让我们考虑 changeBackgroud 的一个抽象概念：someFunc。
// 它使用 copy-and-swap 策略，但函数内还包括对另外两个函数 f1 和 f2 的调用：
void someFunc()
{
    ...             // 对 local 状态做一份副本
    f1();
    f2();
    ...             // 将修改后的状态置换过来
}
// 很显然，如果 f1 或 f2 的异常安全性比“强烈保证”低，就很难让 someFunc
// 成为“强烈异常安全”。




// copy-and-swap 的关键在于“修改对象数据的副本，然后在一个不
// 抛异常的函数中将修改后的数据和原件置换”，因此必须为每一个即
// 将被改动的对象做出一个副本，那得耗用你可能无法（或无意愿）
// 供应的时间和空间。
```

<br>

### 条款 30：透彻了解 inlining 的里里外外

- 将大多数 inlining 限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级（binary upgradability）更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。
- 不要只因为 function templates 出现在头文件，就将它们声明为 inline。

``` cpp {.line-numbers}
// 记住，inline 只是对编译器的一个申请，不是强制命令。
// 这项申请可以隐喻提出，也可以明确提出。
// 隐喻方式是将函数定义于 class 定义式内：
class Person {
public:
    ...
    int age() const { return theAge; }      // 一个隐喻地 inline 申请
    ...                                     // age 被定义于 class 定义式内。
private:
    int theAge;
};

// 这样的函数通常是成员函数，但条款 46 说 friend 函数也可被定义于 class 内，
// 如果真是那样，它们也是被隐喻声明为 inline。





// 明确声明 inline 函数的做法则是在其定义式前加上关键字 inline。
// 例如标准的 max template（来自 <algorithm>）往往这样实现出来：
template<typename T>                                // 明确申请 inline：
inline const T& std::max(const T& a, const T& b)    // std::max 之前有
{ return a < b ? b : a; }                           // 关键字 "inline"

// 我们发现 inline 函数和 templates 两者通常都被定义于头文件内。
// 这使得某些程序员以为 function templates 一定必须是 inline。
// 这个结论不但无效而且可能有害，值得深入看一看。





// Template 的具现化与 inlining 无关。如果你正在写一个 template
// 而你认为所有根据此 template 具现出来的函数都应该 inlined，请将
// 此 template 声明为 inline；这就是上述 std::max 代码的作为。
// 但如果你写的 templates 没有理由要求它所具现的每一个函数都是
// inlined，就应该避免将这个 template 声明为 inline（不论显式或隐式）。
inline void f() { ... }     // 假设编译器有意愿 inline “对 f 的调用”
void (*pf)() = f;           // pf 指向 f
...
f();                        // 这个调用将被 inlined，因为它是一个正常调用。
pf();                       // 这个调用或许不被 inlined，因为它通过函数指针达成。





class Base {
public:
    ...
private:
    std::string bm1, bm2;           // base 成员 1 和 2
};

class Derived : public Base {
public:
    Derived() {}                    // Derived 构造函数是空的，哦，是吗？
    ...
private:
    std::string dm1, dm2, dm3;      // derived 成员 1 - 3
};

// 编译器为稍早说的那个表面上看起来为空的 Derived 构造函数所产
// 生的代码，相当于以下所列：
Derived::Derived()              // “空白 Derived 构造函数”的观念性实现
{
    Base::Base();                           // 初始化“Base 成分”

    try { dm1.std::string::string(); }      // 试图构造 dm1。
    catch (...) {                           // 如果抛出异常就
        Base::~Base();                      // 销毁 base class 成分，并
        throw;                              // 传播该异常。
    }

    try { dm2.std::string::string(); }      // 试图构造 dm1。
    catch (...) {                           // 如果抛出异常就
        dm1.std::string::~string();         // 销毁 dm1，
        Base::~Base();                      // 销毁 base class 成分，并
        throw;                              // 传播该异常。
    }

    try { dm3.std::string::string(); }      // 试图构造 dm1。
    catch (...) {                           // 如果抛出异常就
        dm2.std::string::~string();         // 销毁 dm2，
        dm1.std::string::~string();         // 销毁 dm1，
        Base::~Base();                      // 销毁 base class 成分，并
        throw;                              // 传播该异常。
    }
}
```

<br>

### 条款 31：将文件间的编译依存关系降至最低

- 支持“编译依存性最小化”的一般构想是：相依于声明式，不要相依于定义式。基于此构想的两个手段是 Handle classes 和 Interface classes。
- 程序库头文件应该以“完全且仅有声明式”（full and declaration-only forms）的形式存在。这种做法不论是否涉及 templates 都适用。

``` cpp {.line-numbers}
class Person {
public:
    Person(const std::string& name, const Date& birthday,
           const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string addres() const;
    ...
private:
    std::string theName;        // 实现细目
    Date theBirthDate;          // 实现细目
    Address theAddress;         // 实现细目
};

// 这里的 class Person 无法通过编译——如果编译器没有取得其实现代码所用到的
// class string，Date 和 Address 的定义式。这样的定义式通常由 #include
// 指示符提供，所以 Person 定义文件的最上方很可能存在这样的东西：
#include <string>
#include "date.h"
#include "address.h"
// 不幸的是，这么一来便是在 Person 定义文件和其含入文件之间形成了
// 一种编译依存关系（compilation dependency）。如果这些头文件中
// 有任何一个被改变，或这些头文件所依赖的其他头文件有任何改变，
// 那么每一个含入 Person class 的文件就得重新编译，任何使用
// Person class 的文件也必须重新编译。这样的连串编译依存关系
// （cascading compilation dependencies）会对许多项目造成难以
// 形容的灾难。

// 为什么 C++ 坚持将 class 的实现细目置于 class 定义式中？
// 为什么不这样定义 Person，将实现细目分开叙述？
namespace std {
    class string;           // 前置声明（不正确，详下）
}                           //
class Date;                 // 前置声明
class Address;              // 前置声明
class Person {
public:
    Person(const std::string& name, const Date& birthday,
           const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
};

// 第一，string 不是个 class，它是个 typedef（定义为 basic_string<char>）。
// 因此上述针对 string 而做的前置声明并不正确。

// 关于“前置声明每一件东西”的第二个（同时也是比较重要的）困难的是，
// 编译器必须在编译期间知道对象的大小。考虑这个：
int main()
{
    int x;                  // 定义一个 int
    Person p( params );     // 定义一个 Person
    ...
}





// 编译器将上述代码视同这样子：
int main()
{
    int x;              // 定义一个 int
    Person* p;          // 定义一个指针指向 Person 对象
    ...
}

// 针对 Person 我们可以这样做：
// 把 Person 分割为两个 classes，一个只提供接口，另一个负责实现该接口。
// 如果负责实现的那个所谓 implementation class 取名为 PersonImpl，
// Person 将定义如下：
#include <string>           // 标准程序库组件不该被前置声明。
#include <memory>           // 此乃为了 tr1::shared_ptr 而含入；详后。

class PersonImpl;           // Person 实现类的前置声明。
class Date;                 // Person 接口用到的 classes 的前置声明。
class Address;

class Person {
public:
    Person(const std::string& name, const Date& birthday,
           const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::tr1::shared_ptr<PersonImpl> pImpl;     // 指针，指向实现物；
                                                // std::tr1::shared_ptr 见条款 13.
};

// 在这里，main class（Person）只内含一个指针成员
// （这里使用 tr1::shared_ptr，见条款 13），
// 指向其实现类（PersonImpl）。这般设计常被称为 pimpl idiom
// （pimpl 是 “pointer to implementation” 的缩写）。

// 这样的设计之下，Person 的客户就完全与 Dates，Addresses 以及 Persons
// 的实现细目分离了。那些 classes 的任何实现修改都不需要 Person 客户端
// 重新编译。此外由于客户无法看到 Person 的实现细目，也就不可能
// 写出什么“取决于那些细目”的代码。这真正是“接口与实现分离”！





// 设计策略：

// 1、如果使用 object references 或 object pointers 可以完成任务，
// 就不要使用 objects。

// 2、如果能够，尽量以 class 声明式替换 class 定义式。
class Date;                         // class 声明式。
Date today();                       // 没问题 —— 这里并不需要
void clearAppointments(Date d);     // Date 的定义式。

// 3、为声明式和定义式提供不同的头文件。
#include "datefwd.h"                // 这个头文件声明（但为定义）class Date。
Date today();                       // 同前。
void clearAppointments(Date d);





// 像 Person 这样使用 pimpl idiom 的 classes，往往被称为 Handle classes。
// 办法之一是将它们的所有函数转交给相应的实现类（implementation classes）
// 并由后者完成实际工作。例如下面是 Person 两个成员函数的实现：
#include "Person.h"         // 我们正在实现 Person class，
                            // 所以必须 #include 其 class 定义式。
#include "PersonImpl.h"     // 我们也必须 #include PersonImpl 的
                            // class 定义式，否则无法调用其成员函数；
                            // 注意，PersonImpl 有着和 Person
                            // 完全相同的成员函数，两者接口完全相同。
Person::Person(const std::string& name, const Date& birthday,
               const Address& addr)
    : pImpl(new PersonImpl(name, birthday, addr))
{}

std::string Person::name() const
{
    return pImpl->name();
}

// 请注意，Person 构造函数以 new（见条款 16）调用 PersonImpl 构造函数，
// 以及 Person::name 函数内调用 PersonImpl::name。这是重要的，
// 让 Person 变成一个 Handle class 并不会改变它做的事，只会
// 改变它做事的方法。





// 另一个制作 Handle class 的办法是，令 Person 成为一种特殊的
// abstract base class（抽象基类），称为 Interface class。这种
// class 的目的是详细一一描述 derived classes 的接口（见条款 34），
// 因此它通常不带成员变量，也没有构造函数，只有一个 virtual 析构函数
// （见条款 7）以及一组 pure virtual 函数，用来叙述整个接口。
class Person {
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
    virtual std::string address() const = 0;
    ...
};

// 这个 class 的客户必须以 Person 的 pointers 和 references 来
// 撰写应用程序，因为它不可能针对“内含 pure virtual 函数”的 Person classes
// 具现出实体。（然而却有可能对派生自 Person 的 classes 具现出实体，详下。）
// 就像 Handle classes 的客户一样，除非 Interface class 的接口
// 被修改否则其客户不需重新编译。





class Person {
public:
    ...
    static std::tr1::shared_ptr<Person>     // 返回一个 tr1::shared_ptr，指向
        create(const std::string& name,     // 一个新的 Person，并以给定之参数
               const Date& birthday,        // 初始化。条款 18 告诉你
               const Address& addr);        // 为什么返回的是 tr1::shared_ptr
};

// 客户会这样使用它们：

std::string name;
Date dateOfBirth;
Address address;
...
// 创建一个对象，支持 Person 接口
std::tr1::shared_ptr<Person> pp(Person::create(name, dateOfBirth, address));

...
std::cout << pp->name()             // 通过 Person 的接口使用这个对象
          << " was born on "
          << pp->birthDate()
          << " and now lives at "
          << pp->address();
...                                 // 当 pp 离开作用域，
                                    // 对象会被自动删除，
                                    // 见条款 13。

// 假设 Interface class Person 有个具象的 derived class RealPerson，
// 后者提供继承而来的 virtual 函数的实现：
class RealPerson : public Person {
public:
    RealPerson(const std::string& name, const Date& birthday,
               const Address& addr)
        : theName(name), theBirthDate(birthday), theAddress(addr)
    {}

    virtual ~RealPerson() {  }
    std::string name() const;           // 这些函数的实现码并不显示于此。
    std::string birthDate() const;      // 但它们很容易想象。
    std::string address() const;

private:
    std::string theName;
    Date theBirthDate;
    Address theAddress;
};

// 有了 RealPerson 之后，写出 Person::create 就真的一点也不稀奇了：
std::tr1::shared_ptr<Person> Person::create(const std::string& name,
                                            const Date& birthday,
                                            const Address& addr)
{
    return std::shared_ptr<Person>(new RealPerson(name, birthday, addr));
}

// RealPerson 示范实现 Interface class 的两个最常见机制之一：
// 从 Interface class（Person）继承接口规格，然后实现出接口所
// 覆盖的函数。Interface class 的第二个实现法涉及多重继承，
// 那是条款 40 探索的主题。
```

<br>

## 6、继承与面向对象设计

<br>

### 条款 32：确定你的 public 继承塑模出 is-a 关系

- “public 继承”意味 is-a。适用于 base classes 身上的每一件事情一定也适用于 derived classes 身上，因为每一个 derived class 对象也都是一个 base class 对象。

``` cpp {.line-numbers}
// 以 C++ 进行面向对象编程，最重要的一个规则是：
// public inheritance（公开继承）意味“is-a”（是一种）的关系。

// 如果你令 class D（“Derived”）以 public 形式继承 class B（“Base”），
// 你便是告诉 C++ 编译器（以及你的代码读者）说，
// 每一个类型为 D 的对象同时也是一个类型为 B 的对象，反之不成立。

// 你主张“B 对象可派上用场的人任何地方，D 对象一样可以派上用场”
// （译注：此即所谓 Liskov Substitution Principle），
// 因为每一个 D 对象都是一种（是一个）B 对象。反之如果你需要一个 D 对象，
// B 对象无法效劳，因为虽然每个 D 对象都是一个 B 对象，反之并不成立。

class Person { ... };
class Student : public Person { ... };

// 人的概念比学生更一般化，学生是人的一种特殊形式。

// 在 C++ 领域中，任何函数如果期望获得一个类型为 Person（或 pointer-to-Person
// 或 reference-to-Person）的实参，都也愿意接受一个 Student 对象
// （或 pointer-to-Student 或 reference-to-Student）：

void eat(const Person& p);              // 任何人都会吃
void study(const Student& s);           // 只有学生才到校学习
Person p;                               // p 是人
Student s;                              // s 是学生
eat(p);                                 // 没问题，p 是人
eat(s);                                 // 没问题，s 是学生，而学生也是（is-a）人
study(s);                               // 没问题，s 是个学生
study(p);                               // 错误！p 不是个学生

// 这个论点只对 public 继承才成立。只有当 Student 以 public 形式继承 Person，
// C++ 的行为才会如我所描述。private 继承的意义与此完全不同（见条款 39），
// 至于 protected 继承，那是一种其意义至今仍然困惑我的东西。





// public 继承和 is-a 之间的等价关系听起来颇为简单，但有时候你的直觉
// 可能会误导你。举个例子，企鹅（penguin）是一种鸟，这是事实。鸟可以
// 飞，这也是事实。如果我们天真地以 C++ 描述这层关系，结果如下：
class Bird {
public:
    virtual void fly();             // 鸟可以飞
    ...
};

class Penguin : public Bird {       // 企鹅是一种鸟
    ...
};





class Bird {
    ...                             // 没有声明 fly 函数
};

class FlyingBird : public Bird {
public:
    virtual void fly();
    ...
};

class Penguin : public Bird {
    ...                             // 没有声明 fly 函数
};

// 即便如此，此刻我们仍然未能完全处理好这些鸟事，因为对某些软件系统而言，
// 可能不需要区分会飞的鸟和不会飞的鸟。所谓最佳设计，取决于系统希望做什么
// 事，包括现在与未来。





// 另有一种思想派别处理我所谓“所有的鸟都会飞，企鹅是鸟，但是企鹅不会飞，喔噢”的问题，
// 就是为企鹅重新定义 fly 函数，令它产生一个运行期错误：
void error(const std::string& msg);         // 定义于另外某处

class Penguin : public Bird {
public:
    virtual void fly() { error("Attempt to make a penguin fly!"); }
    ...
};

// 很重要的是，你必须认知这里所说的某些东西可能和你所想的不同。
// 这里并不是说“企鹅不会飞”，而是说“企鹅会飞，但尝试那么做事一种错误”。
// 如何描述其间的差异？从错误被侦测出来的时间点观之，“企鹅不会飞”
// 这一限制可由编译器强制实施，但若违反“企鹅尝试飞行，是一种错误”这一条规则，
// 只有运行期才能检测出来。





// 为了表现“企鹅不会飞，就这样”的限制，你不可以为 Penguin 定义 fly 函数：
class Bird {
    ...                             // 没有声明 fly 函数
};

class Penguin : public Bird {
    ...                             // 没有声明 fly 函数
};

// 现在，如果你试图让企鹅飞，编译器会对你的背信加以谴责：
Penguin p;
p.fly();                // 错误！

// 条款 18 说过：好的接口可以防止无效的代码通过编译，因此你应该
// 宁可采取“在编译期拒绝企鹅飞行”的设计，而不是“只在运行期才能
// 侦测它们”的设计。





// class Square 应该以 public 形式继承 class Rectangle 吗？
class Rectangle {
public:
    virtual void setHeight(int newHeight);
    virtual void setWidth(int newWidth);
    virtual int height() const;             // 返回当前值
    virtual int width() const;
    ...
};

void makeBigger(Rectangle& r)       // 这个函数用以增加 r 的面积
{
    int oldHeight = r.height();
    r.setWidth(r.width() + 10);         // 为 r 的宽度加 10
    assert(r.height() == oldHeight);    // 判断 r 的高度是否未曾改变
}
// 显然，上述的 assert 结果永远为真。
// 因为 makeBigger 只改变 r 的宽度；r 的高度从未被更改。

// 现在考虑这段代码，其中使用 public 继承，允许正方形被视为一种矩形：
class Square : public Rectangle { ... };
Square s;
...
assert(s.width() == s.height());        // 这对所有正方形一定为真。
makeBigger(s);                          // 由于继承，s 是一种（is-a）矩形，
                                        // 所以我们可以增加其面积。
assert(s.width() == s.height());        // 对所有正方形应该仍然为真。

// 这也很明显，第二个 assert 结果也应该永远为真。
// 因为根据定义，正方形的宽度和其高度相同。

// 但现在我们遇上了一个问题，我们如何调解下面各个 assert 判断式：
// （1）调用 makeBigger 之前，s 的高度和宽度相同；
// （2）在 makeBigger 函数内，s 的宽度改变，但高度不变。
// （3）makeBigger 返回之后，s 的高度再度和其宽度相同。
// （注意 s 是以 by reference 方式传给 makeBigger，所以
// makeBigger 修改的是 s 自身，不是 s 的副本。）





// is-a 并非是唯一存在于 classes 之间的关系。另两个常见的关系是
// has-a（有一个）和 is-implemented-in-terms-of（根据某物实现出）。
```

<br>

### 条款 33：避免遮掩继承而来的名称

- derived classes 内的名称会遮掩 base classes 内的名称。在 public 继承下从来没有人希望如此。
- 为了让被遮掩的名称再见天日，可使用 using 声明式或转交函数（forwarding functions）。

``` cpp {.line-numbers}
int x;                      // global 变量
void someFunc()
{
    double x;               // local 变量
    std::cin >> x;          // 读一个新值赋予 local 变量 x
}





class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf2();
    void mf3();
    ...
};

class Derived : public Base {
public:
    virtual void mf1();
    void mf4();
    ...
};

// Base 的作用域
// x（成员变量）
// mf1（1 个函数）
// mf2（1 个函数）
// mf3（1 个函数）

// Derived 的作用域（被 Base 作用域包含）
// mf1（1 个函数）
// mf4（1 个函数）





void Derived::mf4()
{
    ...
    mf2();
    ...
}

// 当编译器看到这里使用名称 mf2，必须估算它指涉（refer to）什么东西。
// 编译器的做法是查找各作用域，看看有没有某个名为 mf2 的声明式。
// 首先查找 local 作用域（也就是 mf4 覆盖的作用域），在那儿没找到
// 任何东西名为 mf2。于是查找其外围作用域，也就是 class Derived 覆盖的
// 作用域。还是没找到任何东西名为 mf2，于是再往外围移动，本例为
// base class。在那儿编译器找到一个名为 mf2 的东西了，于是停止查找。
// 如果 Base 内还是没有 mf2，查找动作便继续下去，首先找内含 Base 的
// 那个 namespace(s) 的作用域（如果有的话），最后往 global 作用域找去。





// 这次让我们重载 mf1 和 mf3，并且添加一个新版 mf3 到 Derived 去。
// 如条款 36 所说，这里发生的事情是：Derived 重载了 mf3，那是一个
// 继承而来的 non-virtual 函数。这会使整个设计立刻显得疑云重重，
// 但为了充分认识继承体系内的“名称可视性”，我们暂时安之若素。
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
    ...
};

class Derived : public Base {
public:
    virtual void mf1();
    void mf3();
    void mf4();
    ...
};

// Base 的作用域
// x（成员变零）
// mf1（2 个函数）
// mf2（1 个函数）
// mf3（2 个函数）

// Derived 的作用域（被 Base 作用域包含）
// mf1（1 个函数）
// mf3（1 个函数）
// mf4（1 个函数）

// 这段代码带来的行为会让每一位第一次面对它的 C++ 程序员大吃一惊。
// 以作用域为基础的“名称遮掩规则”并没有改变，因此 base class
// 内所有名为 mf1 和 mf3 的函数都被 derived class 内的 mf1 和
// mf3 函数遮掩掉了。从名称查找观点来看，Base::mf1 和 Base::mf3
// 不再被 Derived 继承！
Derived d;
int x;
...
d.mf1();            // 没问题，调用 Derived::mf1
d.mf1(x);           // 错误！因为 Derived::mf1 遮掩了 Base::mf1
d.mf2();            // 没问题，调用 Base::mf2
d.mf3();            // 没问题，调用 Derived::mf3
d.mf3(x);           // 错误！因为 Derived::mf3 遮掩了 Base::mf3

// 上述规则都适用，即使 base classes 和 derived classes 内的
// 函数有不同的参数类型也适用，而且不论函数是 virtual 或 non-virtual 一体适用。

// 这些行为背后的基本原理是为了防止你在程序库或应用框架（application framework）
// 内建立新的 derived class 时附带地从疏远的 base classes 继承重载函数。
// 不幸的是你通常会想继承重载函数。实际上如果你正在使用 public 继承而又不继
// 承那些重载函数，就是违反 base 和 derived classes 之间的 is-a 关系，
// 而条款 32 说过 is-a 是 public 继承的基石。因此你几乎总会想要推翻
// （override）C++ 对“继承而来的名称”的缺省遮掩行为。

// 你可以使用 using 声明式达成目标：
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
    ...
};

class Derived : public Base {
public:
    using Base::mf1;        // 让 Base class 内名为 mf1 和 mf3 的所有东西
    using Base::mf3;        // 在 Derived 作用域内都可见（并且 public）
    virtual void mf1();
    void mf3();
    void mf4();
    ...
};

// Base 的作用域
// x（成员变量）
// mf1（2 个函数）
// mf2（1 个函数）
// mf3（2 个函数）

// Derived 的作用域（被 Base 作用域包含）
// mf1（2 个函数）
// mf3（2 个函数）
// mf4（1 个函数）

// 现在，继承机制将一如往昔地运作：

Derived d;
int x;
...
d.mf1();        // 仍然没问题，仍然调用 Derived::mf1
d.mf1(x);       // 现在没问题了，调用 Base::mf1
d.mf2();        // 仍然没问题，仍然调用 Base::mf2
d.mf3();        // 没问题，调用 Derived::mf3
d.mf3(x);       // 现在没问题了，调用 Base::mf3

// 这意味如果你继承 base class 并加上重载函数，而你又希望重新定义
// 或覆写（推翻）其中一部分，那么你必须为那些原本会被遮掩的每个名
// 称引入一个 using 声明式，否则某些你希望继承的名称会被遮掩。





// 有时候你并不想继承 base classes 的所有函数，这是可以理解的。
// 然而在 private 继承之下（见条款 39）它却可能是有意义的。
// 例如假设 Derived 以 private 形式继承 Base，而 Derived 唯一
// 想继承的 mf1 是那个无参数版本。using 声明式在这里派不上用场，
// 因为 using 声明式会令继承而来的某给定名称之所有同名函数在
// derived class 中都可见。不，我们需要不同的技术，即一个简单的
// 转交函数（forwarding function）：
class Base {
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    ...                         // 与前同
};

class Derived : private Base {
public:
    virtual void mf1()          // 转交函数（forwarding function）；
    { Base::mf1(); }            // 暗自成为 inline（见条款 30）
    ...
};

...
Derived d;
int x;
d.mf1();                        // 很好，调用的是 Derived::mf1
d.mf1(x);                       // 错误！Base::mf1() 被遮掩了

// inline 转交函数（forwarding function）的另一个用途是为那些
// 不支持 using 声明式（注：这并非正确行为）的老旧编译器另辟一
// 条新路，将继承而得的名称汇入 derived class 作用域内。
```

<br>

### 条款 34：区分接口继承和实现继承

- 接口继承和实现继承不同。在 public 继承之下，derived classes 总是继承 base class 的接口。
- pure virtual 函数只具体指定接口继承。
- 简朴的（非纯）impure virtual 函数具体制定接口继承及缺省实现继承。
- non-virtual 函数具体指定接口继承以及强制性实现继承。

``` cpp {.line-numbers}
class Shape {
public:
    virtual void draw() const = 0;
    virtual void error(const std::string& msg);
    int objectID() const;
    ...
};

class Rectangle : public Shape { ... };
class Ellipse : public Shape { ... };

// Shape 是个抽象 class；它的 pure virtual 函数 draw 使它成为一个
// 抽象 class。所以客户不能够创建 Shape class 的实体，只能创建其
// derived classes 的实体。尽管如此，Shape 还是强烈影响了所有以 public 形式
// 继承它的 derived classes，因为：
// （1）成员函数的接口总是会被继承。

class Shape {
public:
    virtual void draw() const = 0;
    ...
};

// pure virtual 函数有两个最突出的特性：它们必须被任何“继承了它们”的
// 具象 class 重新声明，而且它们在抽象 class 中通常没有定义。把这两个
// 性质摆在一起，你就会明白：
// （2）声明一个 pure virtual 函数的目的是为了让 derived classes 只继承函数接口。
// 这对 Shape::draw 函数是再合理不过的事了，因为所有 Shape 对象都
// 应该是可绘出的，这是合理的要求。但 Shape class 无法为此函数提供
// 合理的缺省实现，毕竟椭圆形绘法迥异于绘法。Shape::draw 的声明式乃
// 是对具象 derived classes 设计者说，“你必须提供一个 draw 函数，
// 但我不干涉你怎么实现它。”

// 令人意外的是，我们竟然可以为 pure virtual 函数提供定义。也就是说
// 你可以为 Shape::draw 供应一份实现代码，C++ 并不会发出怨言，但调
// 用它的唯一途径是“调用时明确指出其 class 名称”：
Shape* ps = new Shape;              // 错误！Shape 是抽象的
Shape* ps1 = new Rectangle;         // 没问题
ps1->draw();                        // 调用 Rectangle::draw
Shape* ps2 = new Ellipse;           // 没问题
ps2->draw();                        // 调用 Ellipse::draw
ps1->Shape::draw();                 // 调用 Shape::draw
ps2->shape::draw();                 // 调用 Shape::draw

// 简朴的 impure virtual 函数背后的故事和 pure virtual 函数背后的故事
// 和 pure virtual 函数有点不同。一如往常，derived classes 继承其函数
// 接口，但 impure virtual 函数会提供一份实现代码，derived classes
// 可能覆写（override）它。稍加思索，你就会明白：
// （3）声明简朴的（非纯）impure virtual 函数的目的，是让 derived classes
// 继承该函数的接口和缺省实现。

class Shape {
public:
    virtual void error(const std::string& msg);
    ...
};

// Shape::error 的声明式告诉 derived classes 的设计者，“你必须支持
// 一个 error 函数，但如果你不想自己写一个，可以使用 Shape class 提供的缺省版本”。

// 但是允许 impure virtual 函数同时指定函数声明和函数缺省行为，却有可能造成危险。

class Airport { ... };          // 用以表现机场
class Airplane {
public:
    virtual void fly(const Airport& destination);
    ...
};

void Airplane::fly(const Airport& destination)
{
    缺省代码，将飞机飞至指定的目的地
}

class ModelA : public Airplane { ... };
class ModelB : public Airplane { ... };

// 为了表示所有飞机都一定能飞，并阐明“不同型飞机原则上需要不同的 fly 实现”，
// Airplane::fly 被声明为 virtual。然而为了避免在 ModelA 和 ModelB 中撰写
// 相同代码，缺省飞行行为由 Airplane::fly 提供，它同时被 ModelA 和 ModelB 继承。

// XYZ 公司的程序员在继承体系中针对 C 型飞机添加了一个 class，
// 但由于他们急着让新飞机上线服务，竟忘了重新定义其 fly 函数：
class ModelC: public Airplane {
    ...                             // 未声明 fly 函数
};

Airport PDX { ... };                // PDX 是我家附近的机场
Airplane* pa = new ModelC;
...
pa->fly(PDX);                       // 调用 Airplane::fly

// 这将酿成大灾难；这个程序试图以 ModalA 或 ModalB 的飞行方式来飞 ModelC。

// 问题不在 Airplane::fly 有缺省行为，而在于 ModelC 在未明白说出“我要”的情况
// 下就继承了该缺省行为。幸运的是我们可以轻易做到“提供缺省实现给 derived classes，
// 但除非它们明白要求否则免谈”。此间伎俩在于切断“virtual 函数接口” 和其“缺省实现”
// 之间的连接。下面是一种做法：
class Airplane {
public:
    virtual void fly(const Airport& destination) = 0;
    ...
protected:
    void defaultFly(const Airport& destination);
};

void Airplane::defaultFly(const Airport& destination)
{
    缺省行为，将飞机飞至指定的目的地。
}

// 请注意，Airplane::fly 已被改为一个 pure virtual 函数，只提供飞行接口。
// 其缺省行为也出现在 Airplane class 中，但此次系以独立函数 defaultFly 的
// 姿态出现。若想使用缺省实现（例如 ModelA 和 ModelB），可以在其 fly 函数中
// 对 defaultFly 做一个 inline 调用（但请注意条款 30 所言，inline 函数
// 和 virtual 函数之间的交互关系）：
class ModalA : public Airplane {
public:
    virtual void fly(const Airport& destination)
    { defaultFly(destination); }
    ...
};

class ModalB: public Airplane {
public:
    virtual void fly(const Airport& destination)
    { defaultFly(destination); }
    ...
};

// 现在 ModelC class 不可能意外继承不正确的 fly 实现代码了，
// 因为 Airplane 中的 pure virtual 函数迫使 ModelC 必须提供
// 自己的 fly 版本：
class ModelC : public Airplane {
public:
    virtual void fly(const Airport& destination);
    ...
};

void ModelC::fly(const Airport& destination)
{
    将 C 型飞机飞至指定的目的地
}





// 有些人反对以不同的函数分别提供接口和缺省实现，像上述的 fly 和 defaultFly 那样。
// 他们关心因过度雷同的函数名称而引起的 class 命名空间污染问题。
// 但是他们也同意，接口和缺省实现应该分开。
// 我们可以利用“pure virtual 函数必须在 derived classes 中重新声明，
// 但它们也可以拥有自己的实现”这一事实。
// 下面便是 Airplane 继承体系如何给 pure virtual 函数一份定义：
class Airplane {
public:
    virtual void fly(const Airport& destination) = 0;
    ...
};

void Airplane::fly(const Airport& destination)  // pure virtual 函数实现
{
    缺省行为，将飞机飞至指定的目的地
}

class ModelA : public Airplane {
public:
    virtual void fly(const Airport& destination)
    { Airplane::fly(destination); }
    ...
};

class ModelB : public Airplane {
public:
    virtual vodi fly(const Airport& destination)
    { Airplane::fly(destination); }
    ...
};

class ModelC : public Airplane {
public:
    virtual void fly(const Airport& destination);
    ...
};

void ModelC::fly(const Airport& destination)
{
    将 C 型飞机飞至指定的目的地
}

// 这几乎和前一个设计一模一样，只不过 pure virtual 函数 Airplane::fly 替换了
// 独立函数 Airplane::defaultFly。本质上，现在的 fly 被分割为两个基本要素：
// 其声明部分表现的是接口（那是 derived classes 必须使用的），其定义部分则
// 表现出缺省行为（那是 derived classes 可能使用的，但只有在它们明确提出申请
// 时才是）。如果合并 fly 和 defaultFly，就丧失了“让两个函数享有不同保护级别”
// 的机会：习惯上被设为 protected 的函数（defaultFly）如今成了 public
// （因为它在 fly 之中）。





class Shape {
public:
    int objectID() const;
    ...
};

// 如果成员函数是个 non-virtual 函数，意味是它并不打算在
// derived classes 中有不同的行为。实际上一个 non-virtual 成员函数
// 所表现的不变性（invariant）凌驾其特异性（specialization），
// 因为它表示不论 derived class 变得多么特异化，它的行为都不可以改变。
// 就其自身而言：
// （3）声明 non-virtual 函数的目的是为了令 derived classes 继承函数的接口
// 及一份强制性实现。

// 你可以把 Shape::objectID 的声明想做事：“每个 Shape 对象都有一个用来产生
// 对象识别码的函数；此识别码总是采用相同计算方法，该方法由 Shape::objectID
// 的定义式决定，任何 derived class 都不应该尝试改变其行为”。
// 由于 non-virtual 函数代表的意义是不变性（invariant）凌驾特异性（specialization），
// 所以它绝不该在 derived class 中被重新定义。





// pure virtual 函数、simple（impure）virtual 函数、non-virtual 函数
// 之间的差异，使你得以精确指定你想要 derived classes 继承的东西：
// 只继承接口，或是继承接口和一份缺省实现，或是继承接口和一份强制实现。
// 由于这些不同类型的声明意味根本意义并不相同的事情，当你声明你的成员函数时，
// 必须谨慎选择。如果你确实履行，应该能够避免经验不足的 class 设计者最常犯
// 的两个错误。

// 第一个错误是将所有函数声明为 non-virtual。这使得 derived classes 没有
// 余裕空间进行特化工作。non-virtual 析构函数尤其会带来问题。
// 当然啦，设计一个并不想成为 base class 的 class 是绝对合理的，既然这样，
// 将其所有成员函数都声明为 non-virtual 也很适当。

// 另一个常见错误是将所有成员函数声明为 virtual。有时候这样做是正确的，
// 例如条款 31 的 Interface classes。然而这也可能是 class 设计者缺乏坚定立场的前兆。
// 某些函数就是不该在 derived class中被重新定义，
// 果真如此你应该将那些函数声明为 non-virtual。
```

<br>

### 条款 35：考虑 virtual 函数以外的其他选择

- virtual 函数的替代方案包括 NVI 手法及 Strategy 设计模式的多种形式。NVI 手法自身是一个特殊形式的 Template Method 设计模式。
- 将机能从成员函数移到 class 外部函数，带来的一个缺点是，非成员函数无法访问 class 的 non-public 成员。
- tr1::function 对象的行为就像一般函数指针。这样的对象可接纳“与给定之目标签名式（target signature）兼容”的所有可调用物（callable entities）。

``` cpp {.line-numbers}
class GameCharacter {
public:
    virtual int healthValue() const;        // 返回人物的健康指数；
    ...                                     // derived classes 可重新定义它。
};

// healthValue 并未被声明为 pure virtual，这暗示我们将会有个计算
// 健康指数的缺省算法（见条款 34）。

// 这的确是再明白不过的设计，但是从某个角度说却反而成了它的弱点。由于
// 这个设计如此明显，你可能因此没有认真考虑其他替代方案。为了帮助
// 你跳脱面向对象设计路上的常轨，让我们考虑其他一些解法：





// 藉由 Non-virtual Interface 手法实现 Template Method 模式

class GameCharacter {
public:
    int healthValue() const             // derived classes 不重新定义它，
    {                                   // 见条款 36。
        ...                             // 做一些事情工作，详下。
        int retVal = doHealthValue();   // 做真正的工作。
        ...                             // 做一些事后工作，详下。
        return retVal;
    }
    ...
private:
    virtual int doHealthValue() const   // derived classes 可重新定义它。
    {
        ...                             // 缺省算法，计算健康指数。
    }
};

// 在这段（以及本条款其余的）代码中，我直接在 class 定义式内呈现成员函数本体。
// 一如条款 30 所言，那也就让它们全都暗自成了 inline。但其实我以这种方式呈现
// 代码只是为了让你比较容易阅读。我所描述的设计与 inlining 其实并没有关联，
// 所以请不要认为成员函数在这里被定义于 classes 内有特殊用意。不，它没有。





// 这一基本设计，也就是“令客户通过 public non-virtual 成员函数间接调用
// private virtual 函数”，称为 non-virtual interface （NVI）手法。
// 它是所谓 Template Method 设计模式（与 C++ templates 并无关联）的一个
// 独特表现形式。我把这个 non-virtual 函数（healthValue）称为 virtual
// 函数的外覆器（wrapper）。

// NVI 手法的一个优点隐身在上述代码注释“做一些事前工作”和“做一些事后工作”之中。
// 那些注释用来告诉你当时的代码保证在“virtual 函数进行真正的工作之前和之后”被调用。
// 这意味外覆器（wrapper）确保得以在一个 virtual 函数被调用之前设定好适当场景，
// 并在调用结束之后清理场景。“事前工作”可以包括锁定互斥器（locking a mutex）、
// 制造运转日志记录项（log entry）、验证 class 约束条件、验证函数先决条件等等。
// “事后工作”可以包括互斥器解除锁定（unlocking a mutex）、验证函数的事后条件、
// 再次验证 class 约束条件等等。如果你让客户直接调用 virtual 函数，就没有任何
// 好办法可以做这些事。

// 有件事实或许妨碍你跃跃欲试的心：NVI 手法涉及在 derived classes 内重新定义
// private virtual 函数。啊，重新定义若干个 derived classes 并不调用的函数！
// 这里并不存在矛盾。“重新定义 virtual 函数”表示某些事“如何”被完成，
//  “调用 virtual 函数”则表示它“何时”被完成。这些事情都是各自独立互不相干的。
// NVI 手法允许 derived classes 重新定义 virtual 函数，从而赋予它们“如何实现机能”
// 的控制能力，但 base class 保留诉说“函数何时被调用”的权利。一开始这些听起来似乎诡异，
// 但 C++ 的这种“derived classes 可重新定义继承而来的 private virtual 函数”
// 的规则完全合情合理。

// 在 NVI 手法下其实没有必要让 virtual 函数一定得是 private。某些 class 继承
// 体系要求 derived class 在 virtual 函数的实现内必须调用其 base class 的对应兄弟，
// 而为了让这样的调用合法，virtual 函数必须是 protected，不能是 private。
// 有时候 virtual 函数甚至一定得是 public（例如具备多态性质的 base classes 的
// 析构函数——见条款 7），这么一来就不能实施 NVI 手法了。





// 藉由 Function Pointers 实现 Strategy 模式

class GameCharacter;            // 前置声明（forward declaration）
// 以下函数是计算健康指数的缺省算法。
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter {
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
        : healthFunc(hcf)
    {}
    int healthValue() const
    { return healthFunc(*this); }
    ...
private:
    HealthCalcFunc healthFunc;
};

// 这个做法是常见的 Strategy 设计模式的简单应用。拿它和“植基于 GameCharacter 继承
// 体系内之 virtual 函数”的做法比较，它提供了某些有趣弹性：
// （1）同一人物类型之不同实体可以有不同的健康计算函数。例如：
class EvilBadGuy : public GameCharacter {
public:
    explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc)
        : GameCharacter(hcf)
    { ... }
    ...
};

int loseHealthQuickly(const GameCharacter&);    // 健康指数计算函数 1
int loseHealthSlowly(const GameCharacter&);     // 健康指数计算函数 2

EvilBadGuy ebg1(loseHealthQuickly);             // 相同类型的人物搭配
EvilBadGuy ebg2(loseHealthSlowly);              // 不同的健康计算方式

// （2）某已知人物之健康指数计算函数可在运行期变更。例如 GameCharacter 可提供
// 一个成员函数 setHealthCalculator，用来替换当前的健康指数计算函数。

// 换句话说，“健康指数计算函数不再是 GameCharacter 继承体系内的成员函数”这一事实意味，
// 这些计算函数并未特别访问“即将被计算健康指数”的那个对象的内部成分。
// 例如 defaultHealthCalc 并未访问 EvilBadGuy 的 non-public 成分。

// 一般而言，唯一能够解决“需要以 non-member 函数访问 class 的 non-public 成分”的
// 办法就是：弱化 class 的封装。例如 class 可声明那个 non-member 函数为 friends，
// 或是为其实现的某一部分提供 public 访问函数（其他部分则宁可隐藏起来）。
// 运用函数指针替换 virtual 函数，其优点（像是“每个对象可各自拥有自己的健
// 康计算函数”和“可在运行期改变计算函数”）是否足以弥补缺点（例如可能必须降
// 低 GameCharacter 封装性），是你必须根据每个设计情况的不同而抉择的。





// 藉由 tr1::function 完成 Strategy 模式

// 一旦习惯了 templates 以及它们对隐式接口（见条款 41）的使用，基于函数指针的做法
// 看起来便过分苛刻而死板了。为什么要求“健康指数之计算”必须是个函数，
// 而不能是某种“像函数的东西”（例如函数对象）呢？如果一定得是函数，为什么不能是
// 个成员函数呢？为什么一定得返回 int 而不是任何可被转换为 int 的类型呢？

// 如果我们不再使用函数指针（如前例的 healthFunc），而是改用一个类型为 tr1::function
// 的对象，这些约束就全都挥发不见了。就像条款 54 所说的，这样的对象可持有
// （保存）任何可调用（callable entity，也就是函数指针、函数对象、或成员函数指针），
// 只要其签名方式兼容于需求端。以下将刚才的设计改为使用 tr1::function：

class GameCharacter;                                // 如前
int defaultHealthCalc(const GameCharacter& gc);     // 如前
class GameCharacter {
public:
    // HealthCalcFunc 可以是任何“可调用物”（callable entity），可被调用并接受
    // 任何兼容于 GameCharacter 之物，返回任何兼容于 int 的东西。详下。
    typedef std::tr1::function<int (const GameCharacter&)> HealthCalcFunc;
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
        : healthFunc(hcf)
    {}
    int healthValue() const
    { return healthFunc(*this); }
    ...
private:
    HealthCalcFunc healthFunc;
};

// 这个改变如此细小，我总说它没有什么外显影响，除非客户在“指定健康计算函数”
// 这件事上需要更惊人的弹性：
short calcHealth(const GameCharacter&);     // 健康计算函数：
                                            // 注意其返回类型为 non-int
struct HealthCalculator {
    int operator() (const GameCharacter&) const
    { ... }
};

class GameLevel {
public:
    float health(const GameCharacter&) const;   // 成员函数，用以计算健康：
    ...                                         // 注意其 non-int 返回类型
};

class EvilBadGuy : public GameCharacter {       // 同前
    ...
};

class EyeCandyCharacter : public GameCharacter {    // 另一个人物类型：
    ...                                             // 假设其构造函数与
};                                                  // EvilBadGuy 同

EvilBadGuy ebg1(calcHealth);                    // 人物 1，使用某个
                                                // 函数计算健康指数

EyeCandyCharacter ecc1(HealthCalculator());     // 人物 2，使用某个
                                                // 函数对象计算健康指数
GameLevel currentLevel;
...
EvilBadGuy ebg2(                                // 人物 3 使用某个
    std::tr1::bind(&GameLevel::health,          // 成员函数计算健康指数
                   currentLevel,
                   _1)                          // 详见以下
);
// tr1::bind 的作为：它指出 ebg2 的健康计算函数应该总是以
// currentLevel 作为 GameLevel 对象。




// 古典的 Strategy 模式

// 如果你对设计模式（design patterns）比对 C++ 的酷劲更有兴趣，我告诉你，
// 传统（典型）的 Strategy 做法会将健康计算函数做成一个分离的继承体系中的
// virtual 成员函数。

class GameCharacter;        // 前置声明（forward declaration）
class HealthCalcFunc {
public:
    ...
    virtual int calc(const GameCharacter& gc) const
    { ... }
    ...
};

HealthCalcFunc defaultHealthCalc;

class GameCharacter {
public:
    explicit GameCharacter(HealthCalcFunc* phcf = &defaultHealthCalc)
        : pHealthCalc(phcf);
    {}
    int healthValue() const
    { return pHealthCalc->calc(*this); }
    ...
private:
    HealthCalcFunc* pHealthCalc;
};

// 这个解法的吸引力在于，熟悉标准 Strategy 模式的人很容易辨认它，
// 而且它还提供“将一个既有的健康算法纳入使用”的可能性——只要为 HealthCalcFunc
// 继承体系添加一个 derived class 即可。





// 本条款的根本忠告是，当你为解决问题而寻找某个设计方法时，
// 不妨考虑 virtual 函数的替代方案。

// （1）使用 non-virtual interface（NVI）手法，那是 Template Method 设计模式
// 的一种特殊形式。它以 public non-virtual 成员函数包裹较低访问性（private 或
// protected）的 virtual 函数。

// （2）将 virtual 函数替换为 “函数指针成员变量”，这是 Strategy 设计模式
// 的一种分解表现形式。

// （3）以 tr1::function 成员变量替换 virtual 函数，因为允许使用任何可调用物
// （callable entity）搭配一个兼容于需求的签名式。这也是 Strategy 设计模式的某种形式。

// （4）将继承体系内的 virtual 函数替换为另一个继承体系内的 virtual 函数。
// 这是 Strategy 设计模式的传统实现手法。

// 以上并未彻底而详尽地列出 virtual 函数的所有替换方案，
// 但应该足够让你知道的确有不少替换方案。
```

<br>

### 条款 36：绝不重新定义继承而来的 non-virtual 函数

- 绝对不要重新定义继承而来的 non-virtual 函数。

``` cpp {.line-numbers}
class B {
public:
    void mf();
    ...
};

class D : public B { ... };

D x;                    // x 是一个类型为 D 的对象

B* pB = &x;             // 获得一个指针指向 x
pB->mf();               // 经由该指针调用 mf

D* pD = &x;             // 获得一个指针指向 x
pD->mf();               // 经由该指针调用 mf

// 你可能会相当惊讶。毕竟两者都通过对象 x 调用成员函数 mf。由于两者所调用
// 的函数都相同，凭借的对象也相同，所以行为也应该相同，是吗？
// 是的，理应如此，但事实可能不是如此。更确切地说，如果 mf 是个 non-virtual
// 函数而 D 定义有自己的 mf 版本，那就不是如此：

class D : public B {
public:
    void mf();          // 遮掩（hides）了 B::mf；见条款 33
    ...
};
pB->mf();               // 调用 B::mf
pD->mf();               // 调用 D::mf

// 造成这一两面行为的原因是，non-virtual 函数如 B::mf 和 D::mf 都是静态绑定
// （statically bound，见条款 37）。这意思是，由于 pB 被声明为一个 pointers-to-B，
// 通过 pB 调用的 non-virtual 函数永远是 B 所定义的版本，即使 pB 指向一个类型为
// “B 派生之 class”的对象，一如本例。





// 但另一方面，virtual 函数却是动态绑定（dynamically bound，见条款 37），
// 所以它们不受这个问题之苦。如果 mf 是个 virtual 函数，不论是通过 pB 或 pD
// 调用 mf，都会导致调用 D::mf，因为 pB 和 pD 真正指的都是一个类型为 D 的对象。

// 如果你正在编写 class D 并重新定义继承自 class B 的 non-virtual 函数 mf，
// D 对象很可能展现出精神分裂的不一致行径。更明确地说，当 mf 被调用，任何一个 D 对象
// 都可能表现出 B 或 D 的行为；决定因素不在对象自身，而在于“指向该对象之指针”
// 当初的声明类型。References 也会展现出和指针一样难以理解的行径。





// 条款 32 已经说过，所谓 public 继承意味 is-a（是一种）的关系。条款 34 则描述
// 为什么在 class 内声明一个 non-virtual 函数会为该 class 建立起一个不变性
// （invariant），凌驾其特异性（specialization）。如果你将这两个观点施行于两个
// classes B 和 D 以及 non-virtual 成员函数 B::mf 身上，那么：
// 1、适用于 B 对象的每一件事，也适用于 D 对象，因为每个 D 对象都是一个 B 对象；
// 2、B 的 derived classes 一定会继承 mf 的接口和实现，因为 mf 是 B 的一个
// non-virtual 函数。
```

<br>

### 条款 37：绝不重新定义继承而来的缺省参数值

- 绝对不要重新定义一个继承而来的缺省参数值，因为缺省参数值都是静态绑定，而 virtual 函数——你唯一应该覆写的东西——却是动态绑定。

``` cpp {.line-numbers}
// virtual 函数系动态绑定（dynamically bound），
// 而缺省参数值却是静态绑定（statically bound）。





// 对象的所谓静态类型（static type），就是它在程序中被声明时所采用的类型。
// 考虑以下的 class 继承体系：

// 一个用以描述几何形状的 class
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    // 所有形状都必须提供一个函数，用来绘出自己
    virtual void draw(ShapeColor color = Red) const = 0;
    ...
};

class Rectangle : public Shape {
public:
    // 注意，赋予不同的缺省参数值。这真糟糕！
    virtual void draw(ShapeColor color = Green) const;
    ...
};

class Circle : public Shape {
public:
    virtual void draw(ShapeColor color) const;
    // 译注：请注意，以上这么写则当客户以对象调用此函数，一定要指定参数值。
    //       因为静态绑定下这个函数并不从其 base 继承缺省参数值。
    //       但若以指针（或 reference）调用此函数，可以不指定参数值。
    //       因为动态绑定下这个函数会从其 base 继承缺省参数值。
};

// 现在考虑这些指针：
Shape* ps;                      // 静态类型为 Shape*
Shape* pc = new Circle;         // 静态类型为 Shape*
Shape* pr = new Rectangle;      // 静态类型为 Shape*

// 本例中 ps，pc 和 pr 都被声明为 pointer-to-Shape 类型，所以它们都以它为静态类型。
// 注意，不论它们真正指什么，它们的静态类型都是 Shape*。

// 对象的所谓动态类型（dynamic type）则是指“目前所指对象的类型”。也就是说，
// 动态类型可以表现出一个对象将会有什么行为。以上例而言，pc 的动态类型是 Circle*，
// pr 的动态类型是 Rectangle*。ps 没有动态类型，因为它尚未指向任何对象。

// 动态类型一如其名称所示，可在程序执行过程中改变（通常是经由赋值动作）：
ps = pc;                // ps 的动态类型如今是 Circle*
ps = pr;                // ps 的动态类型如今是 Rectangle*

// Virtual 函数系动态绑定而来，意思是调用一个 virtual 函数时，究竟调用哪一份
// 函数实现代码，取决于发出调用的那个对象的动态类型：
pc->draw(Shape::Red);       // 调用 Circle::draw(Shape::Red)
pr->draw(Shape::Red);       // 调用 Rectangle::draw(Shape::Red)

//virtual 函数时动态绑定，而缺省参数值却是静态绑定。意思是你可能会在
// “调用一个定义于 derived class 内的 virtual 函数”的同时，却使用
// base class 为它所指定的缺省参数值：
pr->draw();                 // 调用 Rectangle::draw(Shape::Red)！

// 此例之中，pr 的动态类型是 Rectangle*，所以调用的是 Rectangle 的 virtual 函数，
// 一如你所预期。Rectangle::draw 函数的缺省参数值应该是 GREEN，
// 但由于 pr 的静态类型是 Shape*，所以此一调用的缺省参数值来自 Shape class
// 而非 Rectangle class！结局是这个函数调用有着奇怪并且几乎绝对没人预料得到
// 的组合，由 Shape class 和 Rectangle class 的 draw 声明式各出一半力。

// 以上事实不只局限于“ps，pc 和 pr 都是指针”的情况；即使把指针换成 references
// 问题仍然存在。重点在于 draw 是个 virtual 函数，而它有个缺省参数值在 derived class
// 中被重新定义了。





// 这一切都很好，但如果你试着遵守这条规则，并且同时提供缺省参数值给 base
// 和 derived classes 的用户，又会发生什么事呢？
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    virtual void draw(ShapeColor color = Red) const = 0;
    ...
};

class Rectangle : public Shape {
public:
    virtual void draw(ShapeColor color = Red) const;
    ...
};

// 喔噢，代码重复。更糟的是，代码重复又带着相依性（with dependencies）：
// 如果 Shape 内的缺省参数值改变了，所有“重复给定缺省参数值”的那些 derived
// classes 也必须改变，否则它们最终会导致“重复定义一个继承而来的缺省参数值”。
// 怎么办？

// 聪明的做法是考虑替代设计。条款 35 列了不少 virtual 函数的替代设计，
// 其中之一是 NVI（non-virtual interface）手法：令 base class 内的
// 一个 public non-virtual 函数调用 private virtual 函数，
// 后者可被 derived classes 重新定义。这里我们可以让 non-virtual 函数指定
// 缺省参数，而 private virtual 函数负责真正的工作：
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    void draw(ShapeColor color = Red) const             // 如今它是 non-virtual
    {
        doDraw(color);                                  // 调用一个 virtual
    }
    ...
private:
    virtual void doDraw(ShapeColor color) const = 0;    // 真正的工作
};                                                      // 在此处完成

class Rectangle : public Shape {
public:
    ...
private:
    virtual void doDraw(ShapeColor color) const;        // 注意，不须指定
    ...                                                 // 缺省参数值。
};

// 由于 non-virtual 函数应该绝对不被 derived classes 覆写（见条款 36），
// 这个设计很清楚地使得 draw 函数的 color 缺省参数值总是为 Red。
```

<br>

### 条款 38：通过复合塑模出 has-a 或“根据某物实现出”

- 复合（composition）的意义和 public 继承完全不同。
- 在应用域（application domain），复合意味 has-a（有一个）。在实现域（implementation domain），复合意味 is-implemented-in-terms-of（根据某物实现出）。

``` cpp {.line-numbers}
// 复合（composition）是类型之间的一种关系，当某种类型的对象内含它种类型的对象，
// 便是这种关系。例如：
class Address { ... };
class PhoneNumber { ... };

class Person {
public:
    ...
private:
    std::string name;               // 合成成分物（composed object）
    Address address;                // 同上
    PhoneNumber voiceNumber;        // 同上
    PhoneNumber faxNumber;          // 同上
};

// 在程序员之间复合（composition）这个术语有许多同义词，包括 layering（分层），
// containment（内含），aggregation（聚合）和 embedding（内嵌）。

// 条款 32 曾说，“public 继承”带有 is-a（是一种）的意义。复合也有它自己的意义。
// 实际上它有两个意义。复合意味 has-a（有一个）
// 或 is-implemented-in-terms-of（根据某物实现出）。
// 那是因为你正打算在你的软件中处理两个不同的领域（domains）。
// 程序中的对象其实相当于你所塑造的世界中的某些事物，例如人、汽车、一张张视频画面等等。
// 这样的对象属于应用域（application domain）部分。
// 其他对象则纯粹是实现细节上的人工制品，像是缓冲区（buffers）、互斥器（mutexes）、
// 查找树（search trees）等等。这些对象相当于你的软件的实现域（implementation domain）。
// 当复合发生于应用域内的对象之间，表现出 has-a 的关系：
// 当它发生于实现域内则是表现 is-implemented-in-terms-of 的关系。

// 上述的 Person class 示范 has-a 关系。





template<typename T>            // 将 list 应用于 Set。错误做法。
class Set : public std::list<T> { ... };

// 一如条款 32 所说，如果 D 是一种对象，对 B 为真的每一件事情对 D 也都应该为真。
// 但 list 可以内含重复元素，如果数值 3051 被安插到 list<int> 两次，
// 那个 list 将内含两笔 3051。Set 不可以内含重复元素，
// 如果数值 3051 被安插到 Set<int> 两次，这个 Set 只内含一笔 3051。
// 因此“Set 是一种 list”并不为真，因为对 list 对象为真的某些事情对 Set 对象并不为真。





// 由于这两个 classes 之间并非 is-a 的关系，所以 public 继承不适合用来塑模它们。
// 正确的做法是，你应当了解，Set 对象可根据一个 list 对象实现出来：
template<class T>                       // 将 list 应用于 Set。正确做法。
class Set {
public:
    bool member(const T& item) const;
    void insert(const T& item);
    void remove(const T& item);
    std::size_t size() const;
private:
    std::list<T> rep;                   // 用来表述 Set 的数据
};

// Set 成员函数大量倚赖 list 及标准程序库其他部分提供的技能来完成，
// 所以其实现很直观也很简单，只要你熟悉以 STL 编写程序：
template<typename T>
bool Set<T>::member(const T& item) const
{
    return std::find(rep.begin(), rep.end(), item) != rep.end();
}

template<typename T>
void Set<T>::insert(const T& item)
{
    if (!member(item)) rep.push_back(item);
}

template<typename T>
void Set<T>::remove(const T& item)
{
    typename std::list<T>::iterator it =            // 见条款 42 对
        std::find(rep.begin(), rep.end(), item);    // “typename”的讨论
    if (it != rep.end()) rep.erase(it);
}

template<typename T>
std::size_t Set<T>::size() const
{
    return rep.size();
}

// 这些函数如此简单，因此都适合成为 inlining 候选人。
// 但请记住，在做出任何与 inlining 有关的决定之前，应该先看看条款 30。

// Set 接口也不该造成“对 Set 而言无可置辩的权利”黯然失色，那个权利是指
// 它和 list 间的关系。这关系并非 is-a（虽然最初似乎是），而是
// is-implemented-in-terms-of。
```

<br>

### 条款 39：明智而审慎地使用 private 继承

- Private 继承意味 is-implemented-in-terms of（根据某物实现出）。它通常比复合（composition）的级别低。但是当 derived class 需要访问 protected base class 的成员，或需要重新定义继承而来的 virtual 函数时，这么设计是合理的。
- 和复合（composition）不同，private 继承可以造成 empty base 最优化。这对致力于“对象尺寸最小化”的程序库开发者而言，可能很重要。

``` cpp {.line-numbers}
// 现在我再重复该例的一部分，并以 private 继承替换 public 继承：
class Person { ... };
class Student : private Person { ... };         // 这次改用 private 继承
void eat(const Person& p);                      // 任何人都会吃
void study(const Student& s);                   // 只有学生才在学校学习

Person p;               // p 是人
Student s;              // s 是学生

eat(p);                 // 没问题，p 是人，会吃。
eat(s);                 // 没错！吓，难道学生不是人？！

// 显然 private 继承并不意味 is-a 关系。那么它意味什么？

// 到底 private 继承的行为如何呢？唔，统御 private 继承的首要规则你刚才已经见过了：
// 如果 classes 之间的继承关系是 private，编译器不会自动将一个 derived class 对象
// （例如 Student）转换为一个 base class 对象（例如 Person）。这和 public
// 继承的情况不同。这也就是为什么通过 s 调用 eat 会失败的原因。第二条规则是，
// 由 private base class 继承而来的所有成员，在 derived class 中都会
// 变成 private 属性，纵使它们在 base class 中原本是 protected 或 public 属性。

// Private 继承意味 implemented-in-terms-of（根据某物实现出）。
// 如果你让 class D 以 private 形式继承 class B，你的用意是为了采用 class B 
// 内已经备妥的某些特性，不是因为 B 对象和 D 对象存在有任何观念上的关系。
// private 继承纯粹只是一种实现技术（这就是为什么继承自一个 private base class
// 的每样东西在你的 class 内都是 private：因为它们都只是实现枝节而已）。
// 借用条款 34 提出的术语，private 继承意味只有实现部分被继承，接口部分应略去。
// 如果 D 以 private 形式继承 B，意思是 D 对象根据 B 对象实现而得，再没有其他意涵了。
// Private 继承在软件“设计”层面上没有意义，其意义只及于软件实现层面。

// Private 继承意味 is-implemented-in-terms-of（根据某物实现出）。
// 条款 38 才刚指出复合（composition）的意义也是这样。你如何在两者之间取舍？
// 答案很简单：尽可能使用复合，必要时才使用 private 继承。
// 何时才是必要？主要当 protected 成员和 / 或 virtual 函数牵扯进来的时候。
// 其实还要一种激进情况，那是当空间方面的利害关系足以踢翻 private 继承的支柱时。





class Timer {
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;            // 定时器每滴答一次，
    ...                                     // 此函数就被自动调用一次。
};

// 为了让 Widget 重新定义 Timer 内的 virtual 函数，widget 必须继承自 Timer。
// 但 public 继承在此例并不适当，因为 Widget 并不是个 Timer。是呀，
// Widget 客户总不该能够对着一个 Widget 调用 onTick 吧，因为观念上那并不是
// Widget 接口的一部分。如果允许那样的调用动作，很容易造成客户不正确地使用 Widget 接口，
// 那么违反条款 18 的忠告：“让接口容易被正确使用，不易被误用”。
// 在这类，public 继承不是个好策略。
// 我们必须以 private 形式继承 Timer：
class Widget : private Timer {
private:
    virtual void onTick() const;        // 查看 Widget 的数据...等等。
    ...
};

// 藉由 private 继承，Timer 的 public onTick 函数在 Widget 内变成 private，
// 而我们重新声明（定义）时仍然把它留在那儿。再说一次，把 onTick 放进
// public 接口内会误导客户端以为他们可以调用它，那就违反了条款 18。





// 这是个好设计，但不值几文钱，因为 private 继承并非绝对必要。如果我们决定
// 以复合（composition）取而代之，是可以的。只要在 Widget 内声明一个嵌套
// 式 private class，后者以 public 形式继承 Timer 并重新定义 onTick，
// 然后放一个这种类型的对象于 Widget 内。
class Widget {
private:
    class WidgetTimer : public Timer {
    public:
        virtual void onTick() const;
        ...
    };
    WidgetTimer timer;
    ...
};

// 首先，你或许会想设计 Widget 使它得以拥有 derived classes，但同时你可能会
// 想阻止 derived classes 重新定义 onTick。如果 Widget 继承自 Timer，
// 上面的想法就不可能实现，即使是 private 继承也不可能。（还记得吗，条款 35 曾
// 说 derived classes 可以重新定义 virtual 函数，即使它们不得调用它。）
// 但如果 WidgetTimer 是 Widget 内部的一个 private 成员并继承 Timer，Widget 的
// derived classes 将无法取用 WidgetTimer，因此无法继承它或重新定义它的 virtual 函数。
// 如果你曾经以 Java 或 C# 编程并怀念“阻止 derived classes 重新定义 virtual
// 函数”的能力（也就是 Java 的 final 和 C# 的 sealed）。

// 第二，你或许会想要将 Widget 的编译依存性降至最低。如果 Widget 继承 Timer，
// 当 Widget 被编译时 Timer 的定义必须可见，所以定义 Widget 的那个文件恐怕必须
// #include Timer.h。但如果 WidgetTimer 移出 Widget 之外而 Widget 内含指针
// 指向一个 WidgetTimer，Widget 可以只带着一个简单的 WidgetTimer 声明式，
// 不再需要 #include 任何与 Timer 有关的东西。对大型系统而言，如此的解耦
// （decouplings）可能是重要的措施。关于编译依存性的最小化，详见条款 31。





// 有一种激进情况涉及空间最优化，可能会促使你选择“private 继承”而不是“继承加复合”。
// 这种激进情况真是有够激进，只适用于你所处理的 class 不带任何数据时。
// 这样的 classes 没有 non-static 成员变量，没有 virtual 函数（因为这种
// 函数的存在会为每个对象带来一个 vptr，见条款 7），也没有 virtual base
// classes（因为这样的 base classes 也会招致体积上的额外开销，见条款 40）。
// 于是这种所谓的 empty classes 对象不使用任何空间，因为没有任何隶属对象的数据需要存储。
// 然而由于技术上的理由，C++ 裁定凡是独立（非附属）对象的必须有非零大小，
// 所以如果你这样做：
class Empty {  };           // 没有数据，所以其对象应该不适用任何内存

class HoldsAnInt {          // 应该只需要一个 int 空间
private:
    int x;
    Empty e;                // 应该不需要任何内存
};

// 你会发现 sizeof(HoldsAnInt) > sizeof(int); 喔噢，一个 Empty 成员变量竟然要求内存。
// 在大多数编译器中 sizeof(Empty) 获得 1，因为面对“大小为零之独立（非附属）对象”，
// 通常 C++ 官方勒令默默安插一个 char 到空对象内。然而齐位需求（alignment，见条款 50）
// 可能造成编译器为类似 HoldsAnInt 这样的 class 加上一些衬垫（padding），所以有可能
// HoldsAnInt 对象不只获得一个 char 大小，也许实际上被放大到足够有存放一个 int。

// 但或许你注意到了，我很小心地说“独立（非附属）”对象的大小一定不为零。
// 也就是说，这个约束不适用于 derived class 对象内的 base class 成分，
// 因为它们并非独立（非附属）。如果你继承 Empty，而不是内含一个那种类型的对象：
class HoldsAnInt : private Empty {
private:
    int x;
};

// 几乎可以确定 sizeof(HoldsAnInt) == sizeof(int)。这是所谓的 EBO
// （empty base optimization；空白基类最优化）。

// 现实中的“empty” classes 并不是真的是 empty。虽然它们从未拥有 non-static 成员变量，
// 却往往内含 typedefs，enums，static 成员变量，或 non-virtual 函数。
// STL 就有许多技术用途的 empty classes，其中内含有用的成员（通常是 typedefs），
// 包括 base classes unary_function 和 binary_function，这些是“用户自定义之函数对象”
// 通常会继承的 classes。感谢 EBO 的广泛实践，使这样的继承很少增加
// derived classes 的大小。




// 大多数继承相当于 is-a，这是指 public 继承，不是 private 继承。
// 复合和 private 继承都意味 is-implemented-in-terms-of，但复合比较容易理解，
// 所以无论什么时候，只要可以，你还是应该选择复合。

// 当你面对“并不存在 is-a 关系”的两个 classes，其中一个需要访问另一个的 protected 成员，
// 或需要重新定义其一或多个 virtual 函数，private 继承极有可能成为正统设计策略。
```

<br>

### 条款 40：明智而审慎地使用多重继承

- 多重继承比单一继承复杂。它可能导致新的歧义性，以及对 virtual 继承的需要。
- virtual 继承会增加大小、速度、初始化（及赋值）复杂度等等成本。如果 virtual base classes 不带任何数据，将是最具实用价值的情况。
- 多重继承的确有正当用途。其中一个情节涉及“public 继承某个 Interface class”和“private 继承某个协助实现的 class”的两相组合。

``` cpp {.line-numbers}
// 最先需要认清的一件事是，当 MI 进入设计景框，程序有可能从一个以上的 base classes
// 继承相同的名称（如函数、typedef等等）。那会导致较多的歧义（ambiguity）机会。例如：
class BorrowableItem {           // 图书馆允许你借某些东西
public:
    void checkOut();            // 离开时进行检查
    ...
};

class ElectronicGadget {
private:
    bool checkOut() const;      // 执行自我检测，返回是否测试成功
    ...
};

class MP3Player :               // 注意这里的多重继承
    public BorrowableItem,      // （某些图书馆愿意借出 MP3 播放器）
    public ElectronicGadget
{ ... };                        // 这里，class 的定义不是我们的关心重点

MP3Player mp;
mp.checkOut();                  // 歧义！调用的是哪个 checkOut？

// 注意此例之中对 checkOut 的调用时歧义（模棱两可）的，即使两个函数之中
// 只有一个可取用（BorrowableItem 内的 checkOut 是 public，ElectronicGadget
// 内的却是 private）。这与 C++ 用来解析（resolving）重载函数调用的规则相符：
// 在看到是否有个函数可取用之前，C++ 首先确认这个函数对此调用之言是最佳匹配。
// 找出最佳匹配函数后才检验其可取用性。本例的两个 checkOuts 有相同的匹配程度
// （译注：因此才造成歧义），没有所谓最佳匹配。因此 ElectronicGadget::checkOut
// 的可取用性也就是从未被编译器审查。

// 为了解决这个歧义，你必须明白指出你要调用哪一个 base class 内的函数：
mp.BorrowableItem::checkOut();      // 哎呀，原来是这个 checkOut...

// 你当然也可以尝试明确调用 ElectronicGadget::checkOut，但然后你会获得
// 一个“尝试调用 private 成员函数”的错误。





// 多重继承的意思是继承一个以上的 base classes，但这些 base classes 并不
// 常在继承体系中又有更高级的 base classes，因为那会导致要命的“钻石型多重继承”：
class File { ... };
class InputFile : public File { ... };
class OutputFile : public File { ... };
class IOFile : public InputFile, public OutputFile { ... };

// 从某个角度说，IOFile 从其每一个 base class 继承一份，所以其对象内
// 应该有两份 fileName 成员变量。但从另一个角度说，简单的逻辑告诉我们，
// IOFile 对象只该有一个文件名称，所以它继承自两个 base classes
// 而来的 fileName 不该重复。





// C++ 在这场辩论中并没有倾斜立场；两个方案它都支持——虽然其缺省做法是执行复制
// （也就是上一段说的第一个做法）。如果那不是你要的，你必须令那个带有此数据的
// class 也就是 File 成为一个 virtual base class。为了这样做，
// 你必须令所有直接继承自它的 classes 采用 “virtual 继承”：
class File { ... };
class InputFile : virtual public File { ... };
class OutputFile : virtual public File { ... };
class IOFile : public InputFile, public OutputFile { ... };

// 从正确行为的观点看，public 继承应该总是 virtual。如果这是唯一一个观点，规则很简单：
// 任何时候当你使用 public 继承，请改用 virtual public 继承。
// 但是，啊呀，正确性并不是唯一观点。
// 为避免继承得来的成员变量重复，编译器必须提供若干幕后戏法，而其后果是：
// 使用 virtual 继承的那些 classes 所产生的对象往往比使用 non-virtual 继承
// 的兄弟们体积大，访问 virtual base classes 的成员变量时，也比访问
// non-virtual base classes 的成员变量速度慢。种种细节因编译器不同而异，
// 但基本重点很清楚：你得为 virtual 继承付出代价。

// virtual base 的初始化责任是由继承体系中的最低层（most derived）class 负责，
// 这暗示（1）classes 若派生自 virtual bases 而需要初始化，必须认知其 virtual bases
// —— 不论那些 bases 距离多远，（2）当一个新的 derived class 加入继承体系中，
// 它必须承担其 virtual bases（不论直接或间接）的初始化责任。





// 我对 virtual base classes（亦相当于对 virtual 继承）的忠告很简单。
// 第一，非必要不适用 virtual bases。平常请使用 non-virtual 继承。
// 第二，如果你必须使用 virtual base classes，尽可能避免在其中放置数据。
// 这么一来你就不需担心这些 classes 身上的初始化（和赋值）所带来的诡异事情了。
// Java 和 .NET 的 Interfaces 值得注意，它在许多方面兼容于 C++ 的 virtual
// base classes，而且也不允许含有任何数据。

// 现在让我们看看下面这个用来塑模“人”的 C++ Interface class（见条款 31）：
class IPerson {
public:
    virtual ~IPerson();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
};

// IPerson 的客户必须以 IPerson 的 pointers 和 references 来编写程序，
// 因为抽象 classes 无法被实体化创建对象。为了创建一些可被当做 IPerson 来使用的对象，
// IPerson 的客户使用 factory functions（工厂函数，见条款 31）
// 将“派生自 IPerson 的具象 classes” 实体化：

// factory function（工厂函数），根据一个独一无二的数据库 ID 创建一个 Person 对象。
// 条款 18 告诉你为什么返回类型不是原始指针。
std::tr1::shared_ptr<IPerson> makePerson(DatabaseID personIdentifier);

// 这个函数从使用者手上取得一个数据库 ID
DatabaseID askUserForDatabaseID();

DatabaseID id(askUserForDatabaseID());
std::tr1::shared_ptr<IPerson> pp(makePerson(id));
                                            // 创建一个对象支持 IPerson 接口。
...                                         // 藉由 IPerson 成员函数处理 *pp。

// 但是 makePerson 如何创建对象并返回一个指针指向它呢？
// 无疑地一定有某些派生自 IPerson 的具象 class，在其中 makePerson 可以创建对象。

// 假设这个 class 名为 CPerson。就像具象 class 一样，CPerson 必须提供“继承自 IPerson”
// 的 pure virtual 函数的实现代码。我们可以从无到有写出这些东西，
// 更好的是利用既有组件，后者做了大部分或所有必要事情。
// 例如，假设有个既有的数据库相关 class，名为 PersonInfo，
// 提供 CPerson 所需要的实质东西：
class PersonInfo {
public:
    explicit PersonInof(DatabaseID pid);
    virtual ~PersonInfo();
    virtual const char* theName() const;
    virtual const char* theBirthDate() const;
    ...
private:
    virtual const char* valueDelimOpen() const;     // 详下
    virtual const char* valueDelimClose() const;
    ...
};

const char* PersonInfo::valueDelimOpen() const
{
    return "[";                 // 缺省的起始符号
};

const char* PersonInfo::valueDelimClose() const
{
    return "]";                 // 缺省的结尾符号
}

const char* PersonInfo::theName() const
{
    // 保留缓冲区给返回值使用；由于缓冲区是 static，因此会被自动初始化为“全部是 0”
    static char value[Max_Formatted_Field_Value_Length];

    // 写入起始符号
    std::strcpy(value, valueDelimOpen());

    现在，将 value 内的字符串添附到这个对象的 name 成员变量中
    （小心，避免缓冲区超限）

    // 写入结尾符号
    std::strcat(value, valueDelimClose());
    return value;
}





// 本例之中 CPerson 需要重新定义 valueDelimOpen 和 valueDelimClose，
// 所以单纯的复合无法应付。最直接的解法就是令 CPerson 以 private 形式
// 继承 PersonInfo，虽然条款 39 也说过，只要多加一点工作，
// CPerson 也可以结合“复合 + 继承”技术以求有效重新定义 PersonInfo 的
// virtual 函数。此处我将使用 private 继承。

// 但 CPerson 也必须实现 IPerson 接口，那需得以 public 继承才能完成。
// 这导致多重继承的一个通情达理的应用：将“public 继承自某接口”
// 和“private 继承自某实现”结合在一起：
class IPerson {                         // 这个 class 指出需实现的接口
public:
    virtual ~IPerson();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
};

class DatabaseID { ... };               // 稍后被使用；细节不重要。

class PersonInfo {                      // 这个 class 有若干有用函数，
public:                                 // 可用以实现 IPerson 接口。
    explicit PersonInfo(DatabaseID pid);
    virtual ~PersonInfo();
    virtual const char* theName() const;
    virtual const char* theBirthDate() const;
    virtual const char* valueDelimOpen() const;
    virtual const char* valueDelimClose() const;
    ...
};

class CPerson : public IPerson, private PersonInfo {
public:
    explicit CPerson(DatabaseID pid) : PersonInfo(pid) {  }
    
    virtual std::string name() const        // 实现必要的 IPerson 成员函数
    { return PersonInfo::theName(); }

    virtual std::string birthDate() const   // 实现必要的 IPerson 成员函数
    { return PersonInfo::theBirthDate(); }

private:                                                // 重新定义继承而来的
    const char* valueDelimOpen() const { return ""; }   // virtual “界限函数”
    const char* valueDelimClose() const { return ""; }
};
```

<br>

## 7、模板与泛型编程

<br>

### 条款 41：了解隐式接口和编译期多态

- classes 和 templates 都支持接口（interfaces）和多态（polymorphism）。
- 对 classes 而言接口是显示的（explicit），以函数签名为中心。多态则是通过 virtual 函数发生于运行期。
- 对 template 参数而言，接口是隐式的（implicit），奠基于有效表达式。多态则是通过 template 具现化和函数重载解析（function overloading resolution）发生于编译期。

``` cpp {.line-numebrs}
// 面向对象编程世界总是以显示接口（explicit interfaces）和
// 运行期多态（runtime polymorphism）解决问题。
class Widget {
public:
    Widget();
    virtual ~Widget();
    virtual std::size_t size() const;
    virtual void normalize();
    void swap(Widget& other);               // 见条款 25
    ...
};

void doProcessing(Widget& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        Widget temp(w);
        temp.normalize();
        temp.swap(w);
    }
}

// 我们可以这样说 doProcessing 内的 w：
// （1）由于 w 的类型被声明为 Widget，所以 w 必须支持 Widget 接口。
// 所以我称此为一个显示接口（explicit interface），也就是它在源码中明确可见。
// （2）由于 Widget 的某些成员函数是 virtual，w 对那些函数的调用
// 将表现出运行期多态（runtime polymorphism），也就是说将于运行期根据 w 的
// 动态类型（见条款 37）决定究竟调用哪一个函数。





// Templates 及泛型编程的世界，与面向对象有根本上的不同。
// 在世界中显示接口和运行期多态仍然存在，但重要性降低。
// 反倒是隐式接口（implicit interfaces）和编译器多态
// （compile-time polymorphism） 移到前头了。
// 若想知道那是什么，看看当我们将 doProcessing 从函数
// 转变成函数模板（function template）时发生什么事：
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        T temp(w);
        temp.normalize();
        temp.swap(w);
    }
}

// 现在我们怎么说 doProcessing 内的 w 呢？
// （1）w 必须支持哪一种接口，系由 template 中执行于 w 身上的操作来决定。
// 本例看来 w 的类型 T 好像必须支持 size，normalize 和 swap 成员函数、
// copy 构造函数（用以建立 temp）、不等比较（inequality comparison，
// 用来比较 someNasty-Widget）。我们很快会看到这并非完全正确，但对目前而言足够真实。
// 重要的是，这一组表达式（对此 template 而言必须有效编译）便是 T 必须支持的一组隐式接口
// （implicit interface）。
// （2）凡涉及 w 的任务函数调用，例如 operator> 和 operator !=，有可能造成 template
// 具现化（instantiated），使这些调用得以成功。这样的具现行为发生在编译期。
// “以不同的 template 参数具现化 function templates”会导致调用不同的函数，
// 这便是所谓的编译器多态（compile-time polymorphism）。





// 纵使你从未用过 templates，应该不陌生“运行期多态”和“编译器多态”之间的差异，
// 因为它类似于“哪一个重载函数该被调用”（发生在编译期）和
// “哪一个 virtual 函数该被绑定”（发生在运行期）之间的差异。





// 通常显示接口由函数的签名式（也就是函数名称、参数类型、返回类型）构成。例如 Widget class：
class Widget {
public:
    Widget();
    virtual ~Widget();
    virtual std::size_t size() const;
    virtual void normalize();
    void swap(Widget& other);
};

// 隐式接口就完全不同了。它并不基于函数签名式，而是由有效表达式（valid expression）组成。
// 再次看看 doProcessing template 一开始的条件：
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        ...
    }
}

// T（w 的类型）的隐式接口看起来好像有这些约束：
// （1）它必须提供一个名为 size 的成员函数，该函数返回一个整数值。
// （2）它必须支持一个 operator!= 函数，用来比较两个 T 对象。
// 这里我们假设 someNastyWidget 的类型为 T。
```

<br>

### 条款 42：了解 typename 的双重意义

- 声明 template 参数时，前缀关键字 class 和 typename 可互换。
- 请使用关键字 typename 标识嵌套从属类型名称；但不得在 base class lists（基类列）或 member initialization list（成员初值列）内以它作为 base class 修饰符。

``` cpp {.line-numbers}
// 提一个问题：以下 template 声明式中，class 和 typename 有什么不同？
template<class T> class Widget;         // 使用“class”
template<typename T> class Widget;      // 使用“typename”

// 答案：没有不同。当我们声明 template 类型参数，class 和 typename 的意义完全相同。
// 从 C++ 的角度来看，声明 template 参数时，不论使用
// 关键字 class 或 typename，意义完全相同。





// 然而 C++ 并不总是把 class 和 typename 视为等价。有时候你一定得使用 typename。
// 为了了解其时机，我们必须先谈谈你可以在 template 内指涉（refer to）的两种名称。

// 下面是实践愚蠢想法的一种方式：
template<typename C>
void print2nd(const C& container)       // 打印容器内的第二元素
{                                       // 注意这不是有效的 C++ 代码
    if (container.size() >= 2) {
        C::const_iterator iter(container.begin());      // 取得第一元素的迭代器
        ++iter;                                         // 将 iter 移往第二元素
        int value = *iter;                              // 将该元素复制到某个 int。
        std::cout << value;                             // 打印那个 int。
    }
}

// iter 的类型是 C::const_iterator，实际是什么必须取决于 template 参数 C。
// template 内出现的名称如果相依于某个 template 参数，称之为从属名称（dependent names）。
// 如果从属名称在 class 内呈嵌套状，我们称它为嵌套从属名称（nested dependent name）。
// C::const_iterator 就是这样一个名称。实际上它还是个嵌套从属类型名称
// （nested dependent name），也就是个嵌套从属名称并且指涉某类型。

// print2nd 内的另一个 local 变量 value，其类型是 int。int 是一个并不倚赖任何
// template 参数的名称。这样的名称是谓非从属名称（non-dependent names）。
// 我不知道为什么不叫做独立名称（independent names）。





// 嵌套从属名称有可能导致解析（parsing）困难。
// 举个例子，假设我们令 print2nd 更愚蠢些，这样起头：
template<typename C>
void print2nd(const C& container)
{
    C::const_iterator* x;
    ...
}

// 如果 C::const_iterator 不是个类型呢？如果 C 有个 static 成员变量而
// 碰巧被命名为 const_iterator，或如果 x 碰巧是个 global 变量名称呢？
// 那样的话上述代码就不再是声明一个 local 变量，而是一个相乘动作：
// C::const_iterator 乘以 x。





// 现在再次看看 print2nd 起始处：
template<typename C>
void print2nd(const C& container)
{
    if (container.size() >= 2) {
        C::const_iterator iter(container.begin());      // 这个名称被
        ...                                             // 假设为非类型
    }
}

// 现在应该很清楚为什么这不是有效的 C++ 代码了吧。
// iter 声明式只有在 C:const_iterator 是个类型时才合理，
// 但我们并没有告诉 C++ 说它是，于是 C++ 假设它不是。
// 若要矫正这个形式，我们必须告诉 C++ 说 C::const_iterator 是个类型。
// 只要紧临它之前放置关键字 typename 即可：
template<typename C>                    // 这是合法的 C++ 代码
void print2nd(const C& container)
{
    if (container.size() >= 2) {
        typename C::const_iterator iter(container.begin());
        ...
    }
}

// 一般性规则很简单：任何时候当你想要在 template 中指涉一个嵌套从属类型名称，
// 就必须在紧临它的前一个位置放上关键字 typename。





// typename 只被用来验明嵌套从属类型名称；其他名称不该有它存在。
// 例如下面这个 function template，接受一个容器和一个“指向该容器”的迭代器：
template<typename C>                    // 允许使用“typename”（或“class”）
void f(const C& container,              // 不允许使用“typename”
       typename C::iterator iter);      // 一定要使用“typename”

// 上述的 C 并不是嵌套从属类型名称（它并非嵌套于任何“取决于 template 参数”的东西内），
// 所以声明 container 时并不需要以 typename 为前导，但 C::iterator
// 是个嵌套从属名称，所以必须以 typename 为前导。





// “typename 必须作为嵌套从属类型名称的前缀词”这一规则的例外是，
// typename 不可以出现在 base classes list 内的嵌套从属类型名称之前，
// 也不可在 member initialization list（成员初值列）中作为 base class 修饰符。
// 例如：
template<typename T>
class Derived : public Base<T>::Nested {        // base class list 中
public:                                         // 不允许“typename”。
    explicit Derived(int x)
        : Base<T>::Nested(x)                    // mem.init.list 中
    {                                           // 不允许 “typename”。
        typename Base<T>::Nested temp;          // 嵌套从属类型名称，
        ...             // 既不在 base class list 中也不在 mem.init.list 中，
    }                   // 作为一个 base class 修饰符需加上 typename。
    ...
};





template<typename IterT>
void workWithIterator(IterT iter)
{
    typename std::iterator_traits<IterT>::value_type temp(*iter);
    ...
}

// 别让 std::iterator_traits<IterT>::value_type 惊吓了你，
// 那只不过是标准 traits class（见条款 47）的一种运用，
// 相当于说“类型为 IterT 之对象所指物的相同类型，并将 temp 初始化 iter 所指物”。
// 如果 IterT 是 vector<int>::iterator，temp 的类型就是 int。
// 如果 IterT 是 vector<string>::iterator，temp 的类型就是 string。
// 由于 std::iterator_traits<IterT>::value_type 是个嵌套从属类型名称
// （value_type 被嵌套于 iterator_traits<IterT> 之内而 IterT 是个 template 参数），
// 所以我们必须在它之前放置 typename。





// 普通的习惯是设定 typedef 名称用以代表某个 traits 成员名称，
// 于是常常可以看到类似这样的 local typedef：
template<typename IterT>
void workWithIterator(IterT iter)
{
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    value_type temp(*iter);
    ...
}

// 许多程序员最初认为把“typedef typename”并列颇不和谐，
// 但它是在是指涉“嵌套从属类型名称”的一个合理附带结果。
```

<br>

### 条款 43：学习处理模板化基类内的名称

- 可在 derived class templates 内通过“this->”指涉 base class templates 内的成员名称，或藉由一个明白写出的“base class 资格修饰符”完成。

``` cpp {.line-numbers}
// 假设我们需要撰写一个程序，它能够传送信息到若干不同的公司去。
// 信息要不译成密码，要不就是未经加工的文字。如果编译期间我们有足够信息
// 来决定哪一个信息传至哪一家公司，就可以采用基于 template 的解法：
class CompanyA {
public:
    ...
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string& msg);
    ...
};

class CompantyB {
public:
    ...
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string& msg);
    ...
};
...                         // 针对其他公司设计的 classes。

class MsgInfo { ... };      // 这个 class 用来保存信息，以备将来产生信息

template<typename Company>
class MsgSender {
public:
    ...                     // 构造函数，析构函数等等
    void sendClear(const MsgInfo& info)
    {
        std::string msg;
        在这儿，根据 info 产生信息；
        Company c;
        c.sendCleartext(msg);
    }

    void sendSecret(const MsgInfo& info)        // 类似 sendClear，唯一不同是
    { ... }                                     // 这里调用 c.sendEncrypted
};

// 这个做法行得通。但假设我们有时候想要在每次送出信息时志记（log）某些信息。
// derived class 可轻易加上这样的生产力，那似乎是个合情合理的解法：
template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    ...                         // 构造函数、析构函数等等。
    void sendClearMsg(const MsgInfo& info)
    {
        将“传送前”的信息写至 log；
        sendClear(info);        // 调用 base class 函数；这段代码无法通过编译。
        将“传送后”的信息写至 log；
    }
    ...
};

// 注意这个 derived class 的信息传送函数有一个不同的名称（sendClearMsg），
// 与其 base class 内的名称（sendClear）不同。那是个好设计，因为它避免遮掩
// “继承而得的名称”（见条款 33），也避免重新定义一个继承而得的 
// non-virtual 函数（见条款 36）。但上述代码无法通过编译，
// 至少对严守规律的编译器而言。这样的编译器会抱怨 sendClear 不存在。
// 我们的眼镜可以看到 sendClear 的确在 base class 内，编译器却看不到它们。为什么？

// 问题在于，当编译器遭遇 class template LoggingMsgSender 定义式时，
// 并不知道它继承什么样的 class。当然它继承的是 MsgSender<Company>，
// 但其中的 Company 是个 template 参数，不到后来（当 LoggingMsgSender 被具现化）
// 无法确切知道它是什么。而如果不知道 Company 是什么，就无法知道
// class MsgSender<Company> 看起来想什么——更明确地说是没办法
// 知道它是否有个 sendClear 函数。

// 为了让问题更具体化，假设我们有个 class CompanyZ 坚持使用加密通讯：
class CompanyZ {                // 这个 class 不提供
public:                         // sendCleartext 函数
    ...
    void sendEncrypted(const std::string& msg);
    ...
};

// 一般性的 MsgSender template 对 CompanyZ 并不合适，因为那个 template 提供
// 了一个 sendClear 函数（其中针对其类型参数 Company 调用了 sendCleartext 函数），
// 而这对 CompanyZ 对象并不合理。欲矫正这个问题，
// 我们可以针对 CompanyZ 产生一个 MsgSender 特化版：
template<>                          // 一个全特化的
class MsgSender<CompanyZ> {         // MsgSender：它和一般 template 相同，
public:                             // 差别只在于它删掉了 sendClear。
    ...
    void sendSecret(const MsgInfo& info)
    { ... }
};

// 注意 class 定义式最前有的 “template<>”的语法象征这既不是 template 也不是标准 class，
// 而是个特化版的 MsgSender template，在 template 实参是 CompanyZ 时被使用。
// 这是所谓的模板全特化（total template specialization）：
// template MsgSender 针对类型 CompanyZ 特化了，而且其特化是全面性的，
// 也就是说一旦类型参数被定义为 CompanyZ，再没有其他 template 参数可供变化。





template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        将“传送前”的信息写至 log；
        sendClear(info);            // 如果 Company == CompanyZ，这个函数不存在。
        将“传送后”的信息写至 log；
    }
    ...
};

// 正如注释所言，当 base class 被指定为 MsgSender<CompanyZ> 时这段代码不合法，
// 因为那个 class 并未提供 sendClear 函数！那就是为什么 C++ 拒绝这个调用的原因：
// 它知道 base class templates 有可能被特化，而那个特化版本可能不提供和一般性
// template 相同的接口。因此它往往拒绝在 templatized base classes
// （模板化基类，本例的 MsgSender<Company>）内寻找继承而来的名称（本例的 SendClear）。
// 就某种意义而言，当我们从 Object Oriented C++ 跨进 Template C++（见条款 1），
// 继承就不像以前那般畅行无阻了。

// 为了重头来过，我们必须有某种办法令 C++ “不进入 templatized
// base classes 观察”的行为失效。有三个办法，
// 第一是在 base class 函数调用动作之前加上“this->”：
template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        将“传送前”的信息写至 log；
        this->sendClear(info);          // 成立，假设 sendClear 将被继承。
        将“传送后”的信息写至 log；
    }
    ...
};

// 第二是使用 using 声明式。如果你已经读过条款 33，这个解法应该会令你感到熟悉。
// 条款 33 描述 using 声明式如何将“被掩盖的 base class 名称”带入一个 derived class
// 作用域内。我们可以这样写下 sendClearMsg：
template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    using MsgSender<Company>::sendClear;        // 告诉编译器，请它假设
    ...                                         // sendClear 位于 base class 内。
    void sendClearMsg(const MsgInfo& info)
    {
        ...
        sendClear(info);            // OK，假设 sendClear 将被继承下来。
        ...
    }
    ...
};

// （虽然 using 声明式在这里或在条款 33 都可有效运作，但两处解决的问题其实不相同。
// 这里的情况并不是 base class 名称被 derived class 名称遮掩，
// 而是编译器不进入 base class 作用域内查找，于是我们通过 using 告诉它，请它那么做。）

// 第三个做法是，明白指出被调用的函数位于 base class 内：
template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        ...
        MsgSender<Company>::sendClear(info);    // OK，假设 sendClear
                                                // 将被继承下来。
    }
};

// 但这往往是最不让人满意的一个解法，因为如果被调用的是 virtual 函数，
// 上述的明确资格修饰（explicit qualification）会关闭“virtual 绑定行为”。





// 从名称可视点（visibility point）的角度出发，上述每一个解法做的事情都相同：
// 对编译器承诺“base class template 的任何特化版本都将支持其一般（泛化）
// 版本所提供的接口”。这样一个承诺是编译器在解析（parse）像 LoggingMsgSender
// 这样的 derived class template 时需要的。但如果这个承诺最终未被实践出来，
// 往后的编译最终还是会还给事实一个公道。举个例子，如果稍后的源码内含这个：
LoggingMsgSender<CompanyZ> zMsgSender;
MsgInfo msgData;
...                                 // 在 msgData 内放置信息。
zMsgSender.sendClearMsg(msgData);   // 错误！无法通过编译。

// 其中对 sendClearMsg 的调用动作将无法通过编译，因为在那点上，
// 编译器知道 base class 是个 template 特化版本 MsgSender<CompanyZ>，
// 而且它们知道那个 class 不提供 sendClear 函数，而后者却是 sendClearMsg
// 尝试调用的函数。





// 根本而言，本条款探讨的是，面向“指涉 base class members”之无效 references，
// 编译器的诊断时间可能发生在早期（当解析 derived class template 的定义式时），
// 也可能发生在晚期（当那些 templates 被特定之 template 实参具现化时）。
// C++ 的政策是宁愿较早诊断，这就是为什么“当 base classes 从 templates
// 中被具现化时”它假设它对那些 base classes的内容毫无所悉的缘故。
```

<br>

### 条款 44：将与参数无关的代码抽离 templates

- Templates 生成多个 classes 和多个函数，所以任何 template 代码都不该与某个造成膨胀的 template 参数产生相依关系。
- 因非类型模板参数（non-type template parameters）而造成的代码膨胀，往往可消除，做法是以函数参数或 class 成员变量替换 template 参数。
- 因类型参数（type parameters）而造成的代码膨胀，往往可降低，做法是让带有完全相同二进制表述（binary representations）的具现类型（instantiation types）共享实现码。

``` cpp {.line-numbers}
// Templates 是节省时间和避免代码重复的一个奇方妙法。不再需要键入 20 个类似的 classes
// 而每一个带有 15 个成员函数，你只需要键入一个 class template，留给编译器去
// 具现化那 20 个你需要的相关 classes 和 300 个函数。





// 编写 templates 时，也是做相同的分析，以相同的方式避免重复，但其中有个窍门。
// 在 non-template 代码中，重复十分明确：你可以“看”到这两个函数或两个 classes
// 之间有所重复。然而在 template 代码中，重复是隐晦的：毕竟只存在一份 template 源码，
// 所以你必须训练自己去感受当 template 被具现化多次时可能发生的重复。

// 举个例子，假设你想为固定尺寸的正方矩阵编写一个 template。
// 该矩阵的性质之一是支持逆矩阵运算（matrix inversion）。
template<typename T,            // template 支持 n x n 矩阵，元素是
         std::size_t n>         // 类型为 T 的 objects；见以下
class SquareMatrix {            // 关于 size_t 参数的信息
public:
    ...
    void invert();              // 求逆矩阵
};

// 这个 template 接收一个类型参数 T，除此之外接受一个类型为 size_t 的参数，
// 那是个非类型参数（non-type parameter）。这种参数和类型参数比起来较不常见，
// 但它们完全合法，而且就像本例一样，相当自然。

// 现在，考虑这些代码：
SquareMatrix<double, 5> sm1;
...
sm1.invert();                   // 调用 SquareMatrix<double, 5>::invert
SquareMatrix<double, 10> sm2;
...
sm2.invert();                   // 调用 SquareMatrix<double, 10>::invert

// 这会具现化两份 invert。这些函数并非完完全全相同，因为其中一个操作的是 5 * 5 矩阵
// 而另一个操作的是 10 * 10 矩阵，但除了常量 5 和 10，两个函数的其他部分完全相同。
// 这是 template 引出代码膨胀的一个典型例子。





// 如果你看到两个函数完全相同，只除了一个使用 5 而另一个使用 10，你会怎么做？
// 你的本能会为它们建立一个带数值参数的函数，然后以 5 和 10 来调用这个带参数的函数，
// 而不重复代码。你的本能很好，下面是对 SquareMatrix 的第一次修改：
template<typename T>                            // 与尺寸无关的 base class，
class SquareMatrixBase {                        // 用于正方形矩阵
protected:
    ...
    void invert(std::size_t matrixSize);        // 以给定的尺寸求逆矩阵
    ...
};

template<typename T, std::size_t n>
class SquareMatrix : private SquareMatrixBase<T> {
private:
    using SquareMatrixBase<T>::invert;          // 避免遮掩 base 版的
                                                // invert；见条款 33
    
public:
    ...
    void invert() { this->invert(n); }          // 制造一个 inline 调用，调用
                                                // base class 版的 invert。稍后
                                                // 说明为什么这儿出现 this->
};

// SquareMatrixBase::invert 只是企图成为“避免 derived classes 代码重复”
// 的一种方法，所以它以 protected 替换 public。调用它而造成的额外成本应该是 0，
// 因为 derived classes 的 inverts 调用 base class 版本时用的是 inline 调用
// （这里的 inline 是隐晦的，见条款 30）。这些函数使用“this->”记号，因为若不这样做，
// 便如条款 43 所说，模板化基类（templatized base classes，例如 SquareMatrixBase<T>）
// 内的函数名称会被 derived classes 掩盖。也请注意 SquareMatrix 和
// SquareMatrixBase 之间的继承关系是 private。这反应一个事实：
// 这里的 base class 只是为了帮助 derived classes 实现，不是为了表现 SquareMatrix
// 和 SquareMatrixbase 之间的 is-a 关系（关于 private 继承，见条款 39）。

// SquareMatrixBase::invert 如何知道该操作什么数据？虽然它从参数中知道矩阵尺寸，
// 但它如何知道哪个特定矩阵的数据在哪儿？想必只有 derived class 知道。
// Derived class 如何联络其 base class 做逆运算动作？
// 一个可能的做法是为 SquareMatrixBase::invert 添加另一个参数，也许是个指针，
// 指向一块用来放置矩阵数据的内存起始点。那行得通，但十之八九 invert 不是唯一一个
// 可写为“形式与尺寸无关并可移至 SquareMatrixBase 内”的 SquareMatrix 函数。
// 如果有若干这样的函数，我们唯一要做的就是找出保存矩阵元素值得那块内存。
// 我们可以对所有这样的函数添加一个额外参数，却得一次又一次地告诉 SquareMatrixBase
// 相同的信息，这样似乎不好。

// 另一个办法是令 SquareMatrixBase 贮存一个指针，指向矩阵数值所在的内存。
// 而只要它存储了那些东西，也就可能存储矩阵尺寸。成果看起来像这样：
template<typename T>
class SquareMatrixBase {
protected:
    SquareMatrixBase(std::size_t n, T* pMem)        // 存储矩阵大小和一个
        : size(n), pData(pMem) {  }                 // 指针，指向矩阵数值。

    void setDataPtr(T* ptr) { pData = ptr; }        // 重新赋值给 pData。
    ...
private:
    std::size_t size;                               // 矩阵的大小。
    T* pData;                                       // 指针，指向矩阵内容。
};

// 这允许 derived classes 决定内存分配方式。某些实现版本也许会决定将矩阵数据存储在
// SquareMatrix 对象内部：
template<typename T, std::size_t n>
class SquareMatrix : private SquareMatrixBase<T> {
public:
    SquareMatrix()                                  // 送出矩阵大小和
        : SquareMatrixBase<T>(n, data) {  }         // 数据指针给 base class。
    ...
private:
    T data[n * n];
};

// 这种类型的对象不需要动态分配内存，但对象自身可能非常大。另一种做法是
// 把每一个举矩阵的数据放进 heap（也就是通过 new 来分配内存）：
template<typename T, std::size_t n>
class SquareMatrix : private SquareMatrixBase<T> {
public:
    SquareMatrix()                          // 将 base class 的数据指针设为 null，
        : SquareMatrixBase<T>(n, 0),        // 为矩阵内容分配内存，
          pData(new T[n * n])               // 将指向该内存的指针存储起来，
    { this->setDataPtr(pData.get()); }      // 然后将它的一个副本交给 base class
    ...
private:
    boost::scoped_array<T> pData;           // 关于 boost::scoped_array，
                                            // 见条款 13。
};

// 不论数据存储于何处，从膨胀的角度检讨之，关键是现在许多——说不定是所有——
// SquareMatrix 成员函数可以单纯地以 inline 方式调用 base class 版本，
// 后者由“持有同型元素”（不论矩阵大小）之所有矩阵共享。在此同时，不同大小
// 的 SquareMatrix 对象有着不同的类型，所以即使（例如 SquareMatrix<double, 5>
// 成员函数，我们也没机会传递一个 SquareMatrix<double, 5> 对象到一个期望获得
// SquareMatrix<double, 10> 的函数去。很棒，对吗？）

// 是的，很棒，但必须付出代价。硬是绑着矩阵尺寸的那个 invert 版本，有可能生成
// 比共享版本（其中尺寸乃以函数参数传递或存储在对象内）更佳的代码。例如在尺寸
// 专属版中，尺寸是个编译期常量，因此可以藉由常量的广传达到最优化，包括把它们
// 折进被生成指令中成为直接操作数。这在“与尺寸无关”的版本中是无法办到的。
```

<br>

### 条款 45：运用成员函数模板接收所有兼容类型

- 请使用 member function templates（成员函数模板）生成“可接收所有兼容类型”的函数。
- 如果你声明 member templates 用于“泛化 copy 构造”或“泛化 assignment 操作”，你还是需要声明正常的 copy 构造函数和 copy assignment 操作符。

``` cpp {.line-numbers}
// 所谓智能指针（Smart pointers）是“行为像指针”的对象，并提供指针没有的机能。

// 真实指针做得很好的一件事是，支持隐式转换（implicit conversions）。
// Derived class 指针可以隐式转换为 base class 指针，“指向 non-const 对象”
// 的指针可以转换为“指向 const 对象”······等等。
// 下面是可能发生于三层继承体系的一些转换：
class Top { ... };
class Middle : public Top { ... };
class Bottom : public Middle { ... };

Top* pt1 = new Middle;                  // 将 Middle* 转换为 Top*
Top* pt2 = new Bottom;                  // 将 Bottom* 转换为 Top*
const Top* pct2 = pt1;                  // 将 Top* 转换为 const Top*

// 但如果想在用户自定的智能指针中模拟上述转换，稍稍有点麻烦。我们希望以下代码通过编译：
template<typnename T>
class SmartPtr {
public:                                 // 智能指针通常
    explicit SmartPtr(T* realPtr);      // 以内置（原始）指针完成初始化
    ...
};

SmartPtr<Top> pt1 =                     // 将 SmartPtr<Middle> 转换为
    SmartPtr<Middle>(new Middle);       // SmartPtr<Top>
SmartPtr<Top> pt2 =                     // 将 SmartPrt<Bottom> 转换为
    SmartPtr<Bottom>(new Bottom);       // SmartPtr<Top>
SmartPrt<const Top> pct2 = pt1;         // 将 SmartPtr<Top> 转换为
                                        // SmartPrt<const Top>

// 但是，同一个 template 的不同具现体（instantiations）之间不存在什么与生俱来的固有关系
// （译注：这里意指如果以带有 base-derived 关系的 B，D 两类型分别具现化某个 template，
// 产生出来的两个具现体不带有 base-derived 关系），所以编译器视 SmartPtr<Middle>
// 和 SmartPtr<Top> 为完全不同的 classes，它们之间的关系并不比······唔······
// 并不比 vector<float> 和 Widget 更密切，呵呵。为了获得我们希望获得的 SmartPtr
// classes 之间的转换能力，我们必须将它们明确地编写出来。





// Templates 和泛型编程（Generic Programming）

// 在上述智能指针实例中，每一个语句创建了一个新式智能指针对象，所以现在我们
// 应该关注如何编写智能指针的构造函数，使其行为能够满足我们的转型需要。
// 一个很具关键的观察结果是：我们永远无法写出我们需要的所有构造函数。在上述继承体系中，
// 我们根据一个 SmartPtr<Middle> 或一个 SmartPtr<Bottom> 构造出一个 SmartPrt<Top>，
// 但如果这个继承体系未来有所扩充，SmartPtr<Top>对象又必须能够根据其他智能指针
// 构造自己。假设日后添加了：
class BelowBottom : public Bottom { ... };

// 我们因此必须令 SmartPtr<BelowBottom> 对象得以生成SmartPtr<Top> 对象，
// 但我们当然不希望一再修改 SmartPtr template 以满足此类需求。

// 就原理而言，此例中我们需要的构造函数数量没有止尽，因为一个 template 可被无限量具现化，
// 以致生成无限量函数。因此，似乎我们需要的不是为 SmartPtr 写一个构造函数，
// 而是为它写一个构造模板。这样的模板（templates）是所谓 member function
// templates（常简称为 member templates），其作用是为 class 生成函数：
template<typename T>
class SmartPtr {
public:
    template<typename U>                    // member templates
    SmartPtr(const SmartPtr<U>& other);     // 为了生成 copy 构造函数
    ...
};

// 以上代码的意思是，对于任何类型 T 和任何类型 U，这里可以根据 SmartPtr<U> 生成
// 一个 SmartPtr<T>——因为 SmartPtr<T> 有个构造函数接受一个 SmartPtr<U> 参数。
// 这一类构造函数根据对象 u 创建对象 t（例如根据 SmartPtr<U> 创建一个 SmartPtr<T>），
// 而 u 和 v 的类型是同一个 template 的不同具现体，有时我们称之为
// 泛化（generalized）copy 构造函数。

// 上面的泛化 copy 构造函数并未被声明为 explicit，那是蓄意的，因为原始指针
// 类型之间的转换（例如从 derived class 指针转为 base class 指针）是隐式转换，
// 无需明白写出转型动作（cast），所以让智能指针仿效这种行径也属合理。在模板化
// 构造函数（templatized constructor）中略去 explicit 就是为了这个目的。





// 完成声明之后，这个为 SmartPtr 而写的“泛化 copy 构造函数”提供的东西比我们需要的更多。
// 是的，我们希望根据一个 SmartPtr<Bottom> 创建一个 SmartPtr<Top>，
// 却不希望根据一个 SmartPtr<Top> 创建一个 SmartPtr<Bottom>，
// 因为那对 public 继承而言（见条款 32）是矛盾的。我们也不希望根据一个
// SmartPtr<double> 创建一个 SmartPtr<int>，因为现实中并没有“将 int* 转换为 double*”
// 的对应隐式转换行为。是的，我们必须从某方面对这一 member template
// 所创建的成员函数群进行拣选或筛除。

// 假设 SmartPtr 遵循 auto_ptr 和 tr1::shared_ptr 所以提供的榜样，
// 也提供一个 get 成员函数，返回智能指针对象（见条款 15）所持有的那个原始指针的副本，
// 那么我们可以在“构造模板”实现代码中约束转换行为，使它符合我们的期望：
template<typename T>
class SmartPtr {
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other)      // 以 other 的 heldPtr
        : heldPtr(other.get()) { ... }      // 初始化 this 的 heldPtr
    T* get() const { return heldPtr; }
    ...
private:
    T* heldPtr;                             // 这个 SmartPtr 持有的内置（原始）指针
};

// 我使用成员初值列（member initialization list）来初始化 SmartPtr<T> 之内类型为
// T* 的成员变量，并以类型为 U* 的指针（由 SmartPtr<U> 持有）作为初值。
// 这个行为只有当“存在某个隐式转换可将一个 U* 指针转为一个 T* 指针”时才能通过编译，
// 而那正是我们想要的。最终效益是 SmartPtr<T> 现在有了一个泛化 copy 构造函数，
// 这个构造函数只在其所获得的实参隶属适当（兼容）类型时才通过编译。

// member function templates（成员函数模板）的效用不限于构造函数，
// 它们常扮演的另一个角色是支持赋值操作。例如 TR1 的 shared_ptr
// （见条款 13）支持所有“来自兼容之内置指针、tr1::shared_ptrs、auto_ptrs 和
// tr1::weak_ptrs（见条款 54）”的构造行为，
// 以及所有来自上述各物（tr1::weak_ptrs 除外）的赋值操作。
template<class T>
class shared_ptr {
public:
    template<class Y>                                   // 构造，来自任何兼容的
        explicit shared_ptr(Y* p);                      // 内置指针、
    template<class Y>
        shared_ptr(shared_ptr<Y> const& r);             // 或 shared_ptr、
    template<class Y>
        explicit shared_ptr(weak_ptr<Y> const& r);      // 或weak_ptr、
    template<class Y>
        explicit shared_ptr(auto_ptr<Y>& r);            // 或 auto_ptr。
    template<class Y>                                   // 赋值，来自任何兼容的
        shared_ptr& operator= (shared_ptr<Y> const& r); // shared_ptr、
    template<class Y>                                   // 或 auto_ptr。
        shared_ptr& operator= (auto_ptr<Y>& r);
};





// 下面是 tr1::shared_ptr 的一份定义摘要，例证上述所言：
template<class T>
class shared_ptr {
public:
    shared_ptr(shared_ptr const& r);                // copy 构造函数
    
    template<class Y>                               // 泛化 copy 构造函数
    shared_ptr(shared_ptr<Y> const& r);

    shared_ptr& operator= (shared_ptr const& r);    // copy assignment.

    template<class Y>                               // 泛化 copy assignment.
    shared_ptr& operator= (shared_ptr<Y> const& r);
    ...
};
```

<br>

### 条款 46：需要类型转换时请为模板定义非成员函数

- 当我们编写一个 class template，而它所提供之“与此 template 相关的”函数支持“所有参数之隐式类型转换”时，请将那些函数定义为“class template 内部的 friend 函数”。

``` cpp {.line-numbers}
template<typename T>
class Rational {
public:
    Rational(const T& numerator = 0,        // 条款 20 告诉你为什么参数以
             const T& denominator = 1);     // passed by reference 方式传递，
    const T numerator() const;              // 条款 28 告诉你为什么返回值
    const T denominator() const;            // 以 passed by value 方式传递。
    ...                                     // 条款 3 告诉你为什么它们是 const。
};

template<typename T>
const Rational<T> operator* (const Rational<T>& lhs,
                             const Rational<T>& rhs)
{ ... }

// 就像条款 24 一样，我们希望支持混合式（mixed-mode）算术运算，
// 所以我们希望以下代码顺利通过编译。我们也预期它会，因为它正式条款 24 所列的同一份代码，
// 唯一不同的是 Rational 和 operator* 如今都成了 templates：
Rational<int> oneHalf(1, 2);            // 这个例子来自条款 24，
                                        // 唯一不同的是 Rational 改为 template。
Rational<int> result = oneHalf * 2;     // 错误！无法通过编译。

// 上述失败给我们的启示是，模板化的 Rational 内的某些东西似乎和其 non-template 版本不同。
// 事实的确如此。在条款 24 内，编译器知道我们尝试调用什么函数（就是接受两个
// Rationals 参数的那个 operator* 啦），但这里编译器不知道我们想要调用哪个函数。
// 取而代之的是，它们试图想出什么函数被名为 operator* 的 template 具现化
// （产生）出来。它们知道它们应该可以具现化某个“名为 operator* 并接受两个
// Rational<T> 参数”的函数，但为完成这一具现化行动，必须先算出 T 是什么。
// 问题是它们没有这个能耐。

// 只要利用一个事实，我们就可以缓和编译器在 template 实参推导方面受到的挑战：
// template class 内的 friend 声明式可以指涉某个特定函数。那意味 class Rational<T>
// 可以声明 operator* 是它的一个 friend 函数。Class templates 并不倚赖
// template 实参推导（后者只施行于 function templates 身上），
// 所以编译器总是能够在 class Rational<T> 具现化时得知 T。因此，
// 令 Rational<T> class 声明适当的 operator* 为其 friend 函数，可简化整个问题：
template<typename T>
class Rational {
public:
    ...
    friend const Rational operator* (const Rational& lhs,   // 声明 operator* 函数，
                                     const Rational& rhs);  // 细节详下。
};

template<typename T>                                // 定义
const Rational<T> operator* (const Rational<T>& lhs,   // operator* 函数。
                             const Rational<T>& rhs)
{ ... }

// 现在对 operator* 的混合式调用可以通过编译了，因为当对象 oneHalf 被声明为
// 一个 Rational<int>，class Rational<int> 于是被具现化出来，而作为过程的一部分，
// friend 函数 operator*（接受 Rational<int> 参数）也就被自动声明出来。
// 后者身为一个函数而非函数模板（function template），因此编译器可在调用
// 它时使用隐式转换函数（例如 Rational 的 non-explicit 构造函数），
// 而这便是混合式调用之所以成功的原因。





// 但是，此情境下的“成功”是个有趣的字眼，因为虽然这段代码通过编译，却无法连接。
// 稍后我马上回来处理这个问题，首先我要谈谈在 Rational 内声明 operator* 的语法。

// 在一个 class template 内，template 名称可被用来作为“template 和其参数”的
// 简略表达方式，所以在 Rational<T> 内我们可以只写 Rational 而不必写 Rational<T>。
// 本例中的 operator* 被声明为接收并返回 Rationals（而非 Rational<T>s）。
// 如果它被声明如下，一样有效：
template<typename T>
class Rational {
public:
    ...
    friend const Rational<T> operator* (const Rational<T>& lhs,
                                        const Rational<T>& rhs);
    ...
};





// 我们意图令此 class 外部的 operator* template 提供定义，但是行不通——如果我们自己
// 声明了一个函数（那正是 Rational template 内的作为），就有责任定义那个函数。
// 既然我们没有提供定义式，连接器当然找不到它！

// 或许最简单的可行办法就是将 operator* 函数本体合并至其声明式内：
template<typename T>
class Rational {
public:
    ...
    friend const Rational operator* (const Rational& lhs,
                                     const Rational& rhs)
    {
        return Rational(lhs.numerator() * rhs.numerator(),          // 实现码与
                        lhs.denominator() * rhs.denominator());     // 条款 24 同
    }
};

// “Rational 是个 template”这一事实意味上述的辅助函数通常也是个 template，
// 所以定义了 Rational 的头文件代码，很典型地长这个样子：
template<typename T> class Rational;        // 声明 Rational template
template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs,
                             const Rational<T>& rhs);

template<typename T>
class Rational {
public:
    ...
    friend const Rational<T> operator* (const Rational<T>& lhs,
                                        const Rational<T>& rhs)
    { return doMultiply(lhs, rhs); }        // 令 friend 调用 helper
    ...
};

// 许多编译器实质上会强迫你把所有 template 定义式放进头文件内，
// 所以你或许需要在头文件内定义 doMultiply（一如条款 30 所言，
// 这样的 templates 不需非得是 inline 不可），看起来像这样：
template<typename T>                                        // 若有必要，
const Rational<T> doMultiply(const Rational<T>& lhs,        //  在头文件内定义
                             const Rational<T>& rhs)        // helper template
{
    return Rational<T>(lhs.numerator() * rhs.numerator(),
                       lhs.denominator() * rhs.denominator());

}
```

<br>

### 条款 47：请使用 traits classes 表现类型信息

- Traits classes 使得“类型相关信息”在编译期可用。他们以 templates 和“templates 特化”完成实现。
- 整合重载技术（overloading）后，traits classes 有可能在编译期对类型执行 if...else 测试。

``` cpp {.line-numbers}
// STL 主要由“用以表现容器、迭代器和算法”的 templates 构成，
// 但也覆盖若干工具性 templates，其中一个名为 advance，
// 用来将某个迭代器移动某个给定距离：
template<typename IterT, typename, DistT>       // 将迭代器向前移动 d 单位。
void advance(IterT& iter, DistT d);             // 如果 d < 0 则向后移动。





// 对于这 5 种分类，C++ 标准程序库分别提供专属的卷标结构（tag struct）加以确认：
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};





// 我们知道 random access 迭代器支持迭代器算术运算，只耗费常量时间，
// 因此如果面对这种迭代器，我们希望运用其优势。
// 我们真正希望的是以这种方式实现 advance：
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    if (iter is a random access iterator) {
        iter += d;          // 针对 random access 迭代器使用迭代器算术运算
    }
    else {
        if (d >= 0) { while (d--) ++iter; }     // 针对其他迭代器分类
        else { while (d++) --iter; }            // 反复调用 ++ 或 --
    }
}





template<typename IterT>        // template，用来处理
struct iterator_traits;         // 迭代器分类的相关信息

template< ... >
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        ...
    };
    ...
};

// list 的迭代器可双向行进，所以它们应该是这样：
template< ... >
class list {
public:
    class iterator {
    public:
        typedef bidirectional_iterator_tag iterator_category;
        ...
    };
    ...
};

// 至于 iterator_traits，只是鹦鹉学舌般地响应 iterator class 的嵌套式 typedef：

// 类型 IterT 的 iterator_category 其实就是用来表现“IterT 说它自己是什么”。
// 关于 “typedef typename” 的运用，见条款 42。
template<typename IterT>
struct iterator_traits {
    typedef typename IterT::iterator_category iterator_category;
    ...
};





// 这对用户自定义类型行得通，但对指针（也是一种迭代器）行不通，
// 因为指针不可能嵌套 typedef。iterator_traits 的第二部分如下，专门用来对付指针。

// 为了支持指针迭代器，iterator_traits 特别针对指针类型提供一个偏特化版本
// （partial template specialization）。由于指针的行径与 random access
// 迭代器类似，所以 iterator_traits 为指针指定的迭代器类型是：
template<typename IterT>            // template 偏特化
struct iterator_traits<IterT*>      // 针对内置指针
{
    typedef random_access_iterator_tag iterator_category;
    ...
}





// 现在，你应该知道如何设计并实现一个 traits class 了：
// （1）确认若干你希望将来可取得的类型相关信息。
// 例如对迭代器而言，我们希望将来得其分类（category）。
// （2）为该信息选择一个名称（例如 iterator_category）。
// （3）提供一个 template 和一组特化版本（例如稍早说的 iterator_traits），
// 内含你希望支持的类型相关信息。

template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    if (typeid(typename std::iterator_traits<IterT>::iterator_category)
        == typeid(std::random_access_iterator_tag))
    ...
}

// 虽然这看起来前景光明，其实并非我们想要。首先它会导致编译问题，
// 但我将在条款 48 才探讨这一点，此刻有更根本的问题要考虑。
// IterT 类型在编译期间获知，所以 iterator_traits<IterT>::iterator_category
// 也可在编译期间确定。但 if 语句却是在运行期才会核定。
// 为什么将可在编译期完成的事延到运行期才做呢？
// 这不仅浪费时间，也造成可执行文件膨胀。

// 我们真正想要的是一个条件式（也就是一个 if...else 语句）判断“编译期核定成功”之类型。
// 恰巧 C++ 有一个取得这种行为的办法，那就是重载（overloading）。

// 为了让 advance 的行为如我们所期望，我们需要做的是产生两版重载函数，
// 内含 advance 的本质内容，但各自接受不同类型的 iterator_category 对象。
// 我将这两个函数取名为 doAdvance：
template<typename IterT, typename DistT>        // 这份实现用于
void doAdvance(IterT& iter, DistT d,            // random access
               std::random_access_iterator_tag) // 迭代器
{
    iter += d;
}

template<typename IterT, typename DistT>        // 这份实现用于
void doAdvance(IterT& iter, DistT d,            // bidirectional
               std::bidirectional_iterator_tag) // 迭代器
{
    if (d >= 0) { while (d--) ++iter; }
    else { while (d++) --iter; }
}

template<typename IterT, typename DistT>        // 这份实现用于
void doAdvance(IterT& iter, DistT d,            // input
               std::input_iterator_tag)         // 迭代器
{
    if (d < 0) {
        throw std::out_of_range("Negative distance");   // 详下
    }
    while (d--) ++iter;
}
// advance 函数规范说，如果面对的是 random access 和 bidirectional 迭代器，
// 则接收正距离和负距离；但如果面对的是 forward 或 input 迭代器，
// 则移动负距离会导致不明确（未定义）行为。我所检验过的实现码都假设 d 不为负，
// 于是直接进入一个冗长的循环迭代，等待计数器降为 0。上述代码中我以抛出异常取而代之。
// 两种做法都有根据，但“无法预言发生何事”是“不明确行为”之祸源所在。

// 有了这些 doAdvance 重载版本，advance 需要做的只是调用它们并额外传递一个对象，
// 后者必须带有适当的迭代器分类。于是编译器运用重载解析机制（overloading resolution）
// 调用适当的实现代码：
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
}





// 现在我们可以总结如何使用一个 traits class 了：
// （1）建立一组重载函数（身份像劳工）或函数模板（例如 doAdvance），
// 彼此间的差异只在于各自的 traits 参数。令每个函数实现码与其接受之 traits 信息相应和。
// （2）建立一个控制函数（身份像工头）或函数模板（例如 advance），它调用上述那些“劳工函数”并传递 traits class 所提供的信息。
```

<br>

### 条款 48：认识 template 元编程

- Template metaprogramming（TMP，模板元编程）可将工作由运行期移往编译期，因而得以实现早期错误检测和更高的执行效率。
- TMP 可被用来生成“基于政策选择组合”（based on combinations of policy choices）的客户定制代码，也可用来避免生成对某些特殊类型并不适合的代码。

``` cpp {.line-numbers}
// Template metaprogramming（TMP，模板元编程）是编写 template-based C++
// 程序并执行与编译器的过程。所谓 template metaprogram（模板元程序）
// 是以 C++ 写成、执行于 C++ 编译器内的程序。一旦 TMP 程序结束执行，
// 其输出，也就是从 templates 具现出来的若干 C++ 源码，便会一如往常地被编译。

// TMP 有两个伟大的效力。第一，它让某些事情更容易。
// 如果没有它，那些事情将是困难的，甚至是不可能的。
// 第二，由于 template metaprograms 执行于 C++ 编译期，因此可将工作从运行期转移到编译期。





template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    if (iter is a random access iterator) {
        iter += d;          // 针对 random access 迭代器使用迭代器算术运算
    }
    else {
        if (d >= 0) { while (d--) ++iter; }     // 针对其他迭代器类型
        else { while (d++) --iter; }            // 反复调用 ++ 或 --
    }
}

// 我们可以使用 typeid 让其中的伪码成真，取得 C++ 对此问题的一个“正常”解决
// 方案——所有工作都在运行期进行：
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    if (typeid(typename std::iterator_traits<IterT>::iterator_category)
        == typeid(std::random_access_iterator_tag)) {
        iter += d;
    }
    else {
        if (d >= 0) { while (d--) ++iter; }     // 针对其他迭代器分类
        else { while (d++) --iter; }            // 反复调用 ++ 或 --
    }
}

// 条款 47 指出，这个 typeid-based 解法的效率比 traits 解法低，因此在方案中，
// （1）类型测试发生于运行期而非编译期，（2）“运行期类型测试”代码会出现在
// （或说被连接于）可执行文件中。实际上这个例子正可彰显 TMP 如何能够比“正常的”
// C++ 程序更高效，
// 因为 traits 解法就是 TMP。别忘了，traits 引发“编译期发生于类型身上的 if...else 计算”。

// 条款 47 曾经提过 advance 的 typeid-based 实现方式可能导致编译期问题，下面就是个例子：
std::list<int>::iterator iter;
...
advance(iter, 10);              // 移动 iter 向前走 10 个元素；
                                // 上述实现无法通过编译。

// 下满这一版 advance 便是针对上述调用而产生的。
// 将 template 参数 IterT 和 DistT 分别替换为 iter 和 10 的类型之后，我们得到这些：
void advance(std::list<int>::iterator& iter, int d)
{
    if (typeid(std::iterator_traits<std::list<int>::iterator>::iterator_category)
        == typeid(std::random_access_iterator_tag)) {
        iter += d;      // 错误！
    }
    else {
        if (d >= 0) { while (d--) ++iter; }
        else { while (d++) --iter; }
    }
}

// 问题出在我所强调的那一行代码使用了 += 操作符，那便是尝试在一个 list<int>::iterator
// 身上使用 +=，但 list<int>::iterator 是 bidirectional 迭代器（见条款 47），
// 并不支持 +=。只有 random access 迭代器才支持 +=。
// 与此对比的是 traits-based TMP 解法，其针对不同类型而进行的代码，被拆分为不同的函数，
// 每个函数所使用的操作（操作符）都可施行于该函数所对付的类型。





// TMP 的阶乘运算示范如何通过“递归模板具现化”（recursive template instantiation）
// 实现循环，以及如何在 TMP 中创建和使用变量：
template<unsigned n>                // 一般情况：Factorial<n> 的值是
struct Factorial {                  // n 乘以 Factorial<n - 1> 的值
    enum { value = n * Factorial<n - 1>::value };
};

template<>                          // 特殊情况
struct Factorial<0> {               // Factorial<0> 的值是 1
    enum { value = 1 };
};

// 你可以这样使用 Factorial：
int main()
{
    std::cout << Factorial<5>::value;       // 印出 120
    std::cout << Factorial<10>::value;      // 印出 3628800
}
```

<br>

## 8、定制 new 和 delete

<br>

### 条款 49：了解 new-handler 的行为

- set_new_handler 允许客户指定一个函数，在内存分配无法获得满足时被调用。
- Nothrow new 是一个颇为局限的工具，因为它只适用于内存分配；后继的构造函数调用还是可能抛出异常。

``` cpp {.line-numbers}
// 为了指定这个“用以处理内存不足”的函数，客户必须调用 set_new_handler，
// 那是声明于 <new> 的一个标准程序库函数：
namespace std {
    typedef void (*new_handler) ();
    new_handler set_new_handler(new_handler p) throw();
}

// new_handler 是个 typedef，定义出一个指针指向函数，该函数没有参数也不返回任何东西。
// set_new_handler 则是“获得一个 new_handler 并返回一个 new_handler”的函数。
// set_new_handler 声明式尾端的 “throw()” 是一份异常明细，
// 表示该函数不抛出任何异常——虽然事实更有趣些，详见条款 29.

// set_new_handler 的参数是个指针，指向 operator new 无法分配足够内存时
// 该被调用的函数。其返回值也是个指针，指向 set_new_handler 被调用前正在执行
// （但马上就要被替换）的那个 new-handler 函数。

// 你可以这样使用 set_new_handler：

// 以下是当 operator new 无法分配足够内存时，该被调用的函数
void outOfMem()
{
    std::cerr << "Unable to satisfy request for memory\n";
    std::abort();
}

int main()
{
    std::set_new_handler(outOfMem);
    int* pBigDataArray = new int[100000000L];
}

// 就本例而言，如果 operator new 无法为 100000000 个整数分配足够空间，
// outOfMem 会被调用，于是程序在发出一个信息之后夭折（abort）。





// 当 operator new 无法满足内存申请时，它会不断调用 new-handler 函数，
// 直到找到足够内存。引起反复调用的代码显示于条款 51，这里的高级描述已足够获得一个结论，
// 那就是一个设计良好的 new-handler 函数必须做以下事情：
// （1）让更多内存可被使用。
// （2）安装另一个 new-handler。
// （3）卸载 new-handler。
// （4）抛出 bad_alloc（或派生自 bad_alloc）的异常。
// （5）不返回，通常调用 abort 或 exit。





// 有时候你或许希望以不同的方式处理内存分配失败情况，你希望视被分配物属于哪个 class 而定：
class X {
public:
    static void outOfMemory();
    ...
};

class Y {
public:
    static void outOfMemory();
    ...
};

X* p1 = new X;      // 如果分配不成功，
                    // 调用 X::outOfMemory
Y* p2 = new Y;      // 如果分配不成功，
                    // 调用 Y::outOfMemory





// 现在，假设你打算处理 Widget class 的内存分配失败情况。首先你必须
// 登录“当 operator new 无法为一个 Widget 对象分配足够内存时”调用函数，
// 所以你需要声明一个类型为 new_handler 的 static 成员，用以指向 class Widget
// 的 new-handler。看起像这样：
class Widget {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler currentHandler;
};

// Static 成员必须在 class 定义式之外被定义（除非它们是 const 而且是整数形，见条款 2）
// 所以需要这么写：
std::new_handler Widget::currentHandler = 0;    // 在 class 实现文件内初始化为 null

// Widget 内的 set_new_handler 函数会将它获得的指针存储起来，
// 然后返回先前（在此调用之前）存储的指针，这也正是标准版 set_new_handler 的作为：
std::new_handler Widget::set_new_handler(std::new_handler p) throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

// 最后，Widget 的 operator new 做以下事情：
// （1）调用标准 set_new_handler，告知 Widget 的错误处理函数。
// （2）调用 global operator new，执行实际之内存分配。
// （3）如果 global operator new 能够分配足够一个 Widget 对象所用的内存，
// Widget 的 operator new 会返回一个指针，指向分配所得。

// 下面以 C++ 代码再阐述一次。我将从资源处理类（resource-handling class）开始，
// 那里面只是基础性 RAII 操作，在构造过程中获得一笔资源，并在析构过程中释还（见条款 13）：
class NewHandlerHolder {
public:
    explicit NewHandlerHolder(std::new_handler nh)      // 取得目前的
        : handler(nh) {}                                // new-handler。
    
    ~NewHandlerHolder()                                 // 释放它
    { std::set_new_handler(handler); }

private:
    std::new_handler handler;                           // 记录下来

    NewHandlerHolder(const NewHandlerHolder&);          // 阻止 copying
    NewHandlerHolder& operator= (const NewHandlerHolder&);  // （见条款 14）
};

// 这就使得 Widget's operator new 的实现相当简单：
void* Widget::operator new(std::size_t size) throw(std::bad_alloc)
{
    NewHandlerHolder                                // 安装 Widget 的
        h(std::set_new_handler(currentHandler));    // new-handler.
    return ::operator new(size);                    // 分配内存或抛出异常。
}                                                   // 恢复 global new-handler。

// Widget 的客户应该类似这样使用其 new-handling：
void outOfMem();                        // 函数声明。此函数在
                                        // Widget 对象分配失败时被调用。

Widget::set_new_handler(outOfMem);      // 设定 outOfMem 为 Widget 的
                                        // new-handling 函数。

Widget* pw1 = new Widget;               // 如果内存分配失败，
                                        // 调用 outOfMem。

std::string* ps = new std::string;      // 如果内存分配失败，
                                        // 调用 global new-handling 函数
                                        // （如果有的话）。

Widget::set_new_handler(0);             // 设定 Widget 专属的
                                        // new-handling 函数为 null。

Widget* pw2 = new Widget;               // 如果内存分配失败，
                                        // 立刻抛出异常。
                                        // （class Widget 并没有专属的
                                        // new-handling 函数）。





// 实现这一方案的代码并不因 class 的不同而不同，因此在它处加以复用是个合理的构想。
// 一个简单的做法就是建立起一个“mixin”风格的 base class，这种 base class 用来
// 允许 derived classes 继承单一特定能力——在本例中是“设定 class 专属之
// new-handler”的能力。然后将这个 base class 转换为 template，
// 如此一来每个 derived class 将获得实体互异的 class data 复件。

// 这个设计的 base class 部分让 derived classes 继承它们所需的 set_new_handler
// 和 operator new，而 template 部分则确保每一个 derived class 获得一个实体
// 互异的 currentHandler 成员变量。

template<typename T>            // "mixin" 风格的 base class，用以支持
class NewHandlerSupport {       // class 专属的 set_new_handler
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...                         // 其他的 operator new 版本——见条款 52
private:
    static std::new_handler currentHandler;
};

template<typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

template<typename T>
void* NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc)
{
    NewHandlerHolder h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}

// 以下将每一个 currentHandler 初始化为 null
template<typename T>
std::new_handler NewHandlerSupport<T>::currentHandler = 0;

// 有了这个 class template，为 Widget 添加 set_new_handler 支持能力就轻而易举了：
// 只要令 Widget 继承自 NewHandlerSupport<Widget> 就好，像下面这样。
class Widget : public NewHandlerSupport<Widget> {
    ...             // 和先前一样，但不必声明
};                  // set_new_handler 或 operator new





// C++ 标准委员会不想抛弃那些“侦测 null”的族群，于是提供另一形式的 operator new，
// 负责供应传统的“分配失败便返回 null”行为。这个形式被称为“nothrow”形式——
// 某种程度上是因为他们在 new 的使用场合用了 nothrow 对象（定义于头文件 <new>）：
class Widget { ... };
Widget* pw1 = new Widget;                   // 如果分配失败，抛出 bad_alloc.
if (pw1 == 0) ...                           // 这个测试一定失败。
Widget& pw2 = new (std::nothrow) Widget;    // 如果分配 Widget 失败，返回 0。
if (pw2 == 0) ...                           // 这个测试可能成功

// Nothrow new 对异常的强制保证性并不高。
// 结论是：使用 nothrow new 只能保证 operator new 不抛掷异常，
// 不保证像“new (std::nothrow) Widget” 这样的表达式绝不导致异常。
// 因此你其实没有运用 nothrow new 的需要。
```

<br>

### 条款 50：了解 new 和 delete 的合理替换时机

- 有许多理由需要写个自定的 new 和 delete，包括改善效能、对 heap 运用错误进行调试、收集 heap 使用信息。

``` cpp {.line-numebrs}
// 首先，怎么会有人想要替换编译器提供的 operator new 或 operator delete 呢？
// 下面是三个最常见的理由：
// （1）用来检测运用上的错误。
// （2）为了强化效能。
// （3）为了收集使用上的统计数据。





// 观念上，写一个定制 operator new 十分简单。举个例子，下面是个快速发展
// 得出的初阶段 global operator new，促进并协助检测“overruns”（写入点在分配区块尾端之后）
// 或“underruns”（写入点在分配区块起点之前）。其中还存在不少小错误，稍后我会完善它。
static const int signature = 0xDEADBEEF;
typedef unsigned char Byte;
// 这段代码代码还有若干小错误，详下。
void* operator new (std::size_t size) throw(std::bad_alloc)
{
    using namespace std;
    size_t realSize = size + 2 * sizeof(int);       // 增加大小，使能够
                                                    // 塞入两个 signatures。
    void* pMem = malloc(realSize);
    if (!pMem) throw bad_alloc();

    // 将 signature 写入内存的最前段落和最后段落。
    *(static_cast<int*>(pMem)) = signature;
    *(reinterpret_cast<int*>(static_cast<Byte*>(pMem) + realSize - sizeof(int))) = signature;

    // 返回指针，指向恰位于第一个 signature 之后的内存位置。
    return static_cast<Byte*>(pMem) + sizeof(int);
}

// 条款 51 说所以 operator news 都应该内含一个循环，反复调用某个 new-handling 函数，这里却没有。






// 本条款的主题是，了解何时可在“全局性的”或“class 专属的”基础上合理替换
// 缺省的 new 和 delete。
// （1）为了检测运用错误。
// （2）为了收集动态分配内存之使用统计信息。
// （3）为了增加分配和归还的速度。
// （4）为了降低缺省内存管理器带来的空间额外开销。
// （5）为了弥补缺省分配器中的非最佳齐位（suboptimal alignment）。
// （6）为了将相关对象成簇集中。
// （7）为了获得非传统的行为。
```

<br>

### 条款 51：编写 new 和 delete 时需固守常规

- operator new 应该内含一个无穷循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用 new-handler。它也应该有能力处理 0 bytes 申请。Class 专属版本则还应该处理 “比正确大小更大的（错误）申请”。
- operator delete 应该在收到 null 指针时不做任何事。Class 专属版本则还应该处理“比正确大小更大的（错误申请）”。

```cpp {.line-numbers}
// C++ 规定，即使客户要求 0 bytes，operator new 也得返回一个合法指针。
// 这种看似诡异的行为其实是为了简化语言其他部分。
// 下面是个 non-member operator new 伪码（pseudocode）：
void* operator new(std::size_t size) throw(std::bad_alloc)
{                               // 你的 operator new 可能接受额外参数
    using namespace std;
    if (size == 0) {            // 处理 0-byte 申请
        size = 1;               // 将它视为 1-byte 申请。
    }
    while (true) {
        尝试分配 size bytes;
        if (分配成功)
        return (一个指针，指向分配得来的内存);

        // 分配失败；找出目前的 new-handling 函数（见下）
        new_handler globalHandler = set_new_handler(0);
        set_new_handler(globalHandler);

        if (globalHandler) (*globalHandler)();
        else throw std::bad_alloc();
    }
}





// 针对 class X 而设计的 operator new，其行为很典型地只为大小刚好
// 为 sizeof(X) 的对象而设计。然而一旦被继承下去，有可能 base class
// 的 operator new 被调用用以分配 derived class 对象：
class Base {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...
};

class Derived : public Base     // 假设 Derived 未声明 operator new
{ ... };
Derived* p = new Derived;       // 这里调用的是 Base::operator new

// 如果 Base class 专属的 operator new 并非被设计用来对付上述情况（实际上往往如此），
// 处理此情势的最佳做法是将“内存申请量错误”的调用行为改采标准 operator new，像这样：
void* Base::operator new(std::size_t size) throw(std::bad_alloc)
{
    if (size != sizeof(Base))           // 如果大小错误，
        return ::operator new(size);    // 令标准的 operator new 起而处理。
    ...                                 // 否则在这里处理。
}

// operator delete 情况更简单，你需要记住的唯一事情就是 C++ 保证“删除 null 指针永远安全”，
// 所以你必须兑现这项保证。下面是 non-member operator delete 的伪码（pseudocode）：
void operator delete(void* rawMemory) throw()
{
    if(rawMemory == 0) return;          // 如果将被删除的是个 null 指针，
                                        // 那就什么都不做。
    现在，归还 rawMemory 所指的内存;
}

// 这个函数的 member 版本也很简单，只需要多加一个动作检查器删除数量。
// 万一你的 class 专属的 operator new 将大小有误的分配行为转交::operator new 执行，
// 你也必须将大小有误的删除行为转交 ::operator delete 执行：
class Base {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    static void operator delete(void* rawMemory, std::size_t size) throw();
    ...
};

void Base::operator delete(void* rawMemory, std::size_t size) throw()
{
    if (rawMemory == 0) return;         // 检查 null 指针。
    if (size != sizeof(Base)) {         // 如果大小错误，令标准版
        ::operator delete(rawMemory);   // operator delete 处理此一申请。
        return;
    }
    现在，归还 rawMemory 所指的内存;
    return;
}

// 有趣的是，如果即将被删除的对象派生自某个 base class 而后者欠缺 virtual 析构函数，
// 那么 C++ 传给 operator delete 的 size_t 数值可能不正确。
// 这是“让你的 base classes 拥有 virtual 析构函数”的一个够好的理由：
// 条款 7还提过一个更好的理由。此刻只要你提高警觉，如果你的 base classes
// 遗漏 virtual 析构函数，operator delete 可能无法正确工作。
```

<br>

### 条款 52：写了 placement new 也要写 placement delete

- 当你写一个 placement operator new，请确定也写出了对应的 placement operator delete。如果没有这样做，你的程序可能会发生隐微而时断时续的内存泄漏。
- 当你声明 placement new 和 placement delete，请确定不要无意识（非故意）地遮掩了它们的正常版本。

``` cpp {.line-numbers}
// 当你写一个 new 表达式像这样：
Widget* pw = new Widget;
// 共有两个函数被调用：一个是用以分配内存的 operator new，一个是 Widget 的 default 构造函数。

// 假设其中第一个函数调用成功，第二个函数却抛出异常。既然这样，
// 步骤一的内存分配所得必须取消并恢复旧观，否则会造成内存泄漏（memory leak）。
// 在这个时候，客户没有能力归还内存，因为如果 Widget 构造函数抛出异常，
// pw 尚未被赋值，客户手上也就没有指针指向该被归还的内存。
// 取消步骤一并恢复旧观的责任因此落到 C++ 运行期系统身上。

// 运行期系统会高高兴兴地调用步骤一所调用的 operator new 的相应 operator
// delete 版本，前提当然是它必须知道哪一个（因为可能有许多个）operator delete
// 该被调用。如果目前面对的是拥有正确签名式（signature）的 new 和 delete，
// 这并不是问题，因为正常的 operator new：
void* operator new(std::size_t) throw(std::bad_alloc);

// 对应于正常的 operator delete：
void operator delete(void* rawMemory) throw();                      // global 作用域中的正常签名式
void operator delete(vodi* rawMemory, std::size_t size) throw();    // class 作用域中典型的签名式。





// 然而当你开始声明非正常形式的 operator new，也就是带有附加参数的 operator new，
// “究竟哪一个 delete 伴随这个 new”的问题便浮现了。

// 举个例子，假设你写了一个 class 专属的 operator new，要求接受一个 ostream，
// 用来志记（logged）相关分配信息，同时又写了一个正常形式的 class 专属 operator delete：
class Widget {
public:
    ...
    static void* operator new(std::size_t size, std::ostream& logStream)
        throw(std::bad_alloc);      // 非正常形式的 new
    
    static void operator delete(void* pMemory, std::size_t size)
        throw();                    // 正常的 class 专属 delete
    ...
};

// 这个设计有问题。

// 如果 operator new 接受的参数除了一定会有的那个 size_t 之外还有其他，
// 这便是个所谓的 placement new。因此，上述的 operator new 是个 placement 版本。
// 众多 placement new 版本中特别有用的一个是“接受一个指针指向对象该被构造之处”，
// 那样的 operator new 长相如下：
void* operator new(std::size_t, void* pMemory) throw();     // placement new





Widget* pw = new (std::cerr) Widget;    // 调用 operator new 并传递 cerr 为其
                                        // ostream 实参；这个动作会在 Widget
                                        // 构造函数抛出异常时泄漏内存
// 既然这里的 operator new 接受类型为 ostream& 的额外实参，
// 所以对应的 operator delete 就应该是：
void operator delete(void*, std::ostream&) throw();

// 类似于 new 的 placement 版本，operator delete 如果接受额外参数，便称为 placement deletes。





// 规则很简单：如果一个带额外参数的 operator new 没有“带相同额外参数”的对应版 operator delete，
// 那么当 new 的内存分配动作需要取消并恢复旧观时就没有任何 operator delete
// 会被调用。因此，为了消弭稍早代码中的内存泄漏，Widget 有必要
// 声明一个 placement delete，对应于那个有志记功能（logging）的 placement new：
class Widget {
public:
    ...
    static void* operator new(std::size_t size, std::ostream& logStream)
        throw(std::bad_alloc);
    static void operator delete(void* pMemory) throw();
    static void operator delete(void* pMemory, std::ostream& logStream)
        throw();
    ...
};

// 这样改变之后，如果以下语句引发 Widget 构造函数抛出异常：
Widget* pw = new (std::cerr) Widget;        // 一如以往，但这次不再泄漏
// 对应的 placement delete 会被自动调用，让 Widget 有机会确保不泄漏任何内存。





// 然而如果没有抛出异常（通常如此），而客户代码中有个对应的 delete，会发生什么事：
delete pw;          // 调用正常的 operator delete

// 就如上一行注释所言，调用的是正常形式的 operator delete，而非其 placement 版本。
// placement delete 只有在“伴随 placement new 调用而触发的构造函数”出现异常时
// 才会被调用。对着一个指针（例如上述的 pw）施行 delete 绝不会导致调用 placement delete。





// 附带一提，由于成员函数的名称会掩盖其外围作用域中的相同名称（见条款 33），
// 你必须小心避免让 class 专属的 news 掩盖客户期望的其他 news（包括正常版本）。
// 假设你有一个 base class，其中声明唯一一个 placement operator new，
// 客户端会发送他们无法使用正常形式的 new：
class Base {
public:
    ...
    static void operator new(std::size_t size,
                             std::ostream& logStream)
        throw(std::bad_alloc);      // 这个 new 会遮掩正常的 global 形式
    ...
};

Base* pb = new Base;                // 错误！因为正常形式的 operator new 被掩盖。
Base* pb = new (std::cerr) Base;    // 正确，调用 Base 的 placement new。

// 同样道理，derived classes 中的 operator news 会掩盖 global 版本
// 和继承而得的 operator new 版本：
class Derived : public Base {                       // 继承自先前的 Base
public:
    ...
    static void* operator new(std::size_t size)     // 重新声明正常形式的
        throw(std::bad_alloc);                      // 形式的 new
    ...
};

Derived* pd = new (std::clog) Derived;      // 错误！因为 Base 的
                                            // placement new 被掩盖了。
Derived* pd = new Derived;                  // 没问题，调用 Derived 的
                                            // operator new。





// 条款 33 更详细地讨论了这种名称遮掩问题。对于撰写内存分配函数，你需要记住的是，
// 缺省情况下 C++ 在 global 作用域内提供以下形式的 operator new：
void* operator new(std::size_t) throw(std::bad_alloc);            // normal new.
void* operator new(std::size_t, void*) throw();                   // placement new.
void* operator new(std::size_t, const std::nothrow_t&) throw();   // nothrow new，见条款 49.





// 如果你在 class 内声明任何 operator news，它会遮掩上述这些标准形式。
// 除非你的意思就是要阻止 class 的客户使用这些形式，否则请确保它们在你
// 所生成的任何定制型 operator new 之外还可用。对于每一个可用的 operator new
// 也请确定提供对应的 operator delete。如果你希望这些函数有着平常的行为，
// 只要令你的 class 专属版本调用 global 版本即可。

// 完成以上所言的一个简单做法是，建立一个 base class，内含所有正常形式的 new 和 delete：
class StandardNewDeleteForms {
public:
    // normal new/delete
    static void* operator new(std::size_t size) throw(std::bad_alloc)
    { return ::operator new(size); }

    static void operator delete(void* pMemory) throw()
    { ::operator delete(pMemory); }

    // placement new/delete
    static void* operator new(std::size_t size, void* ptr) throw()
    { return ::operator new(size, ptr); }

    static void operator delete(void* pMemory, void* ptr) throw()
    { ::operator delete(pMemory, ptr); }

    // nothrow new/delete
    static void* operator new(std::size_t size, const std::nothrow_t& nt) throw()
    { return ::operator new(size, nt); }

    static void operator delete(void* pMemory, const std::nothrow_t&) throw()
    { ::opereator delete(pMemory); }
};





// 凡是想以自定义形式扩充标准形式的客户，可利用继承机制及 using 声明式（见条款 33）
// 取得标准形式：
class Widget : public StandardNewDeleteForm {               // 继承标准形式
public:
    using StandardNewDeleteForms::operator new;             // 让这些形式可见
    using StandardNewDeleteForm::operator delete;

    static void* operator new(std::size_t size,             // 添加一个自定义的
                              std::ostream& logStream)      // placement new
        throw(std::bad_alloc);
    static void operator delete(void* pMemory,              // 添加一个对应的
                                std::ostream& logStream)    // placement delet
        throw();
    ...
};
```

<br>

## 9、杂项讨论

<br>

### 条款 53：不要轻忽编译器的警告

- 严肃对待编译器发出的警告信息。努力在你的编译器的最高（最严苛）警告级别下争取“无任何警告”的荣誉。
- 不要过度倚赖编译器的报警能力，因为不同的编译器对待事情的态度并不相同。一旦移植到另一个编译器上，你原本倚赖的警告信息有可能消失。

<br>

### 条款 54：让自己熟悉包括 TR1 在内的标准程序库

- C++ 标准程序库的主要机能由 STL、iostreams、locales 组成。并包含 C99 标准程序库。
- TR1 添加了智能指针（例如 tr1::shared_ptr）、一般化函数指针（tr1::function）、hash-based 容器、正则表达式（regular expressions）以及另外 10 个组件的支持。
- TR1 自身只是一份规范。为获得 TR1 提供的好处，你需要一份实物。一个好的实物来源是 Boost。

<br>

### 条款 55：让自己熟悉 Boost

- Boost 是一个社群，也是一个网站。致力于免费、源码开放、同僚复审的 C++ 程序库开发。Boost 在 C++ 标准化过程中扮演深具影响力的角色。
- Boost 提供许多 TR1 组件实现品，以及其他许多程序库。