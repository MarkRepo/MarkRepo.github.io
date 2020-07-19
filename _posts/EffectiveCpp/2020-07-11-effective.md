---
title: effectivecpp
descriptions: effective cpp note
categories: language
tags: design
---

## 1. View C++ as a federation of languages.

Today’s C++ is a *multiparadigm programming language*, one support ing a combination of procedural, object-oriented, functional, generic, and metaprogramming features. This power and flexibility make C++ a tool without equal（使C ++成为无与伦比的工具）, but can also cause some confusion. All the “proper usage” rules seem to have exceptions. How are we to make sense of such a language? The easiest way is to view C++ not as a single language but as a federation of related languages. Within a particular sublanguage, the rules tend to be simple, straightforward, and easy to remember.

1. **C**
2. **Object-Oriented C++**
3. **Template C++.**
4. **The STL**

Keep these four sublanguages in mind, and don’t be surprised when you encounter situations where effective programming requires that you change strategy when you switch from one sublanguage to another.

Rules for effective C++ programming vary, depending on the part of C++ you are using.

## 2. Prefer consts, enums, and inlines to #defines.

This Item might better be called “**prefer the compiler to the preprocessor,**” because #define may be treated as if it’s not part of the language *per se*(per se：本身)

\(1) #define ASPECT_RATIO 1.653

(2) const double AspectRatio = 1.653;

1. **\#define: the name you’re programming with may not be in the symbol table**。
2. **use of the constant may yield smaller code than using a #define**, That’s because the preprocessor’s blind substitution of the macro name ASPECT_RATIO with 1.653 could result in multiple copies of 1.653 in your object code, while the use of the constant AspectRatio should never result in more than one copy

When replacing #defines with constants, two special cases are worth mentioning

1. The first is defining constant pointers.

   Because constant definitions are typically put in header files,  it’s important that the *pointer* be declared const, usually in addition to what the pointer points to. To define a constant char*-based string in a header file, for example, you have to write const *twice*:

   ```cpp
   const char * const authorName = "Scott Meyers";
   ```

   For a complete discussion of the meanings and uses of const, espe- cially in conjunction with pointers, see Item 3

   However, it’s worth reminding you here that string objects are generally preferable to their char*-based progenitors, so authorName is often better defined this way:

   ```cpp
   const std::string authorName("Scott Meyers");
   ```

2. The second special case concerns class-specific constants

   To limit the scope of a constant to a class, you must make it a member, and to ensure there’s at most one copy of the constant, you must make it a *static* member:

   ```cpp
   class GamePlayer { 
    private:
   	static const int NumTurns = 5; //declaration, not definition
   	int scores[NumTurns];
   	...
   };
   ```

   Usually, C++ requires that you provide a definition for anything you use, but class-specific constants that are static and of integral type (e.g., integers, chars, bools) are an exception. As long as you don’t take their address, you can declare them and use them without providing a definition. If you do take the address of a class constant, or if your compiler incorrectly insists on a definition even if you don’t take the address, you provide a separate definition like this:

   ```cpp
   const int GamePlayer::NumTurns; // definition of NumTurns; see
   																// below for why no value is given
   ```

   You put this in an implementation file, not a header file. Because the initial value of class constants is provided where the constant is declared (e.g., NumTurns is initialized to 5 when it is declared), no ini- tial value is permitted at the point of definition(因为在声明常量的地方提供了类常量的初始值（例如，在声明常量时将NumTurns初始化为5），所以在定义点不允许有任何初始值。)

   There’s no way to create a class-specific con- stant using a #define, because #defines don’t respect scope. Once a macro is defined, it’s in force for the rest of the compilation (unless it’s \#undefed somewhere along the line). Which means that not only can’t #defines be used for class-specific constants, they also can’t be used to provide any kind of encapsulation, i.e., there is no such thing as a “private” #define.

   Older compilers may not accept the syntax above, because it used to be illegal to provide an initial value for a static class member at its point of declaration. Furthermore, in-class initialization is allowed only for integral types and only for constants. In cases where the above syntax can’t be used, you put the initial value at the point of definition:

   ```cpp
   class CostEstimate { 
    private:
   	static const double FudgeFactor;	// declaration of static class constant; goes in header file
   	...												
   };
   const double CostEstimate::FudgeFactor = 1.35;// definition of static class constant; goes in impl. file
   ```

   The only exception is when you need the value of a class constant during compilation of the class, such as in the declaration of the array GamePlayer::scores above (where compilers insist on knowing the size of the array during compilation). 

   ​		Then the accepted way to compensate for compilers that (incorrectly) forbid the in-class specification of initial values for static integral class constants is to use what is affectionately (and non-pejoratively) known as “the enum hack.” (那么，补偿（错误地）禁止静态整数类常量的初始值的类内规范的编译器的公认方法是使用被亲切（且非贬义地）称为“枚举hack”的方法。)

   ```cpp
   class GamePlayer { 
    private:
   	enum { NumTurns = 5 };
   	int scores[NumTurns];
   ...
   };
   ```

   The enum hack is worth knowing about for several reasons.	

   1. First, the enum hack behaves in some ways more like a #define than a const does, and sometimes that’s what you want. 

      For example, it’s legal to take the address of a const, but it’s not legal to take the address of an enum, and it’s typically not legal to take the address of a #define, either. If you don’t want to let people get a pointer or reference to one of your integral constants, an enum is a good way to enforce that constraint.

      另外，尽管好的编译器不会为整数类型的const对象预留存储空间（除非您创建指针或对该对象的引用），但草率的编译器可能会，并且您可能不愿意为此类对象预留内存。 像#defines一样，枚举绝不会导致这种不必要的内存分配。

   2. A second reason to know about the enum hack is purely pragmatic(务实). Lots of code employs it, so you need to recognize it when you see it. In fact, the enum hack is a fundamental technique of template metapro- gramming (see Item 48).

Getting back to the preprocessor, another common (mis)use of the #define directive is using it to implement macros that look like func- tions but that don’t incur the overhead of a function call. Here’s a macro that calls some function f with the greater of the macro’s argu- ments:

```cpp
// call f with the maximum of a and b
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

Whenever you write this kind of macro, you have to remember to parenthesize all the arguments in the macro body. Otherwise you can run into trouble when somebody calls the macro with an expression. But even if you get that right, look at the weird things that can happen:

```cpp
inta=5,b=0;
CALL_WITH_MAX(++a, b); // a is incremented twice
CALL_WITH_MAX(++a, b+10); // a is incremented once
```

Here, the number of times that a is incremented before calling f depends on what it is being compared with!

Fortunately, you don’t need to put up with this nonsense. You can get all the efficiency of a macro plus all the predictable behavior and type safety of a regular function by using a template for an inline function (see Item 30):

```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b) { // because we don’t know what T is, 
  f(a>b?a:b);																			//we pass by reference-to-const seeItem20
}
```

Things to Remember

✦ For simple constants, prefer const objects or enums to #defines.

 ✦ For function-like macros, prefer inline functions to #defines.



## 3. Use const whenever possible.

The wonderful thing about const is that it allows you to specify a semantic constraint — a particular object should *not* be modified — and compilers will enforce that constraint.

```cpp
char greeting[] = "Hello"; 
char *p = greeting;							// non-const pointer,  non-const data
const char *p = greeting;				// non-const pointer,  const data
char * const p = greeting; 			// const pointer, non-const data
const char * const p = greeting;// const pointer, const data
```

1. If the word const appears to the left of the asterisk, what’s *pointed to* is constant; if the word const appears to the right of the asterisk, the *pointer itself* is content

2. Declaring an iterator const is like declaring a pointer const (i.e., declaring a T* const pointer): the iterator isn’t allowed to point to something different, but the thing it points to may be modified. If you want an iterator that points to something that can’t be modified (i.e., the STL analogue of a const T* pointer), you want a const_iterator.

3. const的一些最强大用法来自其对函数声明的应用。 在函数声明中，const可以引用函数的返回值，单个参数，对于成员函数，还可以引用整个函数。

4. The purpose of const on member functions is to identify which member functions may be invoked on const objects. Such member functions are important for two reasons. First, they make the interface of a class easier to understand. It’s important to know which functions may modify an object and which may not. Second, they make it possible to work with const objects. That’s a critical aspect of writing efficient code, because, as Item 20 explains, one of the fundamental ways to improve a C++ program’s performance is to pass objects by reference-to-const. That technique is viable only if there are const member functions with which to manipulate the resulting const-qualified objects

5. it’s never legal to modify the return value of a function that returns a built-in type. Even if it were legal, the fact that C++ returns objects by value (see Item 20) would mean that a *copy* of tb.text[0] would be modified, not tb.text[0] itself, and that’s not the behavior you want.

6. mutable frees non-static data members from the constraints of bitwise constness.

7. Having the non-const operator[] call the const version is a safe way to avoid code duplication, even though it requires a cast

```cpp
class TextBlock { 
public:
...
const char& operator[](std::size_t position) const {
	return text[position]; 
}
char& operator[](std::size_t position) {
	return const_cast<char&>(static_cast<const TextBlock&>(*this) [position]); 
}
};
```

Things to Remember

✦  Declaring something const helps compilers detect usage errors. const can be applied to objects at any scope, to function parameters and return types, and to member functions as a whole.

✦  Compilers enforce bitwise constness, but you should program using logical constness.

✦  When const and non-const member functions have essentially identi- cal implementations, code duplication can be avoided by having the non-const version call the const version



## 4. Make sure that objects are initialized  before they're used

*always* initialize your objects before you use them.

1. The rules of C++ stipulate(规定) that data members of an object are initialized *before* the body of a constructor is entered.

2. for built-in type class member,  there’s no guarantee it was initialized at all prior to its assignment.

```cpp
class PhoneNumber { ... };
class ABEntry { // ABEntry = “Address Book Entry” 
 public:
	ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones);
 private:
	std::string theName;
	std::string theAddress; 
	std::list<PhoneNumber> thePhones; 
	int numTimesConsulted;
};

ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
{
  // these are all assignments not initializations
	theName = name; 
  theAddress = address; 
  thePhones = phones; 
  numTimesConsulted = 0;
}
```

3. if possileble, alway use initializer list

   ```cpp
   ABEntry::ABEntry(
     const std::string& name, 
     const std::string& address, 
     const std::list<PhoneNumber>& phones)
   : theName(name),theAddress(address), thePhones(phones),numTimesConsulted(0)
   {}
   ```

   Sometimes the initialization list *must* be used, even for built-in types. For example, data members that are const or are references must be initialized; they can’t be assigned

4. One aspect of C++ that isn’t fickle is the order in which an object’s data is initialized. This order is always the same: base classes are ini- tialized before derived classes (see also Item 12), and within a class, data members are initialized in the order in which they are declared.

5. the relative order of initialization of non- local static objects defined in different translation units is undefined.

   ​	A *static object* is one that exists from the time it’s constructed until the end of the program. Stack and heap-based objects are thus excluded. Included are global objects, objects defined at namespace scope, objects declared static inside classes, objects declared static inside functions, and objects declared static at file scope. Static objects inside functions are known as *local static objects* (because they’re local to a function), and the other kinds of static objects are known as *non-local static objects*. Static objects are destroyed when the program exits, i.e., their destructors are called when main finishes executing.

   ​	Avoid initialization order problems across translation units by re- placing non-local static objects with local static objects.

```cpp
class FileSystem { ... };
FileSystem& tfs() {
  // this replaces the tfs object; it could be // static in the FileSystem class
	// define and initialize a local static object // return a reference to it

	static FileSystem fs; 
  return fs;
}
class Directory { ... };
// as before
// as before, except references to tfs are now to tfs()
Directory::Directory( params ) {
	std::size_t disks = tfs().numDisks();
}

Directory& tempDir() {
	static Directory td( params ); 
  return td;
}

```

## 5. Know what functions C++ silently writes and calls.

1. If you don’t declare them yourself, compilers will declare their own versions of a copy constructor, a copy assignment operator, and a destructor. Furthermore, if you declare no constructors at all, compil- ers will also declare a default constructor for you. All these functions will be both public and inline

   As a result, if you write

```cpp
class Empty{};
```

​	it’s essentially the same as if you’d written this:

```cpp
class Empty { public:
	Empty() { ... }// default constructor
 	Empty(const Empty& rhs) { ... } // copy constructor
	~Empty() { ... }// destructor — see below for whether it’s virtual
 	Empty& operator=(const Empty& rhs) { ... }// copy assignment operator
};
```

​	These functions are generated only if they are needed

2. 假设编译器正在为您编写函数，那么这些函数会做什么？ 默认的构造函数和析构函数主要为编译器提供了一个放置“幕后”代码的地方，例如调用基类和非静态数据成员的构造函数和析构函数。请注意，所生成的析构函数是非虚拟的（请参阅第7条），除非它是针对从基类继承的类，而基类本身声明了虚拟析构函数（在这种情况下，函数的虚拟性来自基类）。对于复制构造函数和复制赋值运算符，编译器生成的版本仅将源对象的每个非静态数据成员复制到目标对象。

3. compiler-generated copy assignment operators behave as I’ve described only when the resulting code is both legal and has a reasonable chance of making sense. If either of these tests fails, compilers will refuse to generate an operator= for your class

   ```cpp
   template<typename T> class NamedObject { 
    public:
    	NamedObject(std::string& name, const T& value);
    private:
   	std::string& nameValue; const T objectValue;
   };
   std::string newDog("Persephone"); 
   std::string oldDog("Satch");
   NamedObject<int> p(newDog, 2);
   NamedObject<int> s(oldDog, 36);
   p = s;    //error , compiler reject generate copy assignment oeprator, as illegal
   ```

## 6. Explicitly disallow the use of compiler- generated functions you do not want.

1. declare the copy constructor and the copy assignment operator *private*(if called, compile error)

2. because member and friend functions can still call your private functions. so declaring member functions private and deliberately not implementing them.(if member or friend func call it, lead to linker error)

   ```cpp
   class HomeForSale { 
   public:
   ...
   private: ...
   	HomeForSale(const HomeForSale&);// declarations only
   	HomeForSale& operator=(const HomeForSale&);
   };
   ```

3. It’s possible to move the link-time error up to compile time (always a good thing — earlier error detection is better than later) by declaring the copy constructor and copy assignment operator private not in HomeForSale itself, but in a base class specifically designed to prevent copying.

   ```cpp
   class Uncopyable { 
     protected:
   	Uncopyable() {} 	// allow construction 
     ~Uncopyable() {}	// and destruction of
     								 	// derived objects...
   	private:
   	Uncopyable(const Uncopyable&); 
     Uncopyable& operator=(const Uncopyable&);// ...but prevent copying
   };
   
   class HomeForSale: private Uncopyable { 
     ...
   };
   ```

   The implementation and use of Uncopyable include some subtleties, such as the fact that inheritance from Uncopyable needn’t be public (see Items 32 and 39) and that Uncopyable’s destructor need not be virtual (see Item 7). Because Uncopyable contains no data, it’s eligible for the empty base class optimization described in Item 39, but because it’s a base class, use of this technique could lead to multiple inheritance (see Item 40). Multiple inheritance, in turn, can some- times disable the empty base class optimization (again, see Item 39). In general, you can ignore these subtleties and just use Uncopyable as shown, because it works precisely as advertised.

## 7. Declare destructors virtual in polymorphic base classes.

1. if not have virutal destructor, the derived part of the object is never destroyed.

2. Any class with virtual functions should almost certainly have a virtual destructor.

3. When a class is not intended to be a base class, making the destructor virtual is usually a bad idea.

   Because objects of that type will increase in size for vptr

4. Inherited from a class that has no virtual destructor, such as stl container and string, is bad idea. So, if class has non-virtual destructor, use final declare it.

5.  declare a pure virtual destructor in the class you want to be abstract; 

   but you must provide a *definition* for the pure virtual destructor: because  destructor of derived class  would call destructor of each base class

6. The rule for giving base classes virtual destructors applies only to *polymorphic* base classes — to base classes designed to allow the manipulation of derived class types through base class interfaces.

## 8. Prevent exceptions from leaving destructors.

Premature program termination or undefined behavior can result from destructors emitting exceptions even without using containers and arrays. C++ does *not* like destructors that emit exceptions!

That’s easy enough to understand, but what should you do if your destructor needs to perform an operation that may fail by throwing an exception?There are two primary ways to avoid the trouble. 

- Terminate the program if close throws

  ```cpp
  DBConn::~DBConn( ) {
    try { 
      db.close(); 
    } catch (...) {
      make log entry that the call to close failed;
    std::abort(); 
    }
  }
  ```

- Swallow the exception arising from the call to close:

  ```cpp
  DBConn::~DBConn( ) {
  	try { 
      db.close(); 
    } 
    catch (...) {
   	//make log entry that the call to close failed; 
    }
  }
  ```

  通常，吞咽异常是一个坏主意，因为它会抑制重要信息---某些事情失败了！

- A better strategy is to design DBConn’s interface so that its clients have an opportunity to react to problems that may arise.

  ```cpp
  class DBConn { 
  public:
  ...
  void close() {// new function for // client use
    db.close();
    closed = true; 
  }
  ~DBConn( ) {
  	if (!closed) { 
      try {// close the connection if the client didn’t
  			db.close();
  		}catch (...) {
        make log entry that call to close failed;	// if closing fails,note that and terminate or swallow
      	...
      }
  private: 
    DBConnection db; 
    bool closed;
  };
  ```

  Things to Remember

  ✦  Destructors should never emit exceptions. If functions called in a destructor may throw, the destructor should catch any exceptions, then swallow them or terminate the program.

  ✦  If class clients need to be able to react to exceptions thrown during an operation, the class should provide a regular (i.e., non-destructor) function that performs the operation.

## 9. Never call virtual functions during construction or destruction.

```cpp
class Transaction { 
 public:
	Transaction( );
	virtual void logTransaction() const = 0;
};

Transaction::Transaction() {
	logTransaction(); 
}
class BuyTransaction: public Transaction { 
public:
	virtual void logTransaction() const;
};
class SellTransaction: public Transaction { 
public:
	virtual void logTransaction() const;
};
```

During base class construction, virtual functions never go down into derived classes. Instead, the object behaves as if it were of the base type.Not only do virtual functions resolve to the base class, but the parts of the language using runtime type information (e.g., dynamic_cast (see Item 27) and typeid) treat the object as a base class type

Because base class constructors execute before derived class constructors, derived class data members have not been initialized when base class constructors run. If virtual functions called during base class construction went down to derived classes, the derived class functions would almost certainly refer to local data members, but those data members would not yet have been initialized. That would be a non-stop ticket to undefined behavior and late-night debugging sessions. Calling down to parts of an object that have not yet been initialized is inherently dangerous, so C++ gives you no way to do it.

The same reasoning applies during destruction. Once a derived class destructor has run, the object’s derived class data members assume undefined values, so C++ treats them as if they no longer exist. Upon entry to the base class destructor, the object becomes a base class object, and all parts of C++ — virtual functions, dynamic_casts, etc., — treat it that way.

The only way to avoid this problem is to make sure that none of your constructors or destructors call virtual functions on the object being created or destroyed and **that all the functions they call obey the same constraint.**

In other words, since you can’t use virtual functions to call down from base classes during construction, you can compensate by having derived classes pass necessary construction information up to base class constructors instead:

```cpp
class Transaction { 
  public:
		explicit Transaction(const std::string& logInfo);
		void logTransaction(const std::string& logInfo) const;
};
Transaction::Transaction(const std::string& logInfo) {
	logTransaction(logInfo); 
}
class BuyTransaction: public Transaction { 
  public:
		BuyTransaction( parameters ): Transaction(createLogString( parameters )) {...}
	private:
		static std::string createLogString( parameters );
};
```

 By making the function static, there’s no danger of accidentally referring to the nascent BuyTransaction object’s as-yet-uninitialized data members.

## 10.Have assignment operators return a reference to *this.

This is only a convention

## 11.Handle assignment to self in operator=.

think about be self-assignment-safe, or you can fall into the trap of accidentally releasing a resource before you’re done using it:

```cpp
class Bitmap { ... };
class Widget { ...
private:
Bitmap *pb;
// ptr to a heap-allocated object
};
// unsafe impl. of operator=
Widget& Widget::operator=(const Widget& rhs) {
	delete pb;
	pb = new Bitmap(*rhs.pb);
	return *this;
}
```

*this (the target of the assignment) and rhs could be the same object, so it's not safe

1. The traditional way to prevent this error is to check for assignment to self via an *identity test* at the top of operator=:

```cpp
Widget& Widget::operator=(const Widget& rhs) {
	if (this == &rhs) return *this;
	delete pb;
	pb = new Bitmap(*rhs.pb);
	return *this; 
}
```

but is also have exception trouble. In particular, if the “new Bitmap” expression yields an exception (either because there is insufficient memory for the allocation or because Bitmap’s copy constructor throws one), the Widget will end up holding a pointer to a deleted map

2. exception-safe include safe-assignment-safe, not to delete pb until after we’ve copied what it points to:

```cpp
Widget& Widget::operator=(const Widget& rhs) {
  Bitmap *pOrig = pb;
  pb = new Bitmap(*rhs.pb); 
  delete pOrig;
  return *this; 
}
```

Now, if “new Bitmap” throws an exception, pb (and the Widget it’s inside of) remains unchanged

3. another way is "copy and swap":

```cpp
class Widget { ...
void swap(Widget& rhs); // exchange *this’s and rhs’s data; see Item 29 for details };
Widget& Widget::operator=(const Widget& rhs) {
  Widget temp(rhs); // make a copy of rhs’s data
  swap(temp); 			// swap *this’s data with the copy’s
  return *this; 
}
```



## 12. Copy all parts of an object.

- Copying functions should be sure to copy all of an object’s data members and all of its base class parts
- Don’t try to implement one of the copying functions in terms of the other. Instead, put common functionality in a third function that both call.
  - It makes no sense to have the copy assignment operator call the copy constructor, because you’d be trying to construct an object that already exists. This is so nonsensical, there’s not even a syntax for it.
  -  having the copy constructor call the copy assignment operator — is equally nonsensical. A constructor initializes new objects, but an assignment operator applies only to objects that have already been initialized. Performing an assignment on an object under construction would mean doing something to a not-yet-initialized object that makes sense only for an initialized object. Nonsense! Don’t try it.

## 13. Use Objects to manage resources

1. Resources are acquired and immediately turned over to resource-managing objects (RAII)
2. Resource-managing objects use their destructors to ensure that resources are released

RAII对象：

3. auto_ptr 的行为类似unique_ptr,会转移owner。For example, STL containers require that their contents exhibit “normal” copying behavior, so containers of auto_ptr aren’t allowed.

4. reference-counting smart pointer* (RCSP).  RCSPs can’t break cycles of references (e.g., two otherwise unused objects that point to one another).
5. Both auto_ptr and tr1::shared_ptr use delete in their destructors, not delete []. (Item 16 describes the difference.) That means that using auto_ptr or tr1::shared_ptr with dynamically allocated arrays is a bad idea。（use vector or string）

## 14. Think carefully about copying behavior in resource-managing classes.

what should happen when an RAII object is copied? you will choose one of the following possibilities

1. Prohibit copying.

   When copying makes no sense for an RAII class, you should prohibit it (see item 6 for Uncopyable)

2. Reference-count the underlying resource (利用shared_ptr并提供自定义deleter，来实现自定义行为)

   ```cpp
   class Lock { 
     public:
   	explicit Lock(Mutex *pm) : mutexPtr(pm, unlock){
   		lock(mutexPtr.get());
   	} 
     private:
     	std::tr1::shared_ptr<Mutex> mutexPtr;
   };
   
   ```

3. Copy the underlying resource.

4. Transfer ownership of the underlying resource

## 15. Provide access to raw resources in resource-managing classes.

1. Explicit conversation
   1. get() member function
   2. explicit T()
2. Implicit conversation
   1. operator-> and operator *
   2. operator T() const;

Things to Remember

✦  APIs often require access to raw resources, so each RAII class should offer a way to get at the resource it manages.

✦  Access may be via explicit conversion or implicit conversion. In gen- eral, explicit conversion is safer, but implicit conversion is more con- venient for clients.

## 16.Use the same form in corresponding uses of new and delete.

Things to Remember

✦ If you use [] in a new expression, you must use [] in the correspond- ing delete expression. If you don’t use [] in a new expression, you mustn’t use [] in the corresponding delete expression.

## 17.Store newed objects in smart pointers in standalone statements.

```cpp
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());	
```

Before processWidget can be called, then, compilers must generate code to do these three things:

- Call priority.
- Execute “new Widget”.
- Call the tr1::shared_ptr constructor.

C++ compilers are granted considerable latitude in determining the order in which these things are to be done

the call to priority can be performed first, second, or third. If compilers choose to perform it second, we end up with this sequence of operations:

1. Execute “new Widget”.
2. Call priority.
3. Call the tr1::shared_ptr constructor.

if the call to priority yields an exception.A leak in the call to processWidget can arise because an exception can intervene between the time a resource is created (via “new Widget”) and the time that resource is turned over to a resource-managing object.

The way to avoid problems like this is simple: use a separate state- ment to create the Widget and store it in a smart pointer, then pass the smart pointer to processWidget:

```cpp
std::tr1::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

## 18.Make interfaces easy to use correctly and hard to use incorrectly.

Things to Remember

✦  Good interfaces are easy to use correctly and hard to use incorrectly. You should strive for these characteristics in all your interfaces.

✦  Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.

✦  Ways to prevent errors include creating new types, restricting opera- tions on types, constraining object values, and eliminating client re- source management responsibilities.

✦ tr1::shared_ptr supports custom deleters. This prevents the cross- DLL problem, can be used to automatically unlock mutexes (see Item 14), etc

##19. Treat class design as type design.

How do you design effective classes? Virtually every class requires that you confront the following questions, the answers to which often lead to constraints on your design:

1. How should objects of your new type be created and de- stroyed?

2. How should object initialization differ from object assign- ment?

3. What does it mean for objects of your new type to be passed by value?

   the copy constructor defines how pass-by-value is implemented for a type

4. What are the restrictions on legal values for your new type?

   您的新类型对合法值有哪些限制？ 通常，一个类的数据成员的值的某些组合是有效的。 这些组合决定了您的类必须保持的不变性。 不变性决定了您必须在成员函数内部进行的错误检查，尤其是构造函数，赋值运算符和“设置”函数。 它还可能会影响您的函数引发的异常，并在您偶然使用它们的情况下影响函数的异常规范。

5. Does your new type fit into an inheritance graph?

6. What kind of type conversions are allowed for your new type?

7. What operators and functions make sense for the new type?

8. What standard functions should be disallowed?

9. Who should have access to the members of your new type?

10. What is the “undeclared interface” of your new type?

11. How general is your new type?

12. Is a new type really what you need?

##  20.Prefer pass-by-reference-to-const to pass-by- value.

Things to Remember

✦ Prefer pass-by-reference-to-const over pass-by-value. It’s typically more efficient and it avoids the slicing problem.

✦ The rule doesn’t apply to built-in types and STL iterator and func- tion object types. For them, pass-by-value is usually appropriate.

## 21.Don’t try to return a reference when you must return an object.

Things to Remember

✦ Never return a pointer or reference to a local stack object, a reference to a heap-allocated object, or a pointer or reference to a local static object if there is a chance that more than one such object will be needed. (Item 4 provides an example of a design where returning a reference to a local static is reasonable, at least in single-threaded environments.)

## 22.Declare data members private.

Things to Remember

✦ Declare data members private. It gives clients syntactically uniform access to data, affords fine-grained access control, allows invariants to be enforced, and offers class authors implementation flexibility.

✦ protected is no more encapsulated than public.

## 23.Prefer non-member non-friend functions to member functions.

Putting all convenience functions in multiple header files — but one namespace — also means that clients can easily *extend* the set of convenience functions.

Things to Remember

✦ Prefer non-member non-friend functions to member functions. Doing so increases encapsulation, packaging flexibility, and functional extensibility

## 24.Declare non-member functions when type conversions should apply to all parameters.

```cpp
class Rational { 
public:
	Rational(int numerator = 0, int denominator = 1);
	int numerator() const; 
  int denominator() const;
private: ...
};

Rational oneHalf(1, 2);
result = oneHalf * 2; // fine (with non-explicit ctor)
result = 2 * oneHalf; // error! (even with non-explicit ctor)
```

It turns out that parameters are eligible for implicit type conversion *only if they are listed in the parameter list*. The implicit parameter cor- responding to the object on which the member function is invoked — the one this points to — is *never* eligible for implicit conversions

You’d still like to support mixed-mode arithmetic, however, and the way to do it is by now perhaps clear: make operator* a non-member function, thus allowing compilers to perform implicit type conversions on *all* arguments:

```cpp
class Rational { ...
};
const Rational operator*(const Rational& lhs, const Rational& rhs){
	return Rational(lhs.numerator() * rhs.numerator(),lhs.denominator() * rhs.denominator());
}
Rational oneFourth(1, 4); 
Rational result;
result = oneFourth * 2; // fine
result = 2 * oneFourth; // hooray, it works!
```

Whenever you can avoid friend functions, you should, because, much as in real life, friends are often more trouble than they’re worth. Sometimes friend- ship is warranted, of course, but the fact remains that just because a function shouldn’t be a member doesn’t automatically mean it should be a friend.

Things to Remember

✦ If you need type conversions on all parameters to a function (includ- ing the one that would otherwise be pointed to by the this pointer), the function must be a non-member.

## 25.Consider support for a non-throwing swap.

0. Std::swap:

```cpp
namespace std {
template<typename T> void swap(T& a, T& b)
{
	T temp(a); 
  a=b;
	b = temp;
} 
}//namespace std
```

the default swap implementation copy three times(use copy contructor or copy assignment), so for types that consisting primarily of a pointer to another type that contains the real data, the default swap is no efficient. For example:

```cpp
class WidgetImpl { 
public:
...
private:
int a, b, c; 
std::vector<double> v; //possibly lots of data —  expensive to copy!
  ...
};
class Widget { 
public:
	Widget(const Widget& rhs);
	Widget& operator=(const Widget& rhs) {
		*pImpl = *(rhs.pImpl);
  }
private:
  WidgetImpl *pImpl;
};
```

1. one way is to speciliaze std::swap for Widget:

```cpp
namespace std {
template<>
void swap<Widget>( Widget& a, Widget& b)
{
	swap(a.pImpl, b.pImpl); 
}
} // namespace std
```

In general, we’re not permitted to alter the contents of the std namespace, but we are allowed to totally specialize standard templates (like swap) for types of our own creation (such as Widget). That’s what we’re doing here.

but this function won’t compile because it’s trying to access the pImpl pointers inside a and b, and they’re private.

We could declare our specialization a friend, but the convention is different: it’s to have Widget declare a public member function called swap that does the actual swapping, then specialize std::swap to call the member function:

```cpp
class Widget { 
public:
void swap(Widget& other) {
	using std::swap;   //explain later
	swap(pImpl, other.pImpl); 
}
};

namespace std {
template<>
void swap<Widget>(Widget& a,Widget& b){
	a.swap(b);  
}
}//std
```

Not only does this compile, it’s also consistent with the STL containers

2. Suppose, however, that Widget and WidgetImpl were class *templates* instead of classes, possibly so we could parameterize the type of the data stored in WidgetImpl:

```cpp
template<typename T>
class WidgetImpl { ... };
template<typename T>
class Widget { ... };
```

we run into trouble with the specialization for std::swap. This is what we want to write:

```cpp
namespace std {
template<typename T>
void swap<Widget<T> >(Widget<T>& a,Widget<T>& b)// error! illegal code!
{ 
  a.swap(b); 
} 
}// std
```

as we’re trying to partially specialize a function template (std::swap), but though C++ allows partial specialization of class templates, it doesn’t allow it for function templates.

When you want to “partially specialize” a function template, the usual approach is to simply add an overload. That would look like this:

```cpp
namespace std {
template<typename T> 
void swap(Widget<T>& a, Widget<T>& b)
{ a.swap(b); }}								// an overloading of std::swap 
  														// (note the lack of “<...>” after 
  														// “swap”), but see below for 
  														// why this isn’t valid code
```

In general, overloading function templates is fine, but std is a special namespace, and the rules governing it are special, too. It’s okay to totally specialize templates in std, but it’s not okay to add *new* tem- plates (or classes or functions or anything else) to std. The contents of std are determined solely by the C++ standardization committee, and we’re prohibited from augmenting what they’ve decided should go there

3. So what to do? We still need a way to let other people call swap and get our more efficient template-specific version. The answer is simple. We still declare a non-member swap that calls the member swap, we just don’t declare the non-member to be a specialization or overloading of std::swap. For example, if all our Widget-related functionality is in the namespace WidgetStuff, it would look like this:

```cpp
namespace WidgetStuff { ...
// templatized WidgetImpl, etc.
// as before, including the swap member function
// non-member swap function not part of the std namespace
template<typename T> class Widget { ... };
...
template<typename T> void swap(Widget<T>& a, Widget<T>& b)
{
a.swap(b);
}
}
```

Now, if any code anywhere calls swap on two Widget objects, the name lookup rules in C++ (specifically the rules known as *argument-depen- dent lookup* or *Koenig lookup*) will find the Widget-specific version in WidgetStuff. Which is exactly what we want.

This approach works as well for classes as for class templates, so it seems like we should use it all the time. Unfortunately, there is a reason for specializing std::swap for classes (I’ll describe it shortly), so if you want to have your class-specific version of swap called in as many contexts as possible (and you do), you need to write both a non-member version in the same namespace as your class and a specialization of std::swap

4. from a client’s point of view.:

```cpp
template<typename T>
void doSomething(T& obj1, T& obj2) {
...
swap(obj1, obj2);
...
}
```

Which swap should this call? The general one in std, which you know exists; a specialization of the general one in std, which may or may not exist; or a T-specific one, which may or may not exist and which may or may not be in a namespace (but should certainly not be in std)? What you desire is to call a T-specific version if there is one, but to fall back on the general version in std if there’s not. Here’s how you fulfill your desire:

```cpp
template<typename T>
void doSomething(T& obj1, T& obj2) {
using std::swap;   // make std::swap available in this function
...
swap(obj1, obj2); // call the best swap for objects of type T
...
}
```

When compilers see the call to swap, they search for the right swap to invoke. C++’s name lookup rules ensure that this will find any T-spe- cific swap at global scope or in the same namespace as the type T. (For example, if T is Widget in the namespace WidgetStuff, compilers will use argument-dependent lookup to find swap in WidgetStuff.) If no T- specific swap exists, compilers will use swap in std, thanks to the using declaration that makes std::swap visible in this function. Even then, however, compilers will prefer a T-specific specialization of std::swap over the general template, so if std::swap has been specialized for T, the specialized version will be used.

5. Getting the right swap called is therefore easy. The one thing you want to be careful of is to not qualify the call, because that will affect how C++ determines the function to invoke. For example, if you were to write the call to swap this way,

```
std::swap(obj1, obj2);
```

that’s why it’s important to totally specialize std::swap for your classes: it makes type-specific swap implementations available to code written in this misguided fashion.

6. conclusion:

* First, if the default implementation of swap offers acceptable efficiency for your class or class template, you don’t need to do anything. Any- body trying to swap objects of your type will get the default version, and that will work fine.

* Second, if the default implementation of swap isn’t efficient enough (which almost always means that your class or template is using some variation of the pimpl idiom), do the following:

  1. Offer a public swap member function that efficiently swaps the value of two objects of your type. For reasons I’ll explain in a moment, this function should never throw an exception.

  2. Offer a non-member swap in the same namespace as your class or template. Have it call your swap member function.

  3. If you’re writing a class (not a class template), specialize std::swap for your class. Have it also call your swap member function.

* Finally, if you’re calling swap, be sure to include a using declaration to make std::swap visible in your function, then call swap without any namespace qualification.

7. Have the member version of swap never throw exceptions. This constraint applies only to the member version! It can’t apply to the non-member version, because the default version of swap is based on copy construction and copy assignment, and, in general, both of those functions are allowed to throw exceptions. 

## 26. Postpone variable definitions as long as possible.

Not only should you postpone a variable’s definition until right before you have to use the variable, you should also try to postpone the definition until you have initialization arguments for it.

```cpp
// Approach A: define outside loop
Widget w;
for (int i = 0; i < n; ++i) {
	w = some value dependent on i;
}
// Approach B: define inside loop 
for (int i = 0; i < n; ++i) {
	Widget w(some value dependent on i);
}
```

unless you know that (1) assignment is less expensive than a constructor-destructor pair and (2) you’re dealing with a performance-sensitive part of your code, you should default to using Approach B.

## 27 Minimize casting

C-style casts look like this:

1. (T) expression // cast expression to be of type T
2. T(expression) //  cast expression to be of type T

C++ also offers four new cast forms

```cpp
const_cast<T>(expression)
dynamic_cast<T>(expression)
reinterpret_cast<T>(expression) 
static_cast<T>(expression)
```

- const_cast is typically used to cast away the constness of objects. It is the only C++-style cast that can do this.
- dynamic_cast is primarily used to perform “safe downcasting,” i.e., to determine whether an object is of a particular type in an inheritance hierarchy. It is the only cast that cannot be performed using the old-style syntax. It is also the only cast that may have a significant runtime cost. (I’ll provide details on this a bit later.)
- reinterpret_cast is intended for low-level casts that yield implementation-dependent (i.e., unportable) results, e.g., casting a pointer to an int. Such casts should be rare outside low-level code. I use it only once in this book, and that’s only when discussing how you might write a debugging allocator for raw memory (see Item 50).
- static_cast can be used to force implicit conversions (e.g., non-const object to const object (as in Item 3), int to double, etc.). It can also be used to perform the reverse of many such conversions (e.g., void* pointers to typed pointers, pointer-to-base to pointer-to-derived), though it cannot cast from const to non-const objects. (Only const_cast can do that.)

Many programmers believe that casts do nothing but tell compilers to treat one type as another, but this is mistaken. Type conversions of any kind (either explicit via casts or implicit by compilers) often lead to code that is executed at runtime:

```cpp
int x, y;
double d = static_cast<double>(x)/y;
```

the cast of the int x to a double almost certainly generates code, because on most architectures, the underlying representation for an int is different from that for a double

```cpp
class Base { ... };
class Derived: public Base { ... };
Derived d;
Base*pb=&d;
```

Here we’re just creating a base class pointer to a derived class object, but sometimes, the two pointer values will not be the same. When that’s the case, an offset is applied *at runtime* to the Derived* pointer to get the correct Base* pointer value.

An interesting thing about casts is that it’s easy to write something that looks right but is wrong:

```cpp
class Window { 
public:
	virtual void onResize() { ... }
	...
};
class SpecialWindow: public Window { 
public:
	virtual void onResize() { 
    static_cast<Window>(*this).onResize();
  }
};
```

the cast creates a new, temporary *copy* of the base class part of *this, then invokes onResize on the copy!

The solution is to eliminate the cast, replacing it with what you really want to say:

```cpp
class SpecialWindow: public Window { 
public:
	virtual void onResize() {
		Window::onResize(); // call Window::onResize on *this
	}
};
```

Things to Remember

✦  Avoid casts whenever practical, especially dynamic_casts in performance-sensitive code. If a design requires casting, try to develop a cast-free alternative.

✦  When casting is necessary, try to hide it inside a function. Clients can then call the function instead of putting casts in their own code.

✦  Prefer C++-style casts to old-style casts. They are easier to see, and they are more specific about what they do.

## 28. Avoid returning “handles” to object internals.

Things to Remember

Avoid returning handles (references, pointers, or iterators) to object internals. Not returning handles increases encapsulation, helps const member functions act const, and minimizes the creation of dangling handles.

## 29.Strive for exception-safe code.

```cpp
class PrettyMenu { 
public:
	void changeBackground(std::istream& imgSrc);
private:
	Mutex mutex;
	Image *bgImage; 
  int imageChanges;
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
  lock(&mutex);
	delete bgImage; 
  ++imageChanges;
	bgImage = new Image(imgSrc);
	unlock(&mutex);
}
```

There are two requirements for exception safety, and this satisfies neither.

1. **Leak no resources.** The code above fails this test, because if the “new Image(imgSrc)” expression yields an exception, the call to unlock never gets executed, and the mutex is held forever.
2. **Don’t allow data structures to become corrupted.** If “new Im- age(imgSrc)” throws, bgImage is left pointing to a deleted object. In addition, imageChanges has been incremented, even though it’s not true that a new image has been installed. 

Addressing the resource leak issue is easy, because Item 13 explains how to use objects to manage resources:

```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc) {
	Lock ml(&mutex);
	delete bgImage; 
  ++imageChanges;
	bgImage = new Image(imgSrc);
}
```

Exception-safe functions offer one of three guarantees:

- Functions offering **the basic guarantee** promise that if an exception is thrown, everything in the program remains in a valid state. No objects or data structures become corrupted, and all objects are in an internally consistent state (e.g., all class invariants are satisfied). However, the exact state of the program may not be pre- dictable
- Functions offering **the strong guarantee** promise that if an excep- tion is thrown, the state of the program is unchanged. Calls to such functions are *atomic* in the sense that if they succeed, they succeed completely, and if they fail, the program state is as if they’d never been called.
- Functions offering **the nothrow guarantee** promise never to throw exceptions, because they always do what they promise to do. All operations on built-in types (e.g., ints, pointers, etc.) are no- throw (i.e., offer the nothrow guarantee). This is a critical building block of exception-safe code.

Exception-safe code must offer one of the three guarantees above. If it doesn’t, it’s not exception-safe。Offer the nothrow guarantee when you can, but for most functions, the choice is between the basic and strong guarantees.

```cpp
class PrettyMenu { 
	std::tr1::shared_ptr<Image> bgImage;
};
void PrettyMenu::changeBackground(std::istream& imgSrc) {
	Lock ml(&mutex);
	bgImage.reset(new Image(imgSrc)); 
  ++imageChanges;
}
```

the tr1::shared_ptr::reset function will be called only if its parameter (the result of “new Image(imgSrc)”) is successfully created. delete is used only inside the call to reset, so if the function is never entered, delete is never used

those two changes *almost* suffice to allow changeBackground to offer the strong exception safety guarantee. What’s the fly in the ointment? The parameter imgSrc.If the Image constructor throws an exception, it’s possible that the read marker for the input stream has been moved, and such movement would be a change in state visible to the rest of the program. Until changeBackground addresses that issue, it offers only the basic exception safety guarantee

There is a general design strategy that typically leads to the strong guarantee, and it’s important to be familiar with it. The strategy is known as “copy and swap.” In principle, it’s very simple. Make a copy of the object you want to modify, then make all needed changes to the copy. If any of the modifying operations throws an exception, the original object remains unchanged. After all the changes have been successfully completed, swap the modified object with the original in a non- throwing operation 

"copy and swap " like this:

```cpp
struct PMImpl { 
  std::tr1::shared_ptr<Image> bgImage; 
  int imageChanges;
};
class PrettyMenu {
private:
	Mutex mutex; 
  std::tr1::shared_ptr<PMImpl> pImpl;
};
void PrettyMenu::changeBackground(std::istream& imgSrc) {
	using std::swap;
	Lock ml(&mutex);
	std::tr1::shared_ptr<PMImpl> pNew(new PMImpl(*pImpl));
	pNew->bgImage.reset(new Image(imgSrc)); ++pNew->imageChanges;
	swap(pImpl, pNew);
}
```

The copy-and-swap strategy is an excellent way to make all-or-nothing changes to an object’s state, but, in general, it doesn’t guarantee that the overall function is strongly exception-safe

```cpp
void someFunc() {
	... // make copy of local state
	f1( ); 
  f2( );
	... // swap modified state into place
}
```

It should be clear that if f1 or f2 is less than strongly exception-safe, it will be hard for someFunc to be strongly exception-safe. For example, suppose that f1 offers only the basic guarantee. For someFunc to offer the strong guarantee, it would have to write code to determine the state of the entire program prior to calling f1, catch all exceptions from f1, then restore the original state.

Things aren’t really any better if both f1 and f2 *are* strongly exception safe. After all, if f1 runs to completion, the state of the program may have changed in arbitrary ways, so if f2 then throws an exception, the state of the program is not the same as it was when someFunc was called, even though f2 didn’t change anything.

Issues such as these can prevent you from offering the strong guaran- tee for a function, even though you’d like to. Another issue is effi- ciency. The crux of copy-and-swap is the idea of modifying a copy of an object’s data, then swapping the modified data for the original in a non-throwing operation. 

For many functions, the basic guarantee is a perfectly reasonable choice

If the functions someFunc calls offer no exception-safety guarantees, someFunc itself can’t offer any guarantees.

Document your decisions, both for clients of your functions and for future main- tainers. A function’s exception-safety guarantee is a visible part of its interface, so you should choose it as deliberately as you choose all other aspects of a function’s interface.

Things to Remember

✦  Exception-safe functions leak no resources and allow no data struc- tures to become corrupted, even when exceptions are thrown. Such functions offer the basic, strong, or nothrow guarantees.

✦  The strong guarantee can often be implemented via copy-and-swap, but the strong guarantee is not practical for all functions.

✦  A function can usually offer a guarantee no stronger than the weak- est guarantee of the functions it calls.

## 30.Understand the ins and outs of inlining.

1. For inline function, you actually get more than you might think, because avoiding the cost of a function call is only part of the story. Compiler optimizations are typically designed for stretches of code that lack function calls, so when you inline a function, you may enable compilers to perform context-specific optimizations on the body of the function. Most compilers never perform such optimizations on “outlined” function calls.

2. The idea behind an inline function is to replace each call of that function with its code body,  this is likely to increase the size of your object code. It can lead to additional paging, a reduced instruction cache hit rate, and the performance penalties that accompany these things.

3. On the other hand, if an inline function body is *very* short, the code generated for the function body may be smaller than the code generated for a function call. If that is the case, inlining the function may actually lead to *smaller* object code and a higher instruction cache hit rate!

4. Bear in mind that inline is a *request* to compilers, not a command. The request can be given implicitly or explicitly. The implicit way is to define a function inside a class definition:

```cpp
class Person { 
  public:
		int age() const { return theAge; } ...
	private:
		int theAge;
};
```

friend functions can also be defined inside classes. When they are, they’re also implicitly declared inline

5. The explicit way to declare an inline function is to precede its definition with the inline keyword:

```cpp
template<typename T> 															// an explicit inline 
inline const T& std::max(const T& a, const T& b) // request: std::max is 
{
  return a < b ? b : a;																//precededby“inline”
} 																
```

both inline functions and templates are typically defined in header files. This leads some programmers to conclude that function templates must be inline. This conclusion is both invalid and potentially harmful, so it’s worth looking into it a bit.

1. Inline functions must typically be in header files, because most build environments do inlining during compilation. In order to replace a function call with the body of the called function, compilers must know what the function looks like.
2. Templates are typically in header files, because compilers need to know what a template looks like in order to instantiate it when it’s used.

Template instantiation is independent of inlining. If you’re writing a template and you believe that all the functions instantiated from the template should be inlined, declare the template inline; that’s what’s done with the std::max implementation above. But if you’re writing a template for functions that you have no reason to want inlined, avoid declaring the template inline (either explicitly or implicitly). 

6. Most compilers refuse to inline functions they deem too complicated。(e.g., those that contain loops or are recursive), and all but the most trivial calls to virtual functions defy inlining. 

Sometimes compilers generate a function body for an inline function even when they are perfectly willing to inline the function. For example, if your program takes the address of an inline function, compilers must typically generate an outlined function body for it. compilers typically don’t perform inlining across calls through function pointers, this means that calls to an inline function may or may not be inlined, depending on how the calls are made:

```cpp
inline void f() {...} 
void (*pf)() = f; 
f();// this call will be inlined, because it’s a “normal” call
pf(); // this call probably won’t be, because it’s through // a function pointer
```

即使您从不使用函数指针，非内联函数的幽灵也会困扰您，因为程序员不一定是唯一要求函数指针的人。 有时，编译器会生成构造函数和析构函数的脱机副本，以便它们可以获取指向这些函数的指针，以便在构造和破坏数组对象时使用

7. In fact, constructors and destructors are often worse candidates for inlining than a casual examination would indicate.

```cpp
 class Base { 
   public:
	 private:
		 std::string bm1, bm2;
};
class Derived: public Base { 
  public:
		Derived() {}
	private:
		std::string dm1, dm2, dm3;
};

//we can imagine implementations generating code equivalent to the follow- ing for the allegedly empty Derived constructor above
Derived::Derived( ) {
  Base::Base( );
	try { 
    dm1.std::string::string(); 
  } catch (...) {
		Base::~Base( ); throw;
	}
	try { 
    dm2.std::string::string(); 
  } catch(...) {
		dm1.std::string::~string( ); 
    Base::~Base( );
		throw;
	}
	try { 
    dm3.std::string::string(); 
  } catch(...) {
		dm2.std::string::~string( ); 
    dm1.std::string::~string( ); 
    Base::~Base( );
		throw;
  }
}
```

The same reasoning applies to the Base constructor, so if it’s inlined, all the code inserted into it is also inserted into the Derived constructor (via the Derived constructor’s call to the Base constructor). And if the string constructor also happens to be inlined, the Derived constructor will gain *five copies* of that function’s code, one for each of the five strings in a Derived object (the two it inherits plus the three it declares itself). Perhaps now it’s clear why it’s not a no-brain decision whether to inline Derived’s constructor. Similar considerations apply to Derived’s destructor, which, one way or another, must see to it that all the objects initialized by Derived’s constructor are properly destroyed.

8. Library designers must evaluate the impact of declaring functions inline, because it’s impossible to provide binary upgrades to the client-visible inline functions in a library. In other words, if f is an inline function in a library, clients of the library compile the body of f into their applications. If a library implementer later decides to change f, all clients who’ve used f must recompile. This is often undesirable. On the other hand, if f is a non-inline function, a modification to f requires only that clients relink. This is a substantially less onerous burden than recompiling and, if the library containing the function is dynamically linked, one that may be absorbed in a way that’s com- pletely transparent to clients.	

9. most debuggers have trouble with inline functions. This should be no great revelation. How do you set a breakpoint in a function that isn’t there? Although some build envi- ronments manage to support debugging of inlined functions, many environments simply disable inlining for debug builds.

10. This leads to a logical strategy for determining which functions should be declared inline and which should not. Initially, don’t inline any- thing, or at least limit your inlining to those functions that must be inline (see Item 46) or are truly trivial

Things to Remember

- ✦  Limit most inlining to small, frequently called functions. This facili- tates debugging and binary upgradability, minimizes potential code bloat, and maximizes the chances of greater program speed.
- ✦  Don’t declare function templates inline just because they appear in header files.

## 31.Minimize compilation dependencies between files.

using the pimpl idiom(pointer to implementation): separate implementation and interface

```cpp
#include <string>
#include <memory> 

class PersonImpl;
class Date; 
class Address;
class Person { 
public:
	Person(const std::string& name, const Date& birthday, const Address& addr);
	std::string name() const; 
	std::string birthDate() const; 
	std::string address() const; ...
private:
	std::tr1::shared_ptr<PersonImpl> pImpl;
};
```

The key to this separation is replacement of dependencies on *definitions* with dependencies on *declarations*. 

That’s the essence of minimizing compilation dependencies: make your header files self-sufficient when- ever it’s practical, and when it’s not, depend on declarations in other files, not definitions。

1. Avoid using objects when object references and pointers will do

   You may define references and pointers to a type with only a declaration for the type. Defining *objects* of a type necessitates the presence of the type’s definition.

2. Depend on class declarations instead of class definitions whenever you can

   Note that you *never* need a class definition to declare a function using that class, not even if the function passes or returns the class type by value:

   ```cpp
   class Date;// class declaration
   
   Date today();										// fine no definition  of Date is needed
   void clearAppointments(Date d); // fine no definition  of Date is needed
   ```

3. Provide separate header files for declarations and definitions.

   In order to facilitate adherence to the above guidelines, header files need to come in pairs: one for declarations, the other for defi- nitions.As a result, library clients should always #include a declaration file in- stead of forward-declaring something themselves, and library au- thors should provide both header files：

   ```cpp
   #include "datefwd.h" // header file declaring (but not // defining) class Date
   Date today(); // as before void clearAppointments(Date d);
   ```

   The name of the declaration-only header file “datefwd.h” is based on the header <iosfwd> from the standard C++ library

4. Handle classes and Interface classes decouple interfaces from imple- mentations, thereby reducing compilation dependencies between files.

Things to Remember

✦  The general idea behind minimizing compilation dependencies is to depend on declarations instead of definitions. Two approaches based on this idea are Handle classes and Interface classes.

✦  Library header files should exist in full and declaration-only forms. This applies regardless of whether templates are involved.

## 32. Make sure public inheritance models “is-a.”

Things to Remember

✦ Public inheritance means “is-a.” Everything that applies to base classes must also apply to derived classes, because every derived class object *is* a base class object.

## 33. Avoid hiding inherited names.

The matter actually has nothing to do with inheritance. It has to do with scopes. 	

```cpp
int x;													// global variable
void someFunc() {
	double x;											// local variable
	std::cin >> x;								// read a new value for local x
}
```

names in inner scopes hide (“shadow”) names in outer scopes. it regardness type, just hide names.

The way that actually works is that the scope of a derived class is nested inside its base class’s scope.

```cpp
class Base { 
private:
	int x;
public:
	virtual void mf1() = 0; 
  virtual void mf1(int);
	virtual void mf2();
	void mf3();
	void mf3(double);
};
class Derived: public Base { 
public:
	virtual void mf1();
	void mf3();
	void mf4();
};
```

```cpp
Derived d; 
int x;
d.mf1();// fine, calls Derived::mf1
d.mf1(x); // error! Derived::mf1 hides Base::mf1
d.mf2();// fine, calls Base::mf2
d.mf3(); // fine, calls Derived::mf3
d.mf3(x);// error! Derived::mf3 hides Base::mf3
```

As you can see, this applies even though the functions in the base and derived classes take different parameter types, and it also applies regardless of whether the functions are virtual or non-virtual. In the same way that, at the beginning of this Item, the double x in the function someFunc hides the int x at global scope, here the function mf3 in Derived hides a Base function named mf3 that has a different type.

With using *declarations*:

```cpp
class Base { 
private:
	int x;
public:
	virtual void mf1() = 0; 
  virtual void mf1(int);
	virtual void mf2();
	void mf3();
	void mf3(double);
};

class Derived: public Base { 
public:
	using Base::mf1; // make all things in Base named mf1 and mf3
  using Base::mf3; // visible (and public) in Derived’s scope
	virtual void mf1(); 
  void mf3();
	void mf4();
};

Derived d; 
int x;
d.mf1( ); // still fine, still calls Derived::mf1 
d.mf1(x);// now okay, calls Base::mf1
d.mf2( );// still fine, still calls Base::mf2
d.mf3( ); // fine, calls Derived::mf3
d.mf3(x);// now okay, calls Base::mf3 (The int x isimplicitly converted to a double so that the call to Base::mf3 is valid.
```

This means that if you inherit from a base class with overloaded func- tions and you want to redefine or override only some of them, you need to include a using declaration for each name you’d otherwise be hiding. If you don’t, some of the names you’d like to inherit will be hidden.

private inheritend, inline forwarding function:

```cpp
class Base { 
public:
	virtual void mf1() = 0; 
  virtual void mf1(int);
}
class Derived: private Base { 
public:
	virtual void mf1() { 
    Base::mf1(); 
  }
};
Derived d; 
int x;
d.mf1(); // fine, calls Derived::mf1
d.mf1(x);// error! Base::mf1() is hidden
```

Another use for inline forwarding functions is to work around ancient compilers that (incorrectly) don’t support using declarations to import inherited names into the scope of a derived class

Things to Remember

✦ Names in derived classes hide names in base classes. Under public inheritance, this is never desirable.

✦ To make hidden names visible again, employ using declarations or forwarding functions.

内层空间中的名称会覆盖外层的，如果有多个名称，互相之间没有覆盖，且都可见，会导致二义性问题。

## 34. Differentiate between inheritance of interface and inheritance of implementation.

As a class designer, 

1. you sometimes want derived classes to inherit only the interface (declaration) of a member function. 
2. Sometimes you want derived classes to inherit both a function’s interface and imple- mentation, but you want to allow them to override the implementation they inherit.
3.  And sometimes you want derived classes to inherit a function’s interface and implementation without allowing them to override anything

```cpp
class Shape { 
public:
	virtual void draw() const = 0;
	virtual void error(const std::string& msg);
	int objectID() const;
};
class Rectangle: public Shape { ... }; 
class Ellipse: public Shape { ... };
```

Member function *interfaces are always inherited*.

1. The purpose of declaring a pure virtual function is to have derived classes inherit a function *interface only*.

实验结论：不管纯虚函数有没有实现，子类必须实现纯虚函数，否则也是一个纯虚类，无法构建对象。因此，必须通过带限定的方式调用基类的纯虚函数。虚函数必须有实现，因为编译器需要为虚表填充函数指针。非虚函数如果没有调用到，可以不实现。

Incidentally, it *is* possible to provide a definition for a pure virtual function. That is, you could provide an implementation for Shape::draw, and C++ wouldn’t complain, but the only way to call it would be to qualify the call with the class name:

```cpp
Shape *ps = new Shape;				// error! Shape is abstract
Shape *ps1 = new Rectangle; 	// fine
ps1->draw();									// calls Rectangle::draw
Shape *ps2 = new Ellipse; 		// fine
ps2->draw();									// calls Ellipse::draw
ps1->Shape::draw();						// calls Shape::draw
ps2->Shape::draw(); 					// calls Shape::draw
```

2. The purpose of declaring a simple virtual function is to have de- rived classes inherit a function *interface as well as a default imple- mentation.*

   It turns out that it can be dangerous to allow simple virtual functions to specify both a function interface and a default implementation. 因为子类可能忘记定义而使用了基类的实现。

3. The purpose of declaring a non-virtual function is to have derived classes inherit a function *interface as well as a mandatory implementation*.

Things to Remember

✦  Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes always inherit base class interfaces.

✦  Pure virtual functions specify inheritance of interface only.

✦  Simple (impure) virtual functions specify inheritance of interface plus inheritance of a default implementation.

✦  Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation.

## 35. Consider alternatives to virtual functions

```cpp
class GameCharacter { 
public:
	virtual int healthValue() const;
};
```

1. The Template Method Pattern via the Non-Virtual Interface Idiom

   ```cpp
   class GameCharacter { 
   public:
   	int healthValue() const {
       ...
   		int retVal = doHealthValue();
       ...
   		return retVal; 
     }
   private:
   	virtual int doHealthValue() const {
   		...
   	}
   }
   ```

   1. This basic design — having clients call private virtual functions indirectly through public non-virtual member functions — is known as the *non-virtual interface (NVI) idiom*.

   2. An advantage of the NVI idiom is suggested by the “do ‘before’ stuff” and “do ‘after’ stuff” comments in the code.

   3. It may have crossed your mind that the NVI idiom involves derived classes redefining private virtual functions — redefining functions they can’t call! There’s no design contradiction here. Redefining a virtual function specifies *how* something is to be done. Calling a virtual function specifies *when* it will be done. These concerns are independent. The NVI idiom allows derived classes to redefine a virtual function, thus giving them control over *how* functionality is implemented, but the base class reserves for itself the right to say *when* the function will be called. It may seem odd at first, but C++’s rule that derived classes may redefine private inherited virtual functions is perfectly sensible.

   Under the NVI idiom, it’s not strictly necessary that the virtual functions be private. In some class hierarchies, derived class implementations of a virtual function are expected to invoke their base class counterparts, and for such calls to be legal, the virtuals must be protected, not private. Sometimes a virtual function even has to be public (e.g., destructors in polymorphic base classes — see Item 7), but then the NVI idiom can’t really be applied.

2. The Strategy Pattern via Function Pointers

   ```cpp
   class GameCharacter; // forward declaration
   // function for the default health calculation algorithm 
   int defaultHealthCalc(const GameCharacter& gc);
   class GameCharacter { 
   public:
   	typedef int (*HealthCalcFunc)(const GameCharacter&);
   	explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc) : healthFunc(hcf ){}
   	int healthValue() const{ 
       return healthFunc(*this); 
     }
   private:
   	HealthCalcFunc healthFunc;
   };
   ```

   it offers some interesting flexibility:

   1. Different instances of the same character type can have different health calculation functions
   2. Health calculation functions for a particular character may be changed at runtime

   but it has no special access to the internal parts of the object, as it's not member function.

   As a general rule, the only way to resolve the need for non-member functions to have access to non-public parts of a class is to weaken the class’s encapsulation.that is something you must decide on a design-by-design basis.

3. The Strategy Pattern via tr1::function

   ```cpp
   class GameCharacter; // as before 
   int defaultHealthCalc(const GameCharacter& gc); // as before
   class GameCharacter { 
   public:
   // HealthCalcFunc is any callable entity that can be called with
   // anything compatible with a GameCharacter and that returns anything 
   // compatible with an int; see below for details
   typedef std::tr1::function<int (const GameCharacter&)> HealthCalcFunc;
   explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc) : healthFunc(hcf ) {}
   int healthValue() const { 
     return healthFunc(*this); 
   }
   private:
   	HealthCalcFunc healthFunc;
   };
   ```

   more flexibility in specifying health calculation functions: allowing clients to use *any compatible callable entity* when calculating a character’s health.

4. The “Classic” Strategy Pattern

   ![image-20200718201859450](/Users/markfqwu/Library/Application Support/typora-user-images/image-20200718201859450.png)

   ```cpp
   class GameCharacter;
   class HealthCalcFunc { 
   public:
   virtual int calc(const GameCharacter& gc) const {...}
   };
   HealthCalcFunc defaultHealthCalc;
   
   class GameCharacter { 
   public:
   explicit GameCharacter(HealthCalcFunc *phcf = &defaultHealthCalc) : pHealthCalc(phcf ){}
   int healthValue() const{ 
     return pHealthCalc->calc(*this); 
   }
   private:
   	HealthCalcFunc *pHealthCalc; };
   ```



## 36. Never redefine an inherited non-virtual function.

```cpp
class B { 
public:
	void mf();
};
class D: public B { 
	void mf();
};
D x;
B*pB=&x;
pB->mf();
D*pD=&x;
pD->mf();
```

 *non-virtual* functions are statically bound, so the determining factor is the declared type of the pointer that points to it. 

theoretical justification:

Item 32 explains that public inheritance means is-a, and Item 34 describes why declaring a non-virtual function in a class establishes an invariant over specialization for that class. If you apply these observations to the classes B and D and to the non-virtual member function B::mf, then:

1. Everything that applies to B objects also applies to D objects, because every D object is-a B object;
2. Classes derived from B must inherit both the interface *and* the implementation of mf, because mf is non-virtual in B.

Now, if D redefines mf, there is a contradiction in your design. If D *really* needs to implement mf differently from B, and if every B object — no matter how specialized — *really* has to use the B implementation for mf, then it’s simply not true that every D is-a B. In that case, D shouldn’t publicly inherit from B. On the other hand, if D *really* has to publicly inherit from B, and if D *really* needs to implement mf differ- ently from B, then it’s just not true that mf reflects an invariant over specialization for B. In that case, mf should be virtual. Finally, if every D *really* is-a B, and if mf really corresponds to an invariant over spe- cialization for B, then D can’t honestly need to redefine mf, and it shouldn’t try to.

## 37. Never redefine a function’s inherited default parameter value.

There are only two kinds of functions you can inherit: virtual and non-virtual. However, it’s always a mistake to redefine an inherited non-virtual function (see Item 36), so we can safely limit our discussion here to the situation in which you inherit a *virtual* function with a default parameter value

That being the case, the justification for this Item becomes quite straightforward: virtual functions are dynamically bound, but default parameter values are statically bound

```cpp
// a class for geometric shapes 
class Shape {
public:
	enum ShapeColor { Red, Green, Blue };
	// all shapes must offer a function to draw themselves 
	virtual void draw(ShapeColor color = Red) const = 0;
};
class Rectangle: public Shape { 
public:
	// notice the different default parameter value — bad! 
  virtual void draw(ShapeColor color = Green) const;
};
class Circle: public Shape { 
public:
	virtual void draw(ShapeColor color) const;
};
```

```cpp
Shape *ps;											// static type = Shape*
Shape *pc = new Circle; 				// static type = Shape*
Shape *pr = new Rectangle;			// static type = Shape*

pc->draw(Shape::Red); // calls Circle::draw(Shape::Red)
pr->draw(Shape::Red); // calls Rectangle::draw(Shape::Red)

```

That means you may end up invoking a virtual function defined in a *derived class* but using a default parameter value from a *base class*.

The result is a call consisting of a strange and almost certainly unanticipated combination of the declarations for draw in both the Shape and Rectangle classes.

Why does C++ insist on acting in this perverse manner? The answer has to do with runtime efficiency. If default parameter values were dynamically bound, compilers would have to come up with a way to determine the appropriate default value(s) for parameters of virtual functions at runtime, which would be slower and more complicated than the current mechanism of determining them during compilation.

When you’re having trouble making a virtual function behave the way you’d like, it’s wise to consider alternative designs, and Item 35 is filled with alternatives to virtual functions.

Things to Remember

✦ Never redefine an inherited default parameter value, because default parameter values are statically bound, while virtual functions — the only functions you should be redefining — are dynamically bound

## 38. Model “has-a” or “is-implemented-in-terms-of” through composition.

```cpp
class Address { ... };
class PhoneNumber { ... };
class Person { 
public:
private:
	std::string name;
	Address address; 
	PhoneNumber voiceNumber; 
  PhoneNumber faxNumber;
};
```

Composition means either “has-a” or “is-implemented-in-terms-of.” That’s because you are dealing with two different domains in your software. Some objects in your programs correspond to things in the world you are modeling, e.g., people, vehicles, video frames, etc. Such objects are part of the *application domain*. Other objects are purely implementa- tion artifacts, e.g., buffers, mutexes, search trees, etc. These kinds of objects correspond to your software’s *implementation domain*. When composition occurs between objects in the application domain, it expresses a has-a relationship. When it occurs in the implementation domain, it expresses an is-implemented-in-terms-of relationship.

Things to Remember

✦ Composition has meanings completely different from that of public inheritance.

✦ In the application domain, composition means has-a. In the imple- mentation domain, it means is-implemented-in-terms-of.

## 39. Use private inheritance judiciously.

```cpp
class Person { ... };
class Student: private Person { ... };
void eat(const Person& p);
void study(const Student& s);
Person p; 
Student s;
eat(p);
eat(s);    //error! a Student isn’t a Person
```

Clearly, private inheritance doesn’t mean is-a.

private inheritence behave:

1. the first rule governing private inheritance you’ve just seen in action:compilers will generally *not* convert a derived class object (such as Student) into a base class object (such as Person) if the inheritance relationship between the classes is private. That’s why the call to eat fails for the object s. 
2. The second rule is that members inherited from a private base class become private members of the derived class, even if they were protected or public in the base class.

Private inheritance means is-implemented-in-terms-of. If you make a class D privately inherit from a class B, you do so because you are interested in taking advantage of some of the features available in class B, not because there is any conceptual relationship between objects of types B and D. As such, private inheritance is purely an implementation technique.(That’s why everything you inherit from a private base class becomes private in your class: it’s all just implementation detail.) Using the terms introduced in Item 34, private inheritance means that implementation *only* should be inherited; interface should be ignored. If D privately inherits from B, it means that D objects are implemented in terms of B objects, nothing more. Private inheritance means nothing during software *design*, only during software *implementation*.

use composition whenever you can, and use private inheritance whenever you must. When must you? Primarily when protected members and/or virtual functions enter the picture, though there’s also an edge case where space concerns can tip the scales toward private inheritance. We’ll worry about the edge case later. After all, it’s an edge case(EBO, empty base optimization).

```cpp
class Timer { 
public:
	explicit Timer(int tickFrequency); 
  virtual void onTick() const;
};
```

Use private inheritance:

```cpp
class Widget: private Timer { 
private:
	virtual void onTick() const;
};
```

Use public inheritence and composition:

```cpp
class Widget { 
private:
	class WidgetTimer: public Timer {
	public:
		virtual void onTick() const;
	};
	WidgetTimer timer;
};
```

two reasons why you might prefer public inheritance plus composition over private inheritance.

1. First, you might want to design Widget to allow for derived classes, but you might also want to prevent derived classes from redefining onTick. If Widget inherits from Timer, that’s not possible, not even if the inheritance is private.see Item 35.But if WidgetTimer is private in Widget and inherits from Timer, Widget’s derived classes have no access to WidgetTimer, hence can’t inherit from it or redefine its virtual functions
2. Second, you might want to minimize Widget’s compilation dependen- cies. If Widget inherits from Timer, Timer’s definition must be available when Widget is compiled, so the file defining Widget probably has to #include Timer.h. On the other hand, if WidgetTimer is moved out of Wid- get and Widget contains only a pointer to a WidgetTimer, Widget can get by with a simple declaration for the WidgetTimer class; it need not #include anything to do with Timer. For large systems, such decou- plings can be important,

Things to Remember

✦  Private inheritance means is-implemented-in-terms of. It’s usually inferior to composition, but it makes sense when a derived class needs access to protected base class members or needs to redefine inherited virtual functions.

✦  Unlike composition, private inheritance can enable the empty base optimization. This can be important for library developers who strive to minimize object sizes.

## 40. Use multiple inheritance judiciously.

1. MI leads to new opportunities for ambiguity

```cpp
class BorrowableItem { 
public:
	void checkOut();
};
class ElectronicGadget {
private:
	bool checkOut() const; ...
};
class MP3Player: public BorrowableItem, public ElectronicGadget{};

MP3Player mp; 
mp.checkOut();    //ambiguous! which checkOut?
```

Note that in this example, the call to checkOut is ambiguous, even though only one of the two functions is accessible.That’s in accord with the C++ rules for resolving calls to overloaded functions: before seeing whether a function is accessible, C++ first identifies the function that’s the best match for the call. **It checks accessibility only after finding the best-match function**. In this case, the name checkOut is ambiguous during name lookup, so neither function overload resolution nor best match determination takes place. The accessibility of ElectronicGadget::checkOut is therefore never examined.

2. deadly MI diamond:

```cpp
class File {}; 
class InputFile: public File {};
class OutputFile: public File {};
class IOFile: public InputFile, public OutputFile{};
```

For base class data member, It happily supports both options, though its default is to perform the replication. If that’s not what you want, you must make the class with the data  a *virtual base class*. To do that, you have all classes that immediately inherit from it use *virtual inheritance*:

```cpp
class File { ... };
class InputFile: virtual public File { ... }; 
class OutputFile: virtual public File { ... }; 
class IOFile: public InputFile,public OutputFile{...};
```

Avoiding the replication of inherited fields requires some behind-the-scenes legerdemain on the part of compilers, and the result is that objects created from classes using virtual inheritance are generally larger than they would be without virtual inheritance. Access to data members in virtual base classes is also slower than to those in non-virtual base classes. The details vary from compiler to compiler, but the basic thrust is clear: virtual inheritance costs.

It costs in other ways, too. The rules governing the initialization of vir- tual base classes are more complicated and less intuitive than are those for non-virtual bases. The responsibility for initializing a virtual base is borne by the *most derived class* in the hierarchy. Implications of this rule include (1) classes derived from virtual bases that require initialization must be aware of their virtual bases, no matter how far distant the bases are, and (2) when a new derived class is added to the hierarchy, it must assume initialization responsibilities for its virtual bases (both direct and indirect).

First, don’t use virtual bases unless you need to. By default, use non-virtual inheritance. Second, if you must use virtual base classes, try to avoid putting data in them. That way you won’t have to worry about oddities in the initialization (and, as it turns out, assignment) rules for such classes

This leads to one reasonable application of multiple inheritance: combine public inheritance of an interface with private inheritance of an implementation:

```cpp
class IPerson { 
public:
	virtual ~IPerson();
	virtual std::string name() const = 0;
	virtual std::string birthDate() const = 0; 
};

class DatabaseID { ... };

class PersonInfo { 
public:
	explicit PersonInfo(DatabaseID pid); 
  virtual ~PersonInfo();
	virtual const char * theName() const; 
  virtual const char * theBirthDate() const;
private:
	virtual const char * valueDelimOpen() const; 
  virtual const char * valueDelimClose() const;
};

class CPerson: public IPerson, private PersonInfo { 
public:
	explicit CPerson(DatabaseID pid): PersonInfo(pid) {}
	virtual std::string name() const{ 
    return PersonInfo::theName(); 
  }
	virtual std::string birthDate() const { 
  	return PersonInfo::theBirthDate(); 
	}
private:
	const char * valueDelimOpen() const { return ""; }
	const char * valueDelimClose() const { return ""; } 
};
```

Things to Remember

✦  Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance.

✦  Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It’s most practical when virtual base classes have no data.

✦  Multiple inheritance does have legitimate uses. One scenario in- volves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.

## 41.Understand implicit interfaces and compile-time polymorphism.

What’s important is that the set of expressions that must be valid in order for the template to compile is the *implicit interface* that T must support.

Because instantiating function templates with different template parameters leads to different functions being called, this is known as *compile-time polymorphism*.

Implicit interfaces are simply made up of a set of valid expressions. The expressions themselves may look complicated, but the constraints they impose are generally straightforward.

Things to Remember

✦  Both classes and templates support interfaces and polymorphism.

✦  For classes, interfaces are explicit and centered on function signatures. Polymorphism occurs at runtime through virtual functions.

✦  For template parameters, interfaces are implicit and based on valid expressions. Polymorphism occurs during compilation through tem- plate instantiation and function overloading resolution.

## 42.Understand the two meanings of typename.

1. When declaring a template type parameter, class and typename mean exactly the same thing

2. C++ doesn’t always view class and typename as equivalent, however. Sometimes you must use typename

   To understand when, we have to talk about two kinds of names you can refer to in a template.:

   ```cpp
   template<typename C>
   void print2nd(const C& container) {
   	if (container.size() >= 2) { 
       C::const_iteratoriter(container.begin()); //isn’t valid C++
       ++iter;
   		int value = *iter;
   		std::cout << value;
   	}
   }
   ```

   * Names in a template that are dependent on a template parameter are called *dependent names*. When a dependent name is nested inside a class, I call it a *nested dependent name*. C::const_iterator is a nested dependent name. In fact, it’s a *nested dependent type name*, i.e., a nested dependent name that refers to a type.
   * int is a name that does not depend on any template parameter. Such names are known as *non-dependent names*

   Nested dependent names can lead to parsing difficulties.

   ```cpp
   template<typename C>
   void print2nd(const C& container) {
   	C::const_iterator * x;
   }
   ```

   This looks like we’re declaring x as a local variable that’s a pointer to a C::const_iterator. But it looks that way only because we “know” that C::const_iterator is a type. But what if C::const_iterator weren’t a type? What if C had a static data member that happened to be named const_iterator, and what if x happened to be the name of a global variable? In that case, the code above wouldn’t declare a local variable, it would be a multiplication of C::const_iterator by x! Sure, that sounds crazy, but it’s *possible*, and people who write C++ parsers have to worry about all possible inputs, even the crazy ones.

   C++ has a rule to resolve this ambiguity: if the parser encounters a nested dependent name in a template, it assumes that the name is *not* a type unless you tell it otherwise. By default, nested dependent names are *not* types

   ```cpp
   template<typename C> // this is valid C++ 
   void print2nd(const C& container){
   	if (container.size() >= 2) {
   		typename C::const_iterator iter(container.begin());
   	}
   }
   ```

   The general rule is simple: anytime you refer to a nested dependent type name in a template, you must immediately precede it by the word typename.other names shouldn’t have it

   The exception to the “typename must precede nested dependent type names” rule is that typename must not precede nested dependent type names in a list of base classes or as a base class identifier in a mem- ber initialization list:

   ```cpp
   template<typename T> 
   class Derived : public Base<T>::Nested{ //base classlist: typename not allowed
   public:
   	explicit Derived(int x):Base<T>::Nested(x){//base class identifier in mem.init.list:typename not allowed  
   		typename Base<T>::Nested temp;// use of nested dependent type name not in a base class list or 
       ...													  // as a base class identifier in a mem. init. list: typename required 	
   	}
   };
   ```

   another typename example:

   ```cpp
   template<typename IterT> 
   void workWithIterator(IterT iter) {
   	typedef typename std::iterator_traits<IterT>::value_type value_type;
   	value_type temp(*iter);
   }
   ```

   Things to Remember

   ✦  When declaring template parameters, class and typename are interchangeable.

   ✦  Use typename to identify nested dependent type names, except in base class lists or as a base class identifier in a member initializa- tion list.

## 43.Know how to access names in templatized base classes.

```cpp
class CompanyA { 
public:
	void sendCleartext(const std::string& msg); 
  void sendEncrypted(const std::string& msg);
};

class CompanyB { 
public:
	void sendCleartext(const std::string& msg); 
  void sendEncrypted(const std::string& msg);
};

class MsgInfo { ... };

template<typename Company> 
class MsgSender {
public:
	void sendClear(const MsgInfo& info) {
		std::string msg; 
    //create msg from info;
		Company c;
		c.sendCleartext(msg); 
  }
	void sendSecret(const MsgInfo& info) { ... }
};

template<typename Company>
class LoggingMsgSender: public MsgSender<Company> { 
public:
  void sendClearMsg(const MsgInfo& info) {
		write "before sending" info to the log;
		sendClear(info);// call base class function; this code will not compile!
		write "after sending" info to the log; 
  }
};
```

The problem is that when compilers encounter the definition for the class template LoggingMsgSender, they don’t know what class it inherits from. Sure, it’s MsgSender\<Company\>, but Company is a template parameter, one that won’t be known until later (when LoggingMsgSender is instantiated). Without knowing what Company is, there’s no way to know what the class MsgSender\<Company\> looks like. In particular, there’s no way to know if it has a sendClear function.

we have to somehow disable C++’s “don’t look in templatized base classes” behavior. There are three ways to do this:

1. preface calls to base class functions with “this->”:

   ```cpp
   template<typename Company>
   class LoggingMsgSender: public MsgSender<Company> { 
   public:
   	void sendClearMsg(const MsgInfo& info) {
   		write "before sending" info to the log; 
     	this->sendClear(info);
   		write "after sending" info to the log; 
   	}
   };
   ```

2. employ a using declaration

   ```cpp
   template<typename Company>
   class LoggingMsgSender: public MsgSender<Company> { 
   public:
     //tell compilers to assume that sendClear is in the base class
     using MsgSender<Company>::sendClear;
   	void sendClearMsg(const MsgInfo& info) {
   		...
   		sendClear(info); 
       ...
   	}
   };
   ```

   (Although a using declaration will work both here and in Item 33, the problems being solved are different. Here, the situation isn’t that base class names are hidden by derived class names, it’s that compilers don’t search base class scopes unless we tell them to.)

3. explicitly specify that the function being called is in the base class:

   ```cpp
   template<typename Company>
   class LoggingMsgSender: public MsgSender<Company> { 
   public:
   	void sendClearMsg(const MsgInfo& info) {
   		MsgSender<Company>::sendClear(info);
   	}
   };
   ```

   通常，这是解决问题的最不理想的方式，因为如果所调用的函数是虚拟的，则显式限定会关闭虚拟绑定行为。

Fundamentally, the issue is whether compilers will diagnose invalid references to base class members sooner (when derived class template definitions are parsed) or later (when those templates are instantiated with specific template arguments). C++’s policy is to prefer early diagnoses, and that’s why it assumes it knows nothing about the contents of base classes when those classes are instantiated from templates.

## 44.Factor parameter-independent code out of templates.

template avoid code replication:

```cpp
template<typename T, std::size_t n>
class SquareMatrix { 
public:
	void invert();
};
```

```cpp
template<typename T> class SquareMatrixBase { 
protected:
	void invert(std::size_t matrixSize);
};

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> { 
private:
	using SquareMatrixBase<T>::invert;
public:
	void invert() { invert(n); }
};
```

The additional cost of calling it should be zero, because derived classes’ inverts call the base class version using inline functions. (The inline is implicit — see Item 30.)

base invert function need know data memory:

```cpp
template<typename T> 
class SquareMatrixBase { 
protected:
	SquareMatrixBase(std::size_t n, T *pMem) : size(n), pData(pMem) {}
	void setDataPtr(T *ptr) { pData = ptr; }
private: 
  std::size_t size;
	T *pData;
};

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> { 
public:
	SquareMatrix() : SquareMatrixBase<T>(n, data) {}
private:
	T data[n*n];
};

// data may be too large, so allocate on memory 
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> { 
public:
	SquareMatrix( ) : SquareMatrixBase<T>(n,0), pData(new T[n*n]){ 
    this->setDataPtr(pData.get()); 
  }
private:
	boost::scoped_array<T> pData;
};
```

 now many --maybe-- all  of SquareMatrix’s member functions can be simple inline calls to (non-inline) base class versions that are shared with all other matrices holding the same type of data, regardless of their size. At the same time, SquareMatrix objects of different sizes are distinct types.

Think adout it :

1. The versions of invert with the matrix sizes hardwired(硬编码) into them are likely to generate better code than the shared version where the size is passed as a function parameter or is stored in the object.	
2. On the other hand, having only one version of invert for multiple matrix sizes decreases the size of the executable, and that could reduce the program’s working set size and improve locality of refer- ence in the instruction cache. Those things could make the program run faster, more than compensating for any lost optimizations in size- specific versions of invert
3. Another efficiency consideration concerns the sizes of objects.moving size-independent versions of functions up into a base class can increase the overall size of each object.It can also lead to resource management complications.
4. This Item has discussed only bloat due to non-type template parame- ters, but type parameters can lead to bloat,
   * int and long have the same binary representation, so the member functions for, say, vector<int> and vector<long> would likely be identical — the very definition of bloat
   * Similarly, on most platforms, all pointer types have the same binary representation, so templates holding pointer types (e.g., list<int*>, list<const int*>, list<SquareMatrix<long, 3>*>, etc.) should often be able to use a single underlying implementation for each member function

Things to Remember

- ✦  Templates generate multiple classes and multiple functions, so any template code not dependent on a template parameter causes bloat(膨胀).
- ✦  Bloat due to non-type template parameters can often be eliminated by replacing template parameters with function parameters or class data members.
- ✦  Bloat due to type parameters can be reduced by sharing implemen- tations for instantiation types with identical binary representations.

## 45.Use member function templates to accept “all compatible types.”

One of the things that real pointers do well is support implicit conver- sions. Derived class pointers implicitly convert into base class point- ers, pointers to non-const objects convert into pointers to const objects.

```cpp
class Top { ... };
class Middle: public Top { ... }; 
class Bottom: public Middle { ... }; 
Top *pt1 = new Middle;
Top *pt2 = new Bottom;
const Top *pct2 = pt1;

//To get the conversions among SmartPtr classes that we want, we have to program them explicitly.
template<typename T> 
class SmartPtr {
public:
	explicit SmartPtr(T *realPtr);
};
SmartPtr<Top> pt1 = SmartPtr<Middle>(new Middle);// convert SmartPtr<Middle> ⇒ // SmartPtr<Top>
SmartPtr<Top> pt2 = SmartPtr<Bottom>(new Bottom);// convert SmartPtr<Bottom> ⇒ // SmartPtr<Top>
SmartPtr<const Top> pct2 = pt1;// convert SmartPtr<Top> ⇒ // SmartPtr<const Top>
```

*member function templates* templates  -- that generate member functions of a class:

```cpp
template<typename T> 
class SmartPtr {
public:
	template<typename U> 
  SmartPtr(const SmartPtr<U>& other);
};
```

The generalized copy constructor above is not declared explicit. That’s deliberate.

to restrict the conversions:

```cpp
template<typename T> 
class SmartPtr {
public:
	template<typename U> 
  SmartPtr(const SmartPtr<U>& other) : heldPtr(other.get()) { ... }
	T* get() const { 
    return heldPtr; 
  }
private:
	T *heldPtr;
};
```

We use the member initialization list to initialize SmartPtr\<T\>’s data member of type T* with the pointer of type U* held by the SmartPtr\<U\>. This will compile only if there is an implicit conversion from a U* pointer to a T* pointer, and that’s precisely what we want.

The utility of member function templates isn’t limited to constructors. Another common role for them is in support for assignment.

```cpp
template<class T> 
class shared_ptr { 
public:
	template<class Y>
	explicit shared_ptr(Y * p);
  
	template<class Y> 
  shared_ptr(shared_ptr<Y> const& r);
  
	template<class Y>
	explicit shared_ptr(weak_ptr<Y> const& r);
  
	template<class Y>
	explicit shared_ptr(auto_ptr<Y>& r);
  
	template<class Y>
	shared_ptr& operator=(shared_ptr<Y> const& r);
  
	template<class Y>
	shared_ptr& operator=(auto_ptr<Y>& r);
};
```

how the auto_ptrs passed to tr1::shared_ptr constructors and assignment operators aren’t declared const, in contrast to how the tr1::shared_ptrs and tr1::weak_ptrs are passed. That’s a consequence of the fact that auto_ptrs stand alone in being modified when they’re copied

Member function templates are wonderful things, but they don’t alter the basic rules of the language. Item 5 explains that two of the four member functions that compilers may generate are the copy construc- tor and the copy assignment operator. tr1::shared_ptr declares a generalized copy constructor, and it’s clear that when the types T and Y are the same, the generalized copy constructor could be instantiated to create the “normal” copy constructor. So will compilers generate a copy constructor for tr1::shared_ptr, or will they instantiate the genera ized copy constructor template when one tr1::shared_ptr object is constructed from another tr1::shared_ptr object of the same type?

As I said, member templates don’t change the rules of the language, and the rules state that if a copy constructor is needed and you don’t declare one, one will be generated for you automatically. Declaring a generalized copy constructor (a member template) in a class doesn’t keep compilers from generating their own copy constructor (a non- template), so if you want to control all aspects of copy construction, you must declare both a generalized copy constructor as well as the “normal” copy constructor. The same applies to assignment. Here’s an excerpt from tr1::shared_ptr’s definition that exemplifies this:

```cpp
template<class T> 
class shared_ptr { 
public:
	shared_ptr(shared_ptr const& r);
	template<class Y> 
  shared_ptr(shared_ptr<Y> const& r);
  
	shared_ptr& operator=(shared_ptr const& r);  
	template<class Y>
	shared_ptr& operator=(shared_ptr<Y> const& r);
};
```

Things to Remember

✦  Use member function templates to generate functions that accept all compatible types.

✦  If you declare member templates for generalized copy construction or generalized assignment, you’ll still need to declare the normal copy constructor and copy assignment operator, too.

## 46.Define non-member functions inside templates when type conversions are desired.

```cpp
template<typename T>
class Rational { 
public:
	Rational(const T& numerator = 0, const T& denominator = 1);
	const T numerator() const; 
  const T denominator() const;
};

template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs){...}

//we want to support mixed-mode arithmetic
Rational<int> oneHalf(1, 2); 				// this example is from Item 24, except Rational is now a template
Rational<int> result = oneHalf * 2; // error! won’t compile
```

In Item 24, compilers know what function we’re trying to call (operator* taking two Rationals), but here, compilers do *not* know which function we want to call. Instead, they’retrying to *figure out* what function to instantiate from the template named operator\*. They know that they’re supposed to instantiate some function named operator\* taking two parameters of type Rational\<T\>, but in order to do the instantiation, they have to figure out what T is. The problem is, they can’t.

In attempting to deduce T, they look at the types of the arguments being passed in the call to operator\*. In this case, those types are Rational\<int\> (the type of oneHalf) and int (the type of 2). **Each parameter is considered separately.**

The deduction using oneHalf is easy. operator\*’s first parameter is declared to be of type Rational\<T\>, and the first argument passed to operator\* (oneHalf) is of type Rational\<int\>, so T must be int. Unfortunately, the deduction for the other parameter is not so simple. operator\*’s second parameter is declared to be of type Rational\<T\>, but the second argument passed to operator\* (2) is of type int. How are compilers to figure out what T is in this case? You might expect them to use Rational\<int\>’s nonexplicit constructor to convert 2 into a Rational\<int\>, thus allowing them to deduce that T is int, but they don’t do that. They don’t, **because implicit type conversion functions are *never* considered during template argument deduction. Never. Such conversions are used during function calls, yes, but before you can call a function, you have to know which functions exist. In order to know that, you have to deduce parameter types for the relevant function templates** (so that you can instantiate the appropriate functions). But implicit type conversion via constructor calls is not considered during template argument deduction. Item 24 involves no templates, so template argument deduction is not an issue. Now that we’re in the template part of C++ (see Item 1), it’s the primary issue.

We can relieve(缓解) compilers of the challenge of template argument deduction by **taking advantage of the fact that a friend declaration in a template class can refer to a specific function.** That means the class Rational\<T\> can declare operator\* for Rational\<T\> as a friend function. **Class templates don’t depend on template argument deduction (that process applies only to function templates), so T is always known at the time the class Rational\<T\> is instantiated.** That makes it easy for the Rational\<T\> class to declare the appropriate operator\* function as a friend:

```cpp
template<typename T> class Rational {
public:
	friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};

template<typename T> 
constRational<T> operator*(constRational<T>&lhs, const Rational<T>& rhs){...}
```

Now our mixed-mode calls to operator\* will compile, because when the object oneHalf is declared to be of type Rational\<int\>, the class Rational\<int\> is instantiated, and as part of that process, the friend function operator\* that takes Rational\<int\> parameters is automatically declared. As a declared *function* (not a function *template*), compilers can use implicit conversion functions (such as Rational’s non-explicit constructor) when calling it, and that’s how they make the mixed- mode call succeed

Inside a class template, the name of the template can be used as shorthand for the template and its parameters, so inside Rational<T>, we can just write Rational instead of Rational<T>

above code has link problem, because Our intent is to have the operator* template outside the class provide that definition, but things don’t work that way. If we declare a function ourselves (which is what we’re doing inside the Rational template), we’re also responsible for defining that function. In this case, we never provide a definition, and that’s why linkers can’t find one.

```cpp
template<typename T> 
class Rational {
public:
	friend const Rational operator*(const Rational& lhs, 
                                  const Rational& rhs){
		return Rational(lhs.numerator() * rhs.numerator(), 
                    lhs.denominator() * rhs.denominator());
  }
};
```

In order to make type conversions possible on all arguments, we need a non-member function (Item 24 still applies); and in order to have the proper function automatically instantiated, we need to declare the function inside the class. The only way to declare a non- member function inside a class is to make it a friend.

As Item 30 explains, functions defined inside a class are implicitly declared inline, and that includes friend functions like operator\*. You can minimize the impact of such inline declarations by having operator\* do nothing but call a helper function defined outside of the class

```cpp
template<typename T> class Rational;

template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, 
                             const Rational<T>& rhs) {
	return Rational<T>(lhs.numerator() * rhs.numerator(),
                     lhs.denominator() * rhs.denominator());
}

template<typename T> 
class Rational {
public:
friend
	const Rational<T> operator*(const Rational<T>& lhs,
                              const Rational<T>& rhs) { 
  	return doMultiply(lhs, rhs); 
	}
};
```

Things to Remember

✦ When writing a class template that offers functions related to the template that support implicit type conversions on all parameters, define those functions as friends inside the class template.

## 47.Use traits classes for information about types.

```cpp
template<typename IterT, typename DistT> // move iter d units forward; if d < 0, move iter backward
void advance(IterT& iter, DistT d);

struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag: public input_iterator_tag {};
struct bidirectional_iterator_tag: public forward_iterator_tag {};
struct random_access_iterator_tag: public bidirectional_iterator_tag {};

template<typename IterT, typename DistT> 
void advance(IterT& iter, DistT d) {
	if (iter is a random access iterator) { 
    iter += d;
	} else {
		if (d >= 0) { 
      while (d--)  ++iter; 
    } else { 
      while (d++) --iter; 
    }
	}
}
```

In other words, we need to get some information about a type. That’s what *traits* let you do: they allow you to get information about a type during compilation.

Traits are a technique and a convention followed by C++ programmers. One of the demands made on the technique is that it has to work as well for built-in types as it does for user-defined types, So the traits information for a type must be external to the type.The standard technique is to put it into a template and one or more specializations of that template.For iterators, the template in the standard library is named iterator_traits:

```cpp
template<typename IterT> // template for information about 
struct iterator_traits; // iterator types
```

The way iterator_traits works is that for each type IterT, a typedef named iterator_category is declared in the struct iterator_traits\<IterT\>. This typedef identifies the iterator category of IterT.

iterator_traits implements this in two parts. First, it imposes the requirement that any user-defined iterator type must contain a nested typedef named iterator_category that identifies the appropriate tag struct.

```cpp
template < ... > // template params elided 
class deque {
public:
	class iterator { 
  public:
		typedef random_access_iterator_tag iterator_category;
	};
};
```

```cpp
template < ... > 
class list { 
public:
	class iterator { 
  public:
		typedef bidirectional_iterator_tag iterator_category;
	};
};
```

iterator_traits just parrots back the iterator class’s nested typedef:

```cpp
// the iterator_category for type IterT is whatever IterT says it is; 
// see Item 42 for info on the use of “typedef typename” 
template<typename IterT>
struct iterator_traits {
	typedef typename IterT::iterator_category iterator_category;
}	
```

 it doesn’t work at all for iterators that are pointers, because there’s no such thing as a pointer with a nested typedef. ,so iterator_traits offers a *partial template specialization* for pointer types. Pointers act as random access iterators, so that’s the category iterator_traits specifies for them:

```cpp
template<typename T> // partial template specialization 
struct iterator_traits<T*> {// for built-in pointer types
	typedef random_access_iterator_tag iterator_category;
};
```

At this point, you know how to design and implement a traits class:

- Identify some information about types you’d like to make available (e.g., for iterators, their iterator category).

- Choose a name to identify that information (e.g., iterator_category).

- Provide a template and set of specializations (e.g., iterator_traits)

  that contain the information for the types you want to support.

we can refine our pseudocode for advance:

```cpp
template<typename IterT, typename DistT> 
void advance(IterT& iter, DistT d){
	if (typeid(typename std::iterator_traits<IterT>::iterator_category) == 				
      typeid(std::random_access_iterator_tag)){
			...    
  }
}
```

IterT’s type is known during compilation, so iterator_traits\<IterT\>::iterator_category can also be determined during compilation. Yet the if statement is evaluated at runtime. Why do something at runtime that we can do during compilation? It wastes time (literally), and it bloats our executable.

What we really want is a conditional construct (i.e., an if...else statement) for types that is evaluated during compilation. As it happens, C++ already has a way to get that behavior. It’s called overloading.

```cpp
template<typename IterT, typename DistT> 
void doAdvance(IterT& iter, DistT d,std::random_access_iterator_tag) {
	iter += d;
}

template<typename IterT, typename DistT> 
void doAdvance(IterT& iter, DistT d, std::bidirectional_iterator_tag) {
	if (d >= 0) { 
    while (d--) ++iter; 
  } else { 
    while (d++) --iter; 
  }
}

template<typename IterT, typename DistT> 
void doAdvance(IterT& iter, DistT d, std::input_iterator_tag) {
	if(d<0){
    throw std::out_of_range("Negative distance");
  }
  while (d--) ++iter;
}

template<typename IterT, typename DistT> 
void advance(IterT& iter, DistT d) {
	doAdvance( iter, d, typename std::iterator_traits<IterT>::iterator_category() ); 
}
```

We can now summarize how to use a traits class:

- Create a set of overloaded “worker” functions or function templates (e.g., doAdvance) that differ in a traits parameter. Implement each function in accord with the traits information passed.
- Create a “master” function or function template (e.g., advance) that calls the workers, passing information provided by a traits class.

Things to Remember

1. ✦  Traits classes make information about types available during compilation. They’re implemented using templates and template specializations.

2. ✦  In conjunction with overloading, traits classes make it possible to perform compile-time if...else tests on types.

## 48.Be aware of template metaprogramming.

TMP(template metaprogramming) has two great strengths.

 First, it makes some things easy that would otherwise be hard or impossible. 

Second, because template metaprograms execute during C++ compilation, they can shift work from runtime to compile-time. 

```cpp
template<typename IterT, typename DistT> 
void advance(IterT& iter, DistT d){
	if (typeid(typename std::iterator_traits<IterT>::iterator_category) == 				
      typeid(std::random_access_iterator_tag)){
			iter += d;
  }else {
    if (d >= 0) { 
      while (d--) ++iter; 
    } else { 
      while (d++) --iter; 
    }
  }
}

std::list<int>::iterator iter;
advance(iter, 10);
```

Consider the version of advance that will be generated for the above call. After substituting iter’s and 10’s types for the template parameters IterT and DistT, we get this:

```cpp
void advance(std::list<int>::iterator& iter, int d) {
  if (typeid(std::iterator_traits<std::list<int>::iterator>::iterator_category) == 				
      typeid(std::random_access_iterator_tag)) {
      iter += d;
  } else {
    if (d >= 0) { 
      while (d--) ++iter; 
    } else { 
      while (d++) --iter; 
    } 
  }
}
```

The problem is the highlighted line, the one using +=. In this case, we’re trying to use += on a list<int>::iterator, but list<int>::iterator is a bidirectional iterator (see Item 47), so it doesn’t support +=.  we know we’ll never try to execute the += line, because the typeid test will always fail for list<int>::iterators, but compilers are obliged to make sure that all source code is valid, even if it’s not executed, and “iter += d” isn’t valid when iter isn’t a random access iterator.

TMP has been shown to be Turing-complete, which means that it is powerful enough to compute anything. Using TMP, you can declare variables, perform loops, write and call functions, etc. But such constructs look very different from their “normal” C++ counterparts. For example if...else conditionals in TMP are expressed via templates and template specializations. TMP loops don’t involve recursive function calls, they involve recursive *template instantiations*.

```cpp
template<unsigned n> 
struct Factorial {
  enum { value = n * Factorial<n-1>::value };
};
template<> 
struct Factorial<0> {
	enum{value=1}; 
};

int main(){
	std::cout << Factorial<5>::value; 
  std::cout << Factorial<10>::value;  
}
```

uses the enum hack (see Item 2) to declare a TMP variable named value

Things to Remember

✦  Template metaprogramming can shift work from runtime to compile-time, thus enabling earlier error detection and higher runtime performance.

✦  TMP can be used to generate custom code based on combinations of policy choices, and it can also be used to avoid generating code inappropriate for particular types.

## 49.Understand the behavior of the new-handler.

## 50.Understand when it makes sense to replace new and delete.

## 51.Adhere to convention when writing new and delete.

## 52.Write placement delete if you write placement new.

## 53. Pay attention to compiler warnings.

Things to Remember

- ✦  Take compiler warnings seriously, and strive to compile warning- free at the maximum warning level supported by your compilers.
- ✦  Don’t become dependent on compiler warnings, because different compilers warn about different things. Porting to a new compiler may eliminate warning messages you’ve come to rely on.

## 54. Familiarize yourself with the standard library, including TR1.

Before surveying what’s in TR1, it’s worth reviewing the major parts of the standard C++ library specified by C++98:

1. The Standard Template Library (STL)
2. Iostreams
3. Support for internationalization
4. Support for numeric processing
5. An exception hierarchy
6. C89’s standard library

This book shows examples of the following TR1 components:

1. The smart pointers tr1::shared_ptr and tr1::weak_ptr.

2. **tr1::function**

   tr1::function makes it possible to make registerCallback much more flexible, accepting as its argument any callable entity that takes an int or *anything an* int *can be converted into* and that returns a string or *anything convertible to a* string. tr1::function takes as a template parameter its target function signature:

   ```cpp
   void registerCallback(std::tr1::function<std::string (int)> func);
   ```

3. **tr1::bind**

I divide the remaining TR1 components into two sets. The first group offers fairly discrete standalone functionality:

1. Hash tables 

2. Regular expressions,

3. Tuples

4. **tr1::array**

5. **tr1::mem_fn**

   在语法上统一成员函数指针的方式。 正如tr1 :: bind包含并扩展C ++ 98的bind1st和bind2nd的功能一样，tr1 :: mem_fn也包含并扩展C ++ 98的mem_fun和mem_fun_ref的功能。

6. **tr1::reference_wrapper**

7. Random number generation

8. Mathematical special functions

9. C99 compatibility extensions

第二组TR1组件包含用于更复杂的模板编程技术的支持技术，包括模板元编程

1. Type traits
2. **tr1::result_of**

## 55. Familiarize yourself with Boost.

Things to Remember

1. ✦  Boost is a community and web site for the development of free, open source, peer-reviewed C++ libraries. Boost plays an influential role in C++ standardization.

2. ✦  Boost offers implementations of many TR1 components, but it also offers many other libraries, too.