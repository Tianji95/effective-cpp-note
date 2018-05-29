# Note Of Effective C++ And More Effective C++


## Effective C++

#### 一、让自己习惯C++ (Accustoming Yourself to C++ 11)

**1. 视C++ 为一个语言联邦 11（View C++ as a federation of languages 11)**

    主要是因为C++是从四个语言发展出来的：
    C的代码块({}), 语句，数据类型等，
    object-C的class，封装继承多态，virtual动态绑定等，
    template C++的泛型
    STL：容器，迭代器，算法，函数对象等

    因此当这四个子语言相互切换的时候，可以更多地考虑高效编程，例如pass-by-value和pass-by-reference在不同语言中效率不同

总结：
+ C++高效编程守则视状况而变化，取决于使用哪个子语言


**2. 尽量以const, enum, inline替换#define（Prefer consts,enums, and inlines to #defines)**

实际是：应该让编译器代替预处理器定义，因为预处理器定义的变量并没有进入到symbol table里面。编译器有时候会看不到预处理器定义

所以用 

    const double Ratio = 1.653;

来代替 
    
    #define Ratio 1.653

实际上在这个转换中还要考虑到指针，例如需要把指针写成const char* const authorName = "name";而不是只用一个const

以及在class类里面的常量，为了防止被多次拷贝，需要定义成类的成员（添加static）例如

    class GamePlayer{
        static const int numT = 5;
    }

对于类似函数的宏，最好改用inline函数代替，例如：
    
    #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
    template<typename T>
    inline void callWithMax(const T& a, const T& b){
        f(a > b ? a : b);
    }

总结：
+ 对于单纯的常量，最好用const和enums替换#define， 对于形似函数的宏，最好改用inline函数替换#define

**3. 尽可能使用const（Use const whenever possible.)**

const 出现在星号左边，表示被指物是常量，如果出现在星号右边，表示指针本身是常量，如果出现在星号两边，表示被指物和指针都是常量

const最强的用法是在函数声明时，如果将返回值设置成const，或者返回指针设置成const，可以避免很多用户错误造成的意外。

概念上的const：

    考虑这样一段代码
    class CTextBlock{
        public:
            char& operator[](std::size_t position)const{
                return pText[position];
            }
        private:
            char *pText;
    }
    const CTextBlock cctb("Hello");
    char *pc = &cctb[0];
    *pc = 'J'
    这种情况下不会报错，但是一方面声明的时候说了是const，一方面还修改了值。这种逻辑虽然有问题但是编译器并不会报错

但是const使用过程中会出现想要修改某个变量的情况，而另外一部分代码确实不需要修改。这个时候最先想到的方法就是重载一个非const版本。
但是还有其他的方法，例如将非const版本的代码调用const的代码

总结：
+ 将某些东西声明为const可以帮助编译器检查出错误。
+ 编译器强制实施bitwise constneww，但是编写程序的时候应该使用概念上的常量性。
+ 当const和非const版本有着实质等价的实现时，让非const版本调用const版本可以避免代码重复


**4. 确定对象被使用前已先被初始化（Make sure that objects are initialized before they're used)**

对于C++中的C语言来说，初始化变量有可能会导致runtime的效率变低，但是C++部分应该手动保证初始化，否则会出现很多问题。

初始化的函数通常在构造函数上（注意区分初始化和赋值的关系，初始化的效率高，赋值的效率低，而且这些初始化是有次序的，base classes更早于他们的派生类（参看[C++ primer 二刷笔记](https://github.com/Tianji95/note-of-C-plus-plus-primer/blob/master/C%2B%2Bprimer%E4%BA%8C%E5%88%B7%E7%AC%94%E8%AE%B0.md)

除了这些以外，如果我们有两个文件A和B，需要分别编译，A构造函数中用到了B中的对象，那么初始化A和B的顺序就很重要了，这些变量称为（non-local static对象）

解决方法是：将每个non-local static对象搬到自己专属的函数内，并且该对象被声明为static，然后这些函数返回一个reference指向他所含的对象，用户调用这些函数，而不直接涉及这些对象（Singleton模式手法）：

    原代码：
    "A.h"
    class FileSystem{
        public:
            std::size_t numDisks() const;
    };
    extern FileSystem tfs;
    "B.h"
    class Directory{
        public:
            Directory(params){
                std::size_t disks = tfs.numDisks(); //使用tfs
            }
    }
    修改后：
    class FileSystem{...}    //同前
    FileSystem& tfs(){       //这个函数用来替换tfs对象，他在FileSystem class 中可能是一个static，            
        static FileSystem fs;//定义并初始化一个local static对象，返回一个reference
        return fs;
    }
    class Directory{...}     // 同前
    Directory::Directory(params){
        std::size_t disks = tfs().numDisks();
    }
    Directotry& tempDir(){   //这个函数用来替换tempDir对象，他在Directory class中可能是一个static，
        static Directory td; //定义并初始化local static对象，返回一个reference指向上述对象
        return td;
    }

这样做的原理在于C++对于函数内的local static对象会在“该函数被调用期间，且首次遇到的时候”被初始化。当然我们需要避免“A受制于B，B也受制于A”

总结：
+ 为内置型对象进行手工初始化，因为C++不保证初始化他们
+ 构造函数最好使用初始化列初始化而不是复制，并且他们初始化时有顺序的
+ 为了免除跨文件编译的初始化次序问题，应该以local static对象替换non-local static对象

#### 二、构造/析构/赋值运算 (Constructors, Destructors, and Assignment Operators)

**5. 了解C++ 那些自动生成和调用的函数（Know what functions C++ silently writes and calls)**


总结：
+ 编译器可以自动为class生成default构造函数，拷贝构造函数，拷贝赋值操作符，以及析构函数

**6. 若不想使用编译器自动生成的函数，就该明确拒绝（Explicitly disallow the use of compiler-generated functions you do not want)**

这一条主要是针对类设计者而言的，有一些类可能从需求上不允许两个相同的类，例如某一个类表示某一个独一无二的交易记录，那么编译器自动生成的拷贝和复制函数就是无用的，而且是不想要的

总结：
+ 可以将不需要的默认自动生成函数设置成delete的或者弄一个private的父类并且继承下来

**7. 为多态基类声明virtual析构函数（Declare destructors virtual in polymorphic base classes)**

其主要原因是如果基类没有virtual析构函数，那么派生类在析构的时候，如果是delete 了一个base基类的指针，那么派生的对象就会没有被销毁，引起内存泄漏。
例如：
    
    class TimeKeeper{
        public:
        TimeKeeper();
        ~TimeKeeper();
        virtual getTimeKeeper();
    }
    class AtomicClock:public TimeKeeper{...}
    TimeKeeper *ptk = getTimeKeeper();
    delete ptk;
除析构函数以外还有很多其他的函数，如果有一个函数拥有virtual 关键字，那么他的析构函数也就必须要是virtual的，但是如果class不含virtual函数,析构函数就不要加virtual了，因为一旦实现了virtual函数，那么对象必须携带一个叫做vptr(virtual table pointer)的指针，这个指针指向一个由函数指针构成的数组，成为vtbl（virtual table），这样对象的体积就会变大，例如：

    class Point{
        public://析构和构造函数
        private:
        int x, y
    }

本来上面那个代码只占用64bits(假设一个int是32bits)，存放一个vptr就变成了96bits，因此在64位计算机中无法塞到一个64-bits缓存器中，也就无法移植到其他语言写的代码里面了。

总结：
+ 如果一个函数是多态性质的基类，应该有virtual 析构函数
+ 如果一个class带有任何virtual函数，他就应该有一个virtual的析构函数
+ 如果一个class不是多态基类，也没有virtual函数，就不应该有virtual析构函数

**8. 别让异常逃离析构函数（Prevent exceptions from leaving destructors)**

这里主要是因为如果循环析构10个Widgets，如果每一个Widgets都在析构的时候抛出异常，就会出现多个异常同时存在的情况，这里如果把每个异常控制在析构的话就可以解决这个问题：解决方法为：

原代码：

    class DBConn{
    public:
        ~DBConn(){
            db.close();
        }
    private:
        DBConnection db;
    }

修改后的代码：
    
    class DBConn{
    public:
        void close(){
            db.close();
            closed = true;
        }

        ~DBConn(){
            if(!closed){
                try{
                    db.close();
                }
                catch(...){
                    std::abort();
                }
            }
        }
    private:
        bool closed;
        DBConnection db;
    }
这种做法就可以一方面将close的的方法交给用户，另一方面在用户忽略的时候还能够做“强迫结束程序”或者“吞下异常”的操作。相比较而言，交给用户是最好的选择，因为用户有机会根据实际情况操作异常。

总结：
+ 析构函数不要抛出异常，因该在内部捕捉异常
+ 如果客户需要对某个操作抛出的异常做出反应，应该将这个操作放到普通函数（而不是析构函数）里面

**9. 绝不在构造和析构过程中调用virtual函数（Never call virtual functions during construction or destruction)**

主要是因为有继承的时候会调用错误版本的函数，例如

原代码：

    class Transaction{
    public:
        Transaction(){
            logTransaction();
        }
        Virtual void logTransaction const() = 0;
    };
    class BuyTransaction:public Transaction{
        public:
            virtual void logTransaction() const;
    };
    BuyTransaction b;

    或者有一个更难发现的版本：

    class Transaction{
    public:
        Transaction(){init();}
        virtual void logTransaction() const = 0;
    private:
        void init(){
            logTransaction();
        }
    };
这个时候代码会调用 Transaction 版本的logTransaction，因为在构造函数里面是先调用了父类的构造函数，所以会先调用父类的logTransaction版本，解决方案是不在构造函数里面调用，或者将需要调用的virtual弄成non-virtual的

修改以后：

    class Transaction{
    public:
        explicit Transaction(const std::string& logInfo);
        void logTransaction(const std::string& logInfo) const; //non-virtual 函数
    }
    Transaction::Transaction(const std::string& logInfo){
        logTransaction(logInfo); //non-virtual函数
    }
    class BuyTransaction: public Transaction{
    public:
        BuyTransaction(parameters):Transaction(createLogString(parameters)){...} //将log信息传递给base class 构造函数
    private:
        static std::string createLogString(parameters); //注意这个函数是用来给上面那个函数初始化数据的，这个辅助函数的方法
    }

总结：
+ 在构造和析构期间不要调用virtual函数，因为这类调用从不下降至派生类的版本

**10. 令operator= 返回一个reference to *this （Have assignment operators return a reference to *this)**

主要是为了支持连读和连写，例如：
    
    class Widget{
    public:
        Widget& operator=(int rhs){return *this;}
    }
    a = b = c;

**11. 在operator= 中处理“自我赋值” （Handle assignment to self in operator=)**

主要是要处理 a[i] = a[j] 或者 *px = *py这样的自我赋值。有可能会出现一场安全性问题，或者在使用之前就销毁了原来的对象，例如

原代码：
    
    class Bitmap{...}
    class Widget{
    private:
        Bitmap *pb;
    };
    Widget& Widget::operator=(const Widget& rhs){
        delete pb; // 当this和rhs是同一个对象的时候，就相当于直接把rhs的bitmap也销毁掉了
        pb = new Bitmap(*rhs.pb);
        return *this;
    }

修改后的代码

    class Widget{
        void swap(Widget& rhs);    //交换this和rhs的数据
    };
    Widget& Widget::operator=(const Widget& rhs){
        Widget temp(rhs)           //为rhs数据制作一个副本
        swap(temp);                //将this数据和上述副本数据交换
        return *this;
    }//出了作用域，原来的副本销毁

或者有一个效率不太高的版本：
    Widget& Widget::operator=(const Widget& rhs){
        Bitmap *pOrig = pb;       //记住原先的pb
        pb = new Bitmap(*rhs.pb); //令pb指向 *pb的一个副本
        delete pOrig;            //删除原先的pb
        return *this;
    }

总结：
+ 确保当对象自我赋值的时候operator=有比较良好的行为，包括两个对象的地址，语句顺序，以及copy-and-swap
+ 确定任何函数如果操作一个以上的对象，而其中多个对象可能指向同一个对象时，仍然正确

**12. 复制对象时勿忘其每一个成分 （Copy all parts of an object)**

总结：
+ 当编写一个copy或者拷贝构造函数，应该确保复制成员里面的所有变量，以及所有基类的成员
+ 不要尝试用一个拷贝构造函数调用另一个拷贝构造函数，如果想要精简代码的话，应该把所有的功能机能放到第三个函数里面，并且由两个拷贝构造函数共同调用
+ 当新增加一个变量或者继承一个类的时候，很容易出现忘记拷贝构造的情况，所以每增加一个变量都需要在拷贝构造里面修改对应的方法

#### 三、资源管理 (Resource Management)

**13. 以对象管理资源 （Use objects to manage resources)**

主要是为了防止在delete语句执行前return，所以需要用对象来管理这些资源。这样当控制流离开f以后，该对象的析构函数会自动释放那些资源。
例如shared_ptr就是这样的一个管理资源的对象。他是在自己的析构函数里面做delete操作。所以如果自己需要管理资源的时候，也要在类内进行delete，通过对象来管理资源

总结：
+ 建议使用shared_ptr
+ 如果需要自定义shared_ptr，请通过定义自己的资源管理类来对资源进行管理

**14. 在资源管理类中小心copying行为 （Think carefully about copying behavior in resource-managing classes)**

在资源管理类里面，如果出现了拷贝复制行为的话，需要注意这个复制具体的含义，从而保证和我们想要的效果一样

思考下面代码在复制中会发生什么：
    
    class Lock{
    public:
        explicit Lock(Mutex *pm):mutexPtr(pm){
            lock(mutexPtr);//获得资源锁
        }
        ~Lock(){unlock(mutexPtr);}//释放资源锁
    private:
        Mutex *mutexPtr;
    }
    Lock m1(&m)//锁定m
    Lock m2(m1);//好像是锁定m1这个锁。。而我们想要的是除了复制资源管理对象以外，还想复制它所包括的资源（deep copy）。通过使用shared_ptr可以有效避免这种情况。

需要注意的是：copy函数有可能是编译器自动创建出来的，所以在使用的时候，一定要注意自动生成的函数是否符合我们的期望

总结;
+ 复制RAII对象（Resource Acquisition Is Initialization）必须一并复制他所管理的资源（deep copy）
+ 普通的RAII做法是：禁止拷贝，使用引用计数方法

**15. 在资源管理类中提供对原始资源的访问（Provide access to raw resources in resource-managing classes)**

例如：shared_ptr<>.get()这样的方法，或者->和*方法来进行取值。但是这样的方法可能稍微有些麻烦，有些人会使用一个隐式转换，但是经常会出错：
    
    class Font; class FontHandle;
    void changeFontSize(FontHandle f, int newSize){    }//需要调用的API

    Font f(getFont());
    int newFontSize = 3;
    changeFontSize(f.get(), newFontSize);//显式的将Font转换成FontHandle

    class Font{
        operator FontHandle()const { return f; }//隐式转换定义
    }
    changeFontSize(f, newFontSize)//隐式的将Font转换成FontHandle
    但是容易出错，例如
    Font f1(getFont());
    FontHandle f2 = f1;就会把Font对象换成了FontHandle才能复制

总结：
+ 每一个资源管理类RAII都应该有一个直接获得资源的方法
+ 隐式转换对客户比较方便，显式转换比较安全，具体看需求

**16. 成对使用new和delete时要采取相同形式 （Use the same form in corresponding uses of new and delete)**

总结：
+ 即： 使用new[]的时候要使用delete[], 使用new的时候一定不要使用delete[]

**17. 以独立语句将new的对象置入智能指针 （Store newed objects in smart pointers in standalone statements)**

主要是会造成内存泄漏，考虑下面的代码：
    
    int priority();
    void processWidget(shared_ptr<Widget> pw, int priority);
    processWidget(new Widget, priority());// 错误，这里函数是explicit的，不允许隐式转换（shared_ptr需要给他一个普通的原始指针
    processWidget(shared_ptr<Widget>(new Widget), priority()) // 可能会造成内存泄漏

    内存泄漏的原因为：先执行new Widget，再调用priority， 最后执行shared_ptr构造函数，那么当priority的调用发生异常的时候，new Widget返回的指针就会丢失了。当然不同编译器对上面这个代码的执行顺序不一样。所以安全的做法是：

    shared_ptr<Widget> pw(new Widget)
    processWidget(pw, priority())

总结：
+ 凡是有new语句的，尽量放在单独的语句当中，特别是当使用new出来的对象放到智能指针里面的时候

#### 四、设计与声明 (Designs and Declarations)

**18. 让接口容易被正确使用，不易被误用  （Make interfaces easy to use correctly and hard to use incorrectly)**





**19. 设计class犹如设计type  （Treat class design as type design)**

**20. 以pass-by-reference-to-const替换pass-by-value  （Prefer pass-by-reference-to-const to pass-by-value)**

**21. 必须返回对象时，别妄想返回其reference  （Don't try to return a reference when you must return an object)**


**22. 将成员变量声明为private  （Declare data members private)**


**23. 以non-member、non-friend替换member函数  （Prefer non-member non-friend functions to member functions)**


**24. 若所有参数皆需类型转换，请为此采用non-member函数  （Declare non-member functions when type conversions should apply to all parameters)**


**25. 考虑写出一个不抛异常的swap函数  （Consider support for a non-throwing swap)**


#### 五、实现 (Implementations)


**26. 尽可能延后变量定义式的出现时间  （Postpone variable definitions as long as possible)**


**27. 尽量不要进行强制类型转换  （Minimize casting)**


**28. 避免返回handles指向对象内部成分  （Avoid returning "handles" to object internals)**

**29. 为“异常安全”而努力是值得的  （Strive for exception-safe code)**


**30. 透彻了解inlining  （Understand the ins and outs of inlining)**



**31. 将文件间的编译依存关系降至最低  （Minimize compilation dependencies between files)**


#### 六、继承与面向对象设计 (Inheritance and Object-Oriented Design)


**32. 确定你的public继承塑模出is-a关系  （Make sure public inheritance models "is-a.")**

**33. 避免遮掩继承而来的名称  （Avoid hiding inherited names)**



**34. 区分接口继承和实现继承  （Differentiate between inheritance of interface and inheritance of implementation)**



**35. 考虑virtual函数以外的其他选择  （Consider alternatives to virtual functions)**

**36. 绝不重新定义继承而来的non-virtual函数  （Never redefine an inherited non-virtual function)**

**37. 绝不重新定义继承而来的缺省参数值  （Never redefine a function's inherited default parameter value)**



**38. 通过复合塑模出has-a或"根据某物实现出"  （Model "has-a" or "is-implemented-in-terms-of" through composition)**

**39. 明智而审慎地使用private继承  （Use private inheritance judiciously)**

**40. 明智而审慎地使用多重继承  （Use multiple inheritance judiciously)**




#### 七、模板与泛型编程 (Templates and Generic Programming)

**41. 了解隐式接口和编译期多态 （Understand implicit interfaces and compile-time polymorphism)**


**42. 了解typename的双重意义 （Understand the two meanings of typename)**


**43. 学习处理模板化基类内的名称 （Know how to access names in templatized base classes)**


**44. 将与参数无关的代码抽离templates （Factor parameter-independent code out of templates)**


**45. 运用成员函数模板接受所有兼容类型 （Use member function templates to accept "all compatible types.")**


**46. 需要类型转换时请为模板定义非成员函数 （Define non-member functions inside templates when type conversions are desired)**


**47. 请使用traits classes表现类型信息 （Use traits classes for information about types)**

**48. 认识template元编程 （Be aware of template metaprogramming)**


#### 八、定制new和delete (Customizing new and delete)

**49. 了解new-handler的行为 （Understand the behavior of the new-handler)**

**50. 了解new和delete的合理替换时机 （Understand when it makes sense to replace new and delete)**

**51. 编写new和delete时需固守常规（Adhere to convention when writing new and delete)**

**52. 写了placement new也要写placement delete（Write placement delete if you write placement new)**

#### 杂项讨论 (Miscellany)

**53. 不要轻忽编译器的警告（Pay attention to compiler warnings)**

**54. 让自己熟悉包括TR1在内的标准程序库 （Familiarize yourself with the standard library, including TR1)**

**55. 让自己熟悉Boost （Familiarize yourself with Boost)**




## More Effective C++


#### 一、基础议题

**1. 区分指针和引用**

**2. 优先考虑C++风格的类型转换**

**3. 决不要把多态用于数组**


**4. 避免不必要的默认构造函数**

#### 二、运算符

**5. 小心用户自定义的转换函数**


**6. 区分自增运算符和自减运算符的前缀形式与后缀形式**

**7. 不要重载"&&"、"||"和","**

**8. 理解new和delete在不同情形下的含义**


#### 三、异常


**9. 使用析构函数防止资源泄漏**

**10. 防止构造函数里的资源泄漏**

**11. 阻止异常传递到析构函数以外**

**12. 理解抛出异常与传递参数或者调用虚函数之间的不同**

**13. 通过引用捕获异常**

**14. 审慎地使用异常规格**


**15. 理解异常处理所付出的代价**


#### 四、效率


**16. 记住80-20准则**

**17. 考虑使用延迟计算**

**18. 分期摊还预期的计算开销**

**19. 了解临时对象的来源**

**20. 协助编译器实现返回值优化**

**21. 通过函数重载避免隐式类型转换**

**22. 考虑使用op=来取代单独的op运算符**

**23. 考虑使用其他等价的程序库**

**24. 理解虚函数、多重继承、虚基类以及RTTI所带来的开销**



#### 五、技巧

**25. 使构造函数和非成员函数具有虚函数的行为**

**26. 限制类对象的个数**

**27. 要求或禁止对象分配在堆上**

**28. 智能(smart)指针**

**29. 引用计数**

**30. 代理类**

**31. 基于多个对象的虚函数**


#### 六、杂项


**32. 在将来时态下开发程序**

**33. 将非尾端类设计为抽象类**

**34. 理解如何在同一程序中混合使用C**

**35. 让自己熟悉C++语言标准**




## Effective Modern C++

some note copy from [EffectiveModernCppChinese](https://github.com/racaljk/EffectiveModernCppChinese)

#### 一、类型推导

**1. 理解模板类型推导**

**2. 理解auto类型推导**

**3. 理解decltype**

**4. 学会查看类型推导结果**

#### 二、auto

**5. 优先考虑auto而非显式类型声明**

**6. auto推导若非己愿，使用显式类型初始化惯用法**


#### 三、移步现代C++

**7. 区别使用()和{}创建对象**

**8. 优先考虑nullptr而非0和NULL**

**9. 优先考虑别名声明而非typedefs**

**10. 优先考虑限域枚举而非未限域枚举**

**11. 优先考虑使用delete而非使用未定义的私有声明**

**12. 使用override声明重载函数**

**13. 优先考虑const_iterator而非iterator**

**14. 如果函数不抛出异常请使用noexcept**

**15. 尽可能的使用constexpr**

**16. 确保const成员函数线程安全**

**17. 理解特殊成员函数函数的生成**

#### 四、智能指针

**18. 对于占有性资源使用std::unique_ptr**

**19. 对于共享性资源使用std::shared_ptr**

**20. 对于类似于std::shared_ptr的指针使用std::weak_ptr可能造成悬置**

**21. 优先考虑使用std::make_unique和std::make_shared而非new**

**22. 当使用Pimpl惯用法，请在实现文件中定义特殊成员函数**


#### 五、右值引用，移动语意，完美转发

**23. 理解std::move和std::forward**

**24. 区别通用引用和右值引用**

**25. 对于右值引用使用std::move，对于通用引用使用std::forward**

**26. 避免重载通用引用**

**27. 熟悉重载通用引用的替代品**

**28. 理解引用折叠**

**29. 认识移动操作的缺点**

**30. 熟悉完美转发失败的情况**


#### 六、Lambda表达式

**31. 避免使用默认捕获模式**

**32. 使用初始化捕获来移动对象到闭包中**

**33. 对于std::forward的auto&&形参使用decltype**

**34. 有限考虑lambda表达式而非std::bind**


#### 七、并发API

**35. 优先考虑基于任务的编程而非基于线程的编程**

**36. 如果有异步的必要请指定std::launch::threads**

**37. 从各个方面使得std::threads unjoinable**

**38. 知道不同线程句柄析构行为**

**39. 考虑对于单次事件通信使用void**

**40. 对于并发使用std::atomic，volatile用于特殊内存区**

#### 八、微调

**41. 对于那些可移动总是被拷贝的形参使用传值方式**

**42. 考虑就地创建而非插入**
