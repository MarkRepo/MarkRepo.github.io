---
title: 23种设计模式总结
description:
categories: C++
tags:
  - C++
  - 设计模式
---

## SingleTon(单例模式)

```cpp
//singleton.h
#ifndef _SINGLETON_H_
#define _SINGLETON_H_
#include <iostream>
using namespace std;

class Singleton{
public:
	static Singleton* Instance();
protected:
	Singleton();
private:
	static Singleton* _instance;
}
#endif
```
```cpp
//singleton.cpp
#include "Singleton.h"
Singleton* singleton::_instance = 0;
Singleton::Singleton(){
	cout<<"Singleton...."<<endl;
}
Singleton* single::Instance(){
	if(_instance == 0)
		_instance = new Singleton();
	return _instance;
}
```
>SingleTon不可以被实例化，因此将其构函数声明为protected或者private。另外需要考虑多线程环境下避免构造了多个实例。

## Prototype(原型模式)

模式结构图:
![Prototype](/assets/images/patterns/prototype.png)

```cpp
//prototype.h
#ifndef _PROTOTYPE_H
#define _PROTOTYPE_H

class Prototype{
public:
	virtual ~Prototype();
	virtual Prototype* Clone() const = 0;
protected:
	Prototype();
};

class ConcretePrototype : public Prototype{
public:
	ConcretePrototype();
	ConcretePrototype(const ConcretePrototype &cp);
	~ConcretePrototype();
	Prototype* Clone() const;
};
#endif
```
```cpp
//prototype.cpp
#include "prototype.h"
#include <iostream>
using namespace std;

Prototype::Prototype(){}
Prototype::~Prototype(){}
Prototype* Prototype::Clone() const{ return 0; }
ConcretePrototype::ConcretePrototype(){}
ConcretePrototype::~ConcretePrototype(){}
ConcretePrototype::ConcretePrototype(const ConcretePrototype& cp){
	cout<<"ConcretePrototype copy..."<<endl;
}
Prototype* ConcretePrototype::Clone()const{
	return new ConcretePrototype(*this);
}
```

## Factory(工厂模式)

Factory模式两个最重要的功能:

+ 定义创建对象的接口，封装了对象的创建。
+ 使得具体化类的工作延迟到了子类中。

Factory模式结构示意图:
![Prototype](/assets/images/patterns/factory.png)

```cpp
//Product.h
#ifndef _PRODUCT_H_
#define _PRODUCT_H_

class Product{
public:
    virtual ~Product() = 0;
protected:
    Product();
};

class ConcreteProduct:public Product{
public:
	~ConcreteProduct();
	ConcreteProduct();
};
#endif
```
```cpp
//Product.cpp
#include "Product.h"
#include <iostream>
using namespace std;

Product::Product(){}
Product::~Product(){}
ConcreteProduct::ConcreteProduct(){
    cout<<"ConcreteProduct..."<<endl;
}
ConcreteProduct::~ConcreteProduct(){}
```
```cpp
//Factory.h
#ifndef _FACTORY_H_
#define _FACTORY_H_

class Product;
class Factory{
public:
    virtual ~Factory() = 0;
    virtual Product* CreateProduct() = 0;
protected:
    Factory();
};

class ConcreteFactory:public Factory{
public:
    ~ConcreteFactory();
    ConcreteFactory();
    Product* CreateProduct();
};
#endif
```
```cpp
//Factory.cpp
#include "Factory.h"
#include "Product.h"
#include <iostream>
using namespace std;

Factory::Factory(){}
Factory::~Factory(){}
ConcreteFactory::ConcreteFactory(){
    cout<<"ConcreteFactory....."<<endl;
}
ConcreteFactory::~ConcreteFactory(){}
Product* ConcreteFactory::CreateProduct(){
    return new ConcreteProduct();
}
```

## AbstractFactroy(抽象工厂模式)

AbstractFactory模式用来解决这类问题：要创建一组相关或者相互依赖的对象。  
结构图：
![abstractFactory](/assets/images/patterns/abstract_factory.jpg)

实现：

```cpp
//Product.h
#ifndef _PRODUCT_H_ 
#define _PRODUCT_H_

class AbstractProductA { 
public: 
    virtual ~AbstractProductA();
protected: 
    AbstractProductA();
};
class AbstractProductB{ 
public: 
    virtual ~AbstractProductB();
protected: 
    AbstractProductB();
};
class ProductA1:public AbstractProductA{ 
public: 
    ProductA1();
    ~ProductA1();
}
class ProductA2:public AbstractProductA {
public: 
    ProductA2();
    ~ProductA2();
};
class ProductB1:public AbstractProductB 
{ 
public: 
    ProductB1();
    ~ProductB1();
};
class ProductB2:public AbstractProductB 
{ 
public: 
    ProductB2();
    ~ProductB2();
};
#endif
```
```cpp
//Product.cpp
#include "Product.h"
#include <iostream> 
using namespace std;
AbstractProductA::AbstractProductA(){}
AbstractProductA::~AbstractProductA(){}
AbstractProductB::AbstractProductB(){}
AbstractProductB::~AbstractProductB() {}
ProductA1::ProductA1(){cout<<"ProductA1..."<<endl;}
ProductA1::~ProductA1(){}
ProductA2::ProductA2(){cout<<"ProductA2..."<<endl;}
ProductA2::~ProductA2(){}
ProductB1::ProductB1(){cout<<"ProductB1..."<<endl;}
ProductB1::~ProductB1(){}
ProductB2::ProductB2(){cout<<"ProductB2..."<<endl;}
ProductB2::~ProductB2(){}
```
```cpp
//AbstractFactory.h
#ifndef _ABSTRACTFACTORY_H_ 
#define _ABSTRACTFACTORY_H_

class AbstractProductA; 
class AbstractProductB;
class AbstractFactory{ 
public: 
    virtual ~AbstractFactory();
    virtual AbstractProductA* CreateProductA() = 0;
    virtual AbstractProductB* CreateProductB() = 0;
protected: 
    AbstractFactory();
};
class ConcreteFactory1:public AbstractFactory{ 
public:
    ConcreteFactory1();
    ~ConcreteFactory1();
    AbstractProductA* CreateProductA();
    AbstractProductB* CreateProductB();
};
class ConcreteFactory2:public AbstractFactory { 
public: 
    ConcreteFactory2();
    ~ConcreteFactory2();
    AbstractProductA* CreateProductA();
    AbstractProductB* CreateProductB();
};
#endif
```
```cpp
//AbstractFactory.cpp
#include "AbstractFactory.h" 
#include "Product.h"
#include <iostream> 
using namespace std;

AbstractFactory::AbstractFactory(){}
AbstractFactory::~AbstractFactory(){}
ConcreteFactory1::ConcreteFactory1(){}
ConcreteFactory1::~ConcreteFactory1() {}
AbstractProductA* ConcreteFactory1::CreateProductA(){return new ProductA1(); }
AbstractProductB* ConcreteFactory1::CreateProductB(){return new ProductB1(); }
ConcreteFactory2::ConcreteFactory2(){}
ConcreteFactory2::~ConcreteFactory2(){}
AbstractProductA* ConcreteFactory2::CreateProductA(){return new ProductA2();}
AbstractProductB* ConcreteFactory2::CreateProductB(){return new ProductB2();}
```

## Builder(构建模式)

Builder模式解决这样的问题：当要创建的对象很复杂的时候（通常是由很多其他的对象组合而成），把复杂对象的创建过程和这个对象的表示（展示）分离开来，这样做的好处就是通过一步步的进行复杂对象的构建，由于在每一步的构造过程中可以引入参数，使得经过相同的步骤创建最后得到的对象的展示不一样。  
模式结构图：
![Builder](/assets/images/Patterns/Builder.png)

```cpp
//main.cpp
#include "Builder.h"
#include "Product.h"
#include "Director.h"
#include <iostream>
using namespace std;

int main(int argc,char* argv[]){
    Director* d = new Director(new ConcreteBuilder());
    d->Construct();
    return 0;  
}
```
```cpp
//Product.h
#ifndef _PRODUCT_H_ 
#define _PRODUCT_H_
class Product { 
public: 
    Product();
    ~Product();
    void ProducePart();
};
class ProductPart {
public: 
    ProductPart();
    ~ProductPart();
    ProductPart* BuildPart();
};
#endif
```
```cpp
//Product.cpp
#include "Product.h" 
#include <iostream> 
using namespace std;
Product::Product() { 
    ProducePart();
    cout<<"return a product"<<endl; 
}
Product::~Product(){}
void Product::ProducePart(){
    cout<<"build part of product.."<<endl;
}
ProductPart::ProductPart(){ 
    //cout<<"build productpart.."<<endl; 
}
ProductPart::~ProductPart(){}
ProductPart* ProductPart::BuildPart(){
    return new ProductPart; 
}
```
```cpp
//Builder.h
#ifndef _BUILDER_H_ 
#define _BUILDER_H_
#include <string> 
using namespace std;
class Product;
class Builder { 
public: 
    virtual ~Builder();
    virtual void BuildPartA(const string& buildPara) = 0;
    virtual void BuildPartB(const string& buildPara) = 0;
    virtual void BuildPartC(const string& buildPara) = 0;
    virtual Product* GetProduct() = 0;
protected: 
    Builder();
};
class ConcreteBuilder:public Builder { 
public: 
    ConcreteBuilder();
    ~ConcreteBuilder();
    void BuildPartA(const string& buildPara);
    void BuildPartB(const string& buildPara);
    void BuildPartC(const string& buildPara);
    Product* GetProduct();
};
#endif
```
```cpp
//Builder.cpp
#include "Builder.h" 
#include "Product.h"
#include <iostream> 
using namespace std;
Builder::Builder(){}
Builder::~Builder(){}
ConcreteBuilder::ConcreteBuilder(){}
ConcreteBuilder::~ConcreteBuilder(){}
void ConcreteBuilder::BuildPartA(const string& buildPara) { 
    cout<<"Step1:Build PartA..."<<buildPara<<endl; 
}
void ConcreteBuilder::BuildPartB(const string& buildPara) {
    cout<<"Step1:Build PartB..."<<buildPara<<endl;
}
void ConcreteBuilder::BuildPartC(const string& buildPara) {
    cout<<"Step1:Build PartC..."<<buildPara<<endl; 
}
Product* ConcreteBuilder::GetProduct() {
    BuildPartA("pre-defined");
    BuildPartB("pre-defined");
    BuildPartC("pre-defined");
    return new Product(); 
}
```
```cpp
//Director.h
#ifndef _DIRECTOR_H_ 
#define _DIRECTOR_H_
class Builder;
class Director{ 
public:
    Director(Builder* bld);
    ~Director();
    void Construct();
private: 
    Builder* _bld;
};
#endif
```
```cpp
//Director.cpp
#include "director.h" 
#include "Builder.h"
Director::Director(Builder* bld){_bld = bld;}
Director::~Director(){}
void Director::Construct(){ 
    _bld->BuildPartA("user-defined"); 
    _bld->BuildPartB("user-defined"); 
    _bld->BuildPartC("user-defined"); 
}
```

## Bridge(桥接模式)

使用组合的方式将“抽象”和“实现”彻底的解耦。这里的实现不是指继承基类，实现基类接口，而是指通过对象的组合实现用户的需求。
面向对象分析和设计中的原则: `Favor Compsition Over Inheritance`.  
模式结构图:
![Bridge](/assets/images/patterns/Bridge.png)

```cpp
//main.cpp
#include "Abstraction.h" 
#include "AbstractionImp.h"
#include <iostream> 
using namespace std;

int main(int argc,char* argv[]) {
    AbstractionImp* imp = new ConcreteAbstractionImpA();
    Abstraction* abs = new RefinedAbstraction(imp);
    abs->Operation();
    return 0;
}
```
```cpp
//Abstraction.h
#ifndef _ABSTRACTION_H_ 
#define _ABSTRACTION_H_
class AbstractionImp;
class Abstraction {
public: 
    virtual ~Abstraction();
    virtual void Operation() = 0;
protected: 
    Abstraction();
};
class RefinedAbstraction:public Abstraction {
public: 
    RefinedAbstraction(AbstractionImp* imp);
    ~RefinedAbstraction();
    void Operation();
private: 
    AbstractionImp* _imp;
};
#endif
```
```cpp
//Abstraction.cpp
#include "Abstraction.h" 
#include "AbstractionImp.h"
#include <iostream> 
using namespace std;
Abstraction::Abstraction(){}
Abstraction::~Abstraction(){}
RefinedAbstraction::RefinedAbstraction(AbstractionImp* imp){_imp = imp;}
RefinedAbstraction::~RefinedAbstraction(){}
void RefinedAbstraction::Operation(){_imp->Operation();}
```
```cpp
//AbstractionImp.h
#ifndef _ABSTRACTIONIMP_H_ 
#define _ABSTRACTIONIMP_H_
class AbstractionImp{
public: 
    virtual ~AbstractionImp();
    virtual void Operation() = 0;
protected: 
    AbstractionImp();
};
class ConcreteAbstractionImpA:public AbstractionImp {
public: 
    ConcreteAbstractionImpA();
    ~ConcreteAbstractionImpA();
    virtual void Operation();
};
class ConcreteAbstractionImpB:public AbstractionImp {
public:
    ConcreteAbstractionImpB();
    ~ConcreteAbstractionImpB();
    virtual void Operation();
};
#endif
```
```cpp
//AbstractionImp.cpp
#include "AbstractionImp.h"
#include <iostream> 
using namespace std;

AbstractionImp::AbstractionImp(){}
AbstractionImp::~AbstractionImp() {}
void AbstractionImp::Operation() {
    cout<<"AbstractionImp....imp..."<<endl;
}
ConcreteAbstractionImpA::ConcreteAbstractionImpA(){}
ConcreteAbstractionImpA::~ConcreteAbstractionImpA(){}
void ConcreteAbstractionImpA::Operation(){
    cout<<"ConcreteAbstractionImpA...."<<endl;
}
ConcreteAbstractionImpB::ConcreteAbstractionImpB(){}
ConcreteAbstractionImpB::~ConcreteAbstractionImpB(){}
void ConcreteAbstractionImpB::Operation() {
    cout<<"ConcreteAbstractionImpB...."<<endl;
}
```

## Adapter(适配器模式)

Adapter模式用来实现将一个类（第三方库）的接口转换为客户（购买使用者）希望的接口。
Adapter模式有两种类别：类模式、对象模式。类模式采用继承的方式复用Adaptee的接口，对象模式通过组合的方式实现Adaptee的复用。  
类模式结构图：  
![Adapter_class](/assets/images/patterns/Adapter_class.png)
对象模式结构图：
![Adapter_class](/assets/images/patterns/Adapter_object.png)

类模式实现：

```cpp
//Adapter.h
#ifndef _ADAPTER_H_ 
#define _ADAPTER_H_
class Target {
public: 
    Target();
    virtual ~Target();
    virtual void Request();
};
class Adaptee {
public: 
    Adaptee();
    ~Adaptee();
    void SpecificRequest();
};
class Adapter:public Target,private Adaptee {
public:
    Adapter();
    ~Adapter();
    void Request();
};
#endif
```
```cpp
//Adapter.cpp
#include "Adapter.h"
#include <iostream>
Target::Target(){}
Target::~Target(){}
void Target::Request() {
    std::cout<<"Target::Request"<<std::endl;
}
Adaptee::Adaptee(){}
Adaptee::~Adaptee(){}
void Adaptee::SpecificRequest(){
    std::cout<<"Adaptee::SpecificRequest" <<std::endl; 
}
Adapter::Adapter(){}
Adapter::~Adapter() {}
void Adapter::Request() {
    this->SpecificRequest();
}
```
```cpp
//main.cpp
#include "Adapter.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) { 
	//Adapter* adt = new Adapter();
    Target* adt = new Adapter();
    adt->Request();
    return 0; 
}
```
对象模式实现：

```cpp
//Adapter.h
#ifndef _ADAPTER_H_ 
#define _ADAPTER_H_
class Target {
public: 
    Target();
    virtual ~Target();
    virtual void Request();
};
class Adaptee {
public: 
    Adaptee();
    ~Adaptee();
    void SpecificRequest();
};
class Adapter:public Target {
public:
    Adapter(Adaptee* ade);
    ~Adapter();
    void Request();
private:
    Adaptee* _ade;
};
#endif
```
```cpp
//main.cpp
#include "Adapter.h"
#include <iostream>
Target::Target(){}
Target::~Target(){}
void Target::Request(){ 
    std::cout<<"Target::Request"<<std::endl;
}
Adaptee::Adaptee(){}
Adaptee::~Adaptee(){}
void Adaptee::SpecificRequest(){
    std::cout<<"Adaptee::SpecificRequest" <<std::endl; 
}
Adapter::Adapter(Adaptee* ade){
    this->_ade = ade;
}
Adapter::~Adapter(){}
void Adapter::Request() {
    _ade->SpecificRequest();
}
```
```cpp
//main.cpp
#include "Adapter.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) { 
	//Adapter* adt = new Adapter();
    Adaptee* ade = new Adaptee;
    Target* adt = new Adapter(ade);
    adt->Request();
    return 0; 
}
```

## Decorator(修饰器模式)

Decorator模式通过组合的方式提供了一种给类增加职责（操作）的方法。  
Decorator模式结构图

```cpp
//Decorator.h
#ifndef _DECORATOR_H_ 
#define _DECORATOR_H_
class Component {
public: 
    virtual ~Component();
    virtual void Operation();
protected: 
    Component();
};
class ConcreteComponent:public Component {
public: 
    ConcreteComponent();
    ~ConcreteComponent();
    void Operation();
};
class Decorator:public Component {
public: 
    Decorator(Component* com);
    virtual ~Decorator();
    void Operation();
protected: 
    Component* _com;
};
class ConcreteDecorator:public Decorator {
public: 
    ConcreteDecorator(Component* com);
    ~ConcreteDecorator();
    void Operation();
    void AddedBehavior();
};
#endif
```
```cpp
//Decorator.cpp
#include "Decorator.h"
#include <iostream>
Component::Component(){}
Component::~Component(){}
void Component::Operation(){}
ConcreteComponent::ConcreteComponent(){}
ConcreteComponent::~ConcreteComponent(){}
void ConcreteComponent::Operation() {
    std::cout<<"ConcreteComponent operation..."<<std::endl; 
}
Decorator::Decorator(Component* com) {this->_com = com;}
Decorator::~Decorator() { delete _com;}
void Decorator::Operation(){}
ConcreteDecorator::ConcreteDecorator(Component*com):Decorator(com){}
ConcreteDecorator::~ConcreteDecorator(){}
void ConcreteDecorator::AddedBehavior() {
    std::cout<<"ConcreteDecorator::AddedBehacior...."<<std::endl; 
} 
void ConcreteDecorator::Operation() {
    _com->Operation();
    this->AddedBehavior();
}
```
```cpp
//main.cpp
#include "Decorator.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    Component* com = new ConcreteComponent();
    Decorator* dec = new ConcreteDecorator(com);
    dec->Operation();
    delete dec;
    return 0; 
}
```
Decorator模式的讨论:  
为了多态，通过父类指针指向其具体子类，但是这就带来另外一个问题，当具体子类要添加新的职责，就必须向其父类添加一个这个职责的抽象接口，否则是通过父类指针是调用不到这个方法了。这样处于高层的父类就承载了太多的特征（方法），并且继承自这个父类的所有子类都不可避免继承了父类的这些接口，但是可能这并不是这个具体子类所需要的。而在Decorator模式提供了一种较好的解决方法，当需要添加一个操作的时候就可以通过Decorator模式来解决，你可以一步步添加新的职责

## Composite(组合模式)

将对象组合成树形结构以表示“部分--整体”的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。
Composite 模式结构图：
![composite](/assets/images/patterns/composite.png)

```cpp
//Component.h
#ifndef _COMPONENT_H_ 
#define _COMPONENT_H_
class Component {
public: 
    Component();
    virtual ~Component();
    virtual void Operation() = 0;
    virtual void Add(const Component& );
    virtual void Remove(const Component& );
    virtual Component* GetChild(int );
};
#endif
```
```cpp
//Component.cpp
#include "Component.h"
Component::Component(){}
Component::~Component(){}
void Component::Add(const Component& com) {}
Component* Component::GetChild(int index){return 0;}
void Component::Remove(const Component& com){}
```
```cpp
//Composite.h
#ifndef _COMPOSITE_H_ 
#define _COMPOSITE_H_
#include "Component.h" 
#include <vector> 
using namespace std;
class Composite:public Component {
public: 
    Composite();
    ~Composite();
public: 
    void Operation();
    void Add(Component* com);
    void Remove(Component* com);
    Component* GetChild(int index);
private: 
    vector<Component*> comVec;
};
#endif
```
```cpp
//Composite.cpp
#include "Composite.h" 
#include "Component.h"
#define NULL 0 //define NULL POINTOR
Composite::Composite() { 
//vector<Component*>::iterator itend = comVec.begin(); 
}
Composite::~Composite() {}
void Composite::Operation() {
    vector<Component*>::iterator comIter = comVec.begin();
    for (;comIter != comVec.end();comIter++) {
        (*comIter)->Operation(); 
    } 
}
void Composite::Add(Component* com) {comVec.push_back(com);}
void Composite::Remove(Component* com) {comVec.erase(&com);}
Component* Composite::GetChild(int index){return comVec[index]; }
```
```cpp
//Leaf.h
#ifndef _LEAF_H_ 
#define _LEAF_H_
#include "Component.h"
class Leaf:public Component { 
public: 
    Leaf();
    ~Leaf();
void Operation();
}; 

#endif
```
```cpp
//Leaf.cpp
#include "Leaf.h" 
#include <iostream>
using namespace std;
Leaf::Leaf(){}
Leaf::~Leaf(){}
void Leaf::Operation() {
    cout<<"Leaf operation....."<<endl; 
}
```
```cpp
//main.cpp
#include "Component.h" 
#include "Composite.h" 
#include "Leaf.h" 
#include <iostream> 
using namespace std;

int main(int argc,char* argv[]) {
    Leaf* l = new Leaf(); 
    l->Operation();
    Composite* com = new Composite();
    com->Add(l);
    com->Operation();
    Component* ll = com->GetChild(0);
    ll->Operation();
    return 0;
}
```

## Flyweight(享元模式)

Flyweight 模式以共享的方式高效的支持大量的细粒度对象，对象分为内部状态、外部状态。将可以被共享的状态作为内部状态存储在对象中，而外部状态在适当的时候作为参数传递给对象。  
当以下所有的条件都满足时，可以考虑使用享元模式：

+ 一个系统有大量的对象。
+ 这些对象耗费大量的内存。
+ 这些对象的状态中的大部分都可以外部化。
+ 这些对象可以按照内蕴状态分成很多的组，当把外蕴对象从对象中剔除时，每一个组都可以仅用一个对象代替。
+ 软件系统不依赖于这些对象的身份，换言之，这些对象可以是不可分辨的。

Flyweight模式结构图（不想被共享的对象UnshaerConcreteFlyweight，暂不讨论）
![Flyweight](/assets/images/patterns/flyweight.png)

```cpp
//Flyweight.h
#ifndef _FLYWEIGHT_H_ 
#define _FLYWEIGHT_H_
#include <string> 
using namespace std;
class Flyweight {
public:
    virtual ~Flyweight();
    virtual void Operation(const string& extrinsicState);
    string GetIntrinsicState();
protected: 
    Flyweight(string intrinsicState);
private: 
    string _intrinsicState;
};
class ConcreteFlyweight:public Flyweight {
public: 
    ConcreteFlyweight(string intrinsicState);
    ~ConcreteFlyweight();
    void Operation(const string& extrinsicState);
}; 
#endif
```
```cpp
//Flyweight.cpp
#include "Flyweight.h" 
#include <iostream> 
using namespace std;
Flyweight::Flyweight(string intrinsicState) {
    this->_intrinsicState = intrinsicState;
}
Flyweight::~Flyweight() {}
void Flyweight::Operation(const string& extrinsicState) {}
string Flyweight::GetIntrinsicState() {return this->_intrinsicState; }
ConcreteFlyweight::ConcreteFlyweight(string intrinsicState):Flyweight(intrinsicState) {
    cout<<"ConcreteFlyweight Build....."<<intrinsicState<<endl;
}
ConcreteFlyweight::~ConcreteFlyweight() {}
void ConcreteFlyweight::Operation(const string& extrinsicState) {
    cout<<"ConcreteFlyweight:内蕴["<<this->GetIntrinsicState()<<"] 外蕴["<<extrinsicState<<"]"<<endl; 
}
```
```cpp
//FlyweightFactory.h
#ifndef _FLYWEIGHTFACTORY_H_
#define _FLYWEIGHTFACTORY_H_
#include "Flyweight.h" 
#include <string> 
#include <vector> 
using namespace std;
class FlyweightFactory {
public: 
    FlyweightFactory();
    ~FlyweightFactory();
    Flyweight* GetFlyweight(const string& key);
private: 
    vector<Flyweight*> _fly;
}; 
#endif
```
```cpp
//FlyweightFactory.cpp
#include "FlyweightFactory.h" 
#include <iostream> 
#include <string> 
#include <cassert> 
using namespace std;
using namespace std;
FlyweightFactory::FlyweightFactory(){}
FlyweightFactory::~FlyweightFactory() {}
Flyweight* FlyweightFactory::GetFlyweight(const string& key){
    vector<Flyweight*>::iterator it = _fly.begin();
    for (; it != _fly.end();it++) { 
    	//找到了，就一起用，^_^ 
        if ((*it)->GetIntrinsicState() == key){ 
            cout<<"already created by users...."<<endl;
            return *it; 
        } 
    }
    Flyweight* fn = new ConcreteFlyweight(key);
    _fly.push_back(fn);
    return fn; 
}
```
```cpp
//main.cpp
#include "Flyweight.h" 
#include "FlyweightFactory.h" 
#include <iostream> 
using namespace std;

int main(int argc,char* argv[]){
    FlyweightFactory* fc = new FlyweightFactory();
    Flyweight* fw1 = fc->GetFlyweight("hello"); 
    Flyweight* fw2 = fc->GetFlyweight("world!");
    Flyweight* fw3 = fc->GetFlyweight("hello");
    return 0; 
}
```

## Facade(门面模式)

Fcade 模式在高层组合封装了子系统的接口，解耦了系统。隐藏了子系统的复杂性，使其更易使用。
结构图:
![facade](/assets/images/patterns/facade.png)

```cpp
//Facade.h
#ifndef _FACADE_H_
#define _FACADE_H_ 
class Subsystem1{
public:
    Subsystem1();
    ~Subsystem1();
    void Operation();
};
class Subsystem2{
public:
    Subsystem2();
    ~Subsystem2();
    void Operation(); 
};
class Facade{
public:
    Facade();
    ~Facade();
    void OperationWrapper(); 
private:
    Subsystem1* _subs1;
    Subsystem2* _subs2;
};
#endif
```
```cpp
//Facade.cpp
#include "Facade.h" 
#include <iostream>
using namespace std; 
Subsystem1::Subsystem1(){} 
Subsystem1::~Subsystem1(){} 
void Subsystem1::Operation(){
    cout<<"Subsystem1 operation.."<<endl;
}
Subsystem2::Subsystem2(){} 
Subsystem2::~Subsystem2(){} 
void Subsystem2::Operation(){cout<<"Subsystem2 operation.."<<endl;} 
Facade::Facade(){
    this->_subs1 = new Subsystem1();
    this->_subs2 = new Subsystem2();
}
Facade::~Facade(){
    delete _subs1;
    delete _subs2;
}
void Facade::OperationWrapper(){
    this->_subs1->Operation();
    this->_subs2->Operation();
}
```
```cpp
//main.cpp
#include "Facade.h" 
#include <iostream>
using namespace std; 
int main(int argc,char* argv[]){
    Facade* f = new Facade();
    f->OperationWrapper(); 
    return 0;
}
```

## Proxy(代理模式)

`Proxy Pattern`最大的好处就是实现了逻辑和实现的彻底解耦。  
结构图:
![proxy](/assets/images/patterns/proxy.png)

```cpp
//Proxy.h
#ifndef _PROXY_H_ 
#define _PROXY_H_

class Subject{
public: 
    virtual ~Subject();
    virtual void Request() = 0;
protected: 
    Subject();
};
class ConcreteSubject:public Subject { 
public:
    ConcreteSubject();
    ~ConcreteSubject();
    void Request();
};
class Proxy:public Subject{
public: 
    Proxy();
    Proxy(Subject* sub);
    ~Proxy();
    void Request();
private: 
    Subject* _sub; 
};
#endif
```
```cpp
#include "Proxy.h"
#include <iostream>
using namespace std;
Subject::Subject() {}
Subject::~Subject(){}
ConcreteSubject::ConcreteSubject() {}
ConcreteSubject::~ConcreteSubject(){}
void ConcreteSubject::Request() {
    cout<<"ConcreteSubject......request ...."<<endl;
}
Proxy::Proxy(){}
Proxy::Proxy(Subject* sub){_sub = sub;}
Proxy::~Proxy() {delete _sub; }
void Proxy::Request(){
    cout<<"Proxy request...."<<endl;
    _sub->Request(); 
}
```
```cpp
//main.cpp
#include "Proxy.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    Subject* sub = new ConcreteSubject();
    Proxy* p = new Proxy(sub);
    p->Request();
    return 0; 
}
```

## Template(模板方法模式)

Template 模式解决的问题：对于某一个业务逻辑在不同的对象中有不同的细节实现，但是逻辑的框架是相同的。将逻辑框架放在抽象基类中，并定义好细节的接口，子类中实现细节。Template模式利用多态的概念实现算法细节和高层接口的松耦合。  
Template Pattern 结构图:
![template](/assets/images/patterns/template.png)

```cpp
//template.h
#ifndef _TEMPLATE_H_
#define _TEMPLATE_H_ 

class AbstractClass{
public:
    virtual ~AbstractClass();
    void TemplateMethod(); 
protected:
    virtual void PrimitiveOperation1() = 0;
    virtual void PrimitiveOperation2() = 0;
    AbstractClass(); 
};

class ConcreteClass1:public AbstractClass{
public:
    ConcreteClass1();
    ~ConcreteClass1(); 
protected:
    void PrimitiveOperation1();
    void PrimitiveOperation2(); 
};

class ConcreteClass2:public AbstractClass{
public:
    ConcreteClass2();
    ~ConcreteClass2(); 
protected:
    void PrimitiveOperation1();
    void PrimitiveOperation2();
};
#endif
```
```cpp
//Template.cpp
#include "Template.h" 
#include <iostream>
using namespace std;
AbstractClass::AbstractClass(){} 
AbstractClass::~AbstractClass(){} 
void AbstractClass::TemplateMethod(){
    this->PrimitiveOperation1();
    this->PrimitiveOperation2();
} 
ConcreteClass1::ConcreteClass1(){} 
ConcreteClass1::~ConcreteClass1(){}
void ConcreteClass1::PrimitiveOperation1(){
    cout<<"ConcreteClass1...PrimitiveOperation1"<<endl;
} 
void ConcreteClass1::PrimitiveOperation2(){
    cout<<"ConcreteClass1...PrimitiveOperation2"<<endl;
}
ConcreteClass2::ConcreteClass2(){}
ConcreteClass2::~ConcreteClass2(){}
void ConcreteClass2::PrimitiveOperation1(){
    cout<<"ConcreteClass2...PrimitiveOperation1"<<endl;
}
void ConcreteClass2::PrimitiveOperation2(){
    cout<<"ConcreteClass2...PrimitiveOperation2"<<endl;
}
```
```cpp
//main.cpp
#include "Template.h" 
#include <iostream>
using namespace std; 
int main(int argc,char* argv[]){
    AbstractClass* p1 = new ConcreteClass1();
    AbstractClass* p2 = new ConcreteClass2(); p1->TemplateMethod();
    p2->TemplateMethod();
    return 0;
}
```

## Strategy(策略模式)

Strategy模式和Template模式要解决的问题是相同（类似）的，都是为了给业务逻辑（算法）具体实现和抽象接口之间的解耦。Strategy模式将逻辑（算法）封装到一个类（Context）里面，通过组合的方式将具体算法的实现在组合对象中实现，再通过委托的方式将抽象接口的实现委托给组合对象实现.  
Strategy Pattern 结构图:
![strategy](/assets/images/patterns/strategy.png)

```cpp
//strategy.h
#ifndef _STRATEGY_H_ 
#define _STRATEGY_H_
class Strategy {
public: 
    Strategy();
    virtual ~Strategy();
    virtual void AlgrithmInterface() = 0;
};
class ConcreteStrategyA:public Strategy {
public: 
    ConcreteStrategyA();
    virtual ~ConcreteStrategyA();
    void AlgrithmInterface();
};
class ConcreteStrategyB:public Strategy {
public:
    ConcreteStrategyB();
    virtual ~ConcreteStrategyB();
    void AlgrithmInterface();
};
#endif
```
```cpp
//strategy.cpp
#include "Strategy.h" 
#include <iostream> 
using namespace std;
Strategy::Strategy(){}
Strategy::~Strategy() {cout<<"~Strategy....."<<endl;}
void Strategy::AlgrithmInterface(){}
ConcreteStrategyA::ConcreteStrategyA(){}
ConcreteStrategyA::~ConcreteStrategyA() {
    cout<<"~ConcreteStrategyA....."<<endl; 
}
void ConcreteStrategyA::AlgrithmInterface(){
    cout<<"test ConcreteStrategyA....."<<endl;
}
ConcreteStrategyB::ConcreteStrategyB(){}
ConcreteStrategyB::~ConcreteStrategyB() {
    cout<<"~ConcreteStrategyB....."<<endl; 
}
void ConcreteStrategyB::AlgrithmInterface() {
    cout<<"test ConcreteStrategyB....."<<endl; 
}
```
```cpp
//context.h
#ifndef _CONTEXT_H_ 
#define _CONTEXT_H_
class Strategy;
class Context {
public:
    Context(Strategy* stg);
    ~Context();
    void DoAction(); 
private: 
    Strategy* _stg;
};
#endif
```
```cpp
//context.cpp
#include "Context.h" 
#include "Strategy.h" 
#include <iostream> 
using namespace std;
Context::Context(Strategy* stg) {_stg = stg;}
Context::~Context() {if (!_stg) delete _stg;}
void Context::DoAction() {_stg->AlgrithmInterface();}
```
```cpp
//main.cpp
#include "Context.h" 
#include "Strategy.h"
#include <iostream>
using namespace std;
int main(int argc,char* argv[]) {
    Strategy* ps = new ConcreteStrategyA();
    Context* pc = new Context(ps);
    pc->DoAction();
    if (NULL != pc) 
        delete pc;
    return 0; 
}
```

## state(状态模式)

每个人、事物在不同的状态下会有不同表现（动作），而一个状态又会在不同的表现下转移到下一个不同的状态（State）.State Pattern 将每一个分支都封装到独立的类中，将状态逻辑和动作实现进行分离。提高了系统的扩展性和可维护性。  
State Pattern结构图：
![state](/assets/images/patterns/state.png)

```cpp
State.h
#ifndef _STATE_H_ 
#define _STATE_H_
class Context; //前置声明
class State {
public: 
    State();
    virtual ~State();
    virtual void OperationInterface(Context* ) = 0;
    virtual void OperationChangeState(Context*) = 0;
protected: 
    bool ChangeState(Context* con,State* st);
};
class ConcreteStateA:public State {
public: 
    ConcreteStateA();
    virtual ~ConcreteStateA();
    virtual void OperationInterface(Context* );
    virtual void OperationChangeState(Context*);
};
class ConcreteStateB:public State {
public:
    ConcreteStateB();
    virtual ~ConcreteStateB();
    virtual void OperationInterface(Context* );
    virtual void OperationChangeState(Context*);
}; 
#endif
```
```cpp
//State.cpp
#include "State.h" 
#include "Context.h" 
#include <iostream> 
using namespace std;
State::State() {}
State::~State() {}
void State::OperationInterface (Context* con) {
    cout<<"State::.."<<endl;
}
bool State::ChangeState(Context* con,State* st) {
    con->ChangeState(st);
    return true;
}
void State::OperationChangeState(Context* con){}
ConcreteStateA::ConcreteStateA() {}
ConcreteStateA::~ConcreteStateA() {}
void ConcreteStateA::OperationInterface (Context* con) {
    cout<<"ConcreteStateA::OperationInterface ......"<<endl;
}
void ConcreteStateA::OperationChangeState(Context* con) {
    OperationInterface(con);
    this->ChangeState(con,new ConcreteStateB()); 
}
ConcreteStateB::ConcreteStateB() {}
ConcreteStateB::~ConcreteStateB(){}
void ConcreteStateB::OperationInterface (Context* con) {
    cout<<"ConcreteStateB::OperationInterface......"<<endl; 
}
void ConcreteStateB::OperationChangeState (Context* con) {
    OperationInterface(con);
    this->ChangeState(con,new ConcreteStateA()); 
}
```
```cpp
//Context.h
#ifndef _CONTEXT_H_ 
#define _CONTEXT_H_
class State; /** * **/ 
class Context {
public: 
    Context();
    Context(State* state);
    ~Context();
    void OprationInterface();
    void OperationChangState();
private: 
    friend class State;
    bool ChangeState(State* state);
private: 
    State* _state;
};
#endif
```
```cpp
//Context.cpp
#include "Context.h" 
#include "State.h"
Context::Context(){}
Context::Context(State* state) {this->_state = state; }
Context::~Context() {delete _state; }
void Context::OprationInterface() {_state->OperationInterface(this); }
bool Context::ChangeState(State* state) {
 ///_state->ChangeState(this,state); 
    this->_state = state;
    return true; 
}
void Context::OperationChangState() {
    _state->OperationChangeState(this);
}
```
```cpp
//main.cpp
#include "Context.h" 
#include "State.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    State* st = new ConcreteStateA();
    Context* con = new Context(st);
    con->OperationChangState();
    con->OperationChangState();
    con->OperationChangState();
    if (con != NULL) delete con;
    if (st != NULL) st = NULL;
    return 0;
}
```
State模式在实现中的两个关键点：

+ 将State声明为Context的友元类（friend class），其作用是让State模式访问Context的protected接口ChangeSate（）
+ State及其子类中的操作都将Context*传入作为参数，其主要目的是State类可以通过这个指针调用Context中的方法（在本示例代码中没有体现）。这也是State模式和Strategy模式的最大区别所在
总结：  
State模式很好地实现了对象的状态逻辑和动作实现的分离，状态逻辑分布在State的派生类中实现，而动作实现则可以放在Context类中实现（这也是为什么State派生类需要拥有一个指向Context的指针）。这使得两者的变化相互独立，改变State的状态逻辑可以很容易复用Context的动作，也可以在不影响State派生类的前提下创建Context的子类来更改或替换动作实现

## Observer(观察者模式)

Observer模式要解决的问题为：建立一个一（Subject）对多（Observer）的依赖关系，并且做到当“一”变化的时候，依赖这个“一”的多也能够同步改变.
Observer Pattern 结构图:
![observer](/assets/images/patterns/observer.png)

注：这里的目标Subject提供依赖于它的观察者Observer的注册（Attach）和注销（Detach）操作，并且提供了使得依赖于它的所有观察者同步的操作（Notify）。观察者Observer则提供一个Update操作，注意这里的Observer的Update操作并不在Observer改变了Subject目标状态的时候就对自己进行更新，这个更新操作要延迟到Subject对象发出Notify通知所有Observer进行修改（调用Update）。

```cpp
//Subject.h
#ifndef _SUBJECT_H_
#define _SUBJECT_H_
#include <list>
#include <string> 
using namespace std;
typedef string State;
class Observer;
class Subject {
public: 
    virtual ~Subject();
    virtual void Attach(Observer* obv);
    virtual void Detach(Observer* obv);
    virtual void Notify();
    virtual void SetState(const State& st) = 0;
    virtual State GetState() = 0;
protected: 
    Subject();
private: 
    list<Observer* >* _obvs;
};
class ConcreteSubject:public Subject {
public: 
    ConcreteSubject();
    ~ConcreteSubject();
    State GetState();
    void SetState(const State& st);
private: 
    State _st;
};
#endif
```
```cpp
//Subject.cpp
#include "Subject.h" 
#include "Observer.h"
#include <iostream> 
#include <list> 
using namespace std;
typedef string state;
Subject::Subject(){
    //****在模板的使用之前一定要new，创建 
    _obvs = new list<Observer*>;
}
Subject::~Subject() {}
void Subject::Attach(Observer* obv) {_obvs->push_front(obv);}
void Subject::Detach(Observer* obv) {
    if (obv != NULL)
        _obvs->remove(obv);
}
void Subject::Notify() {
    list<Observer*>::iterator it;
    it = _obvs->begin();
    for (;it != _obvs->end();it++) { 
    //关于模板和iterator的用法
        (*it)->Update(this); 
    } 
}
ConcreteSubject::ConcreteSubject(){ _st = '\0'; }
ConcreteSubject::~ConcreteSubject(){}
State ConcreteSubject::GetState() { return _st; }
void ConcreteSubject::SetState(const State& st) { _st = st; }
```
```cpp
//Observer.h
#ifndef _OBSERVER_H_ 
#define _OBSERVER_H_
#include "Subject.h"
#include <string> 
using namespace std;
typedef string State;
class Observer {
public: 
    virtual ~Observer();
    virtual void Update(Subject* sub) = 0;
    virtual void PrintInfo() = 0;
protected:
    Observer();
    State _st;
};
class ConcreteObserverA:public Observer {
public: 
    virtual Subject* GetSubject(); 
    ConcreteObserverA(Subject* sub);
    virtual ~ConcreteObserverA();
    //传入Subject作为参数，这样可以让一个View属于多个的Subject。
    void Update(Subject* sub);
    void PrintInfo();
private: 
    Subject* _sub;
};
class ConcreteObserverB:public Observer {
public: 
    virtual Subject* GetSubject();
    ConcreteObserverB(Subject* sub);
    virtual ~ConcreteObserverB();
    //传入Subject作为参数，这样可以让一个View属于多个的Subject。
    void Update(Subject* sub);
    void PrintInfo();
private: 
    Subject* _sub;
};
#endif
```
```cpp
//Observer.cpp
#include "Observer.h" 
#include "Subject.h"
#include <iostream> 
#include <string> 
using namespace std;
Observer::Observer(){_st = '\0';}
Observer::~Observer(){} 
ConcreteObserverA::ConcreteObserverA(Subject* sub) {
    _sub = sub;
    _sub->Attach(this);
}
ConcreteObserverA::~ConcreteObserverA() {
    _sub->Detach(this);
    if (_sub != 0) {
        delete _sub;
    } 
}
Subject* ConcreteObserverA::GetSubject() {return _sub;}
void ConcreteObserverA::PrintInfo(){
    cout<<"ConcreteObserverA observer.... "<<_sub->GetState()<<endl; 
}
void ConcreteObserverA::Update(Subject* sub) {
    _st = sub->GetState();
    PrintInfo();
}
ConcreteObserverB::ConcreteObserverB(Subject* sub) {
    _sub = sub;
    _sub->Attach(this); 
}
ConcreteObserverB::~ConcreteObserverB() {
    _sub->Detach(this);
    if (_sub != 0) {
        delete _sub; 
    } 
}
Subject* ConcreteObserverB::GetSubject() {return _sub; }
void ConcreteObserverB::PrintInfo() {
    cout<<"ConcreteObserverB observer.... "<<_sub->GetState()<<endl; 
}
void ConcreteObserverB::Update(Subject* sub) {
    _st = sub->GetState();
    PrintInfo(); 
}
```
```cpp
//main.cpp
#include "Subject.h" 
#include "Observer.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    ConcreteSubject* sub = new ConcreteSubject();
    Observer* o1 = new ConcreteObserverA(sub);
    Observer* o2 = new ConcreteObserverB(sub);
    sub->SetState("old");
    sub->Notify();
    sub->SetState("new"); //也可以由Observer调用
    sub->Notify();
    return 0; 
}
```

## Memento(备忘录模式)

Memento 模式的关键是在不破坏封装的前提下，捕获并保存一个类的内部状态，这样就可以利用该保存的状态实施恢复操作。  
Memento 模式结构图：
![mememto](/assets/images/patterns/mememto.png)

```cpp
//Memento.h
#ifndef _MEMENTO_H_ 
#define _MEMENTO_H_
#include <string> 
using namespace std;
class Memento;
class Originator{
public: 
    typedef string State;
    Originator();
    Originator(const State& sdt);
    ~Originator();
    Memento* CreateMemento();
    void SetMemento(Memento* men);
    void RestoreToMemento(Memento* mt);
    State GetState();
    void SetState(const State& sdt);
    void PrintState();
private: 
    State _sdt;
    Memento* _mt; 
};
class Memento {
private: 
    //这是最关键的地方，将Originator为friend类，可以访问内部信息，但是其他类不能访问 
    friend class Originator; 
    typedef string State;
    Memento();
    Memento(const State& sdt);
    ~Memento();
    void SetState(const State& sdt);
    State GetState();
private: 
        State _sdt;
};
#endif
```
```cpp
//Memento.cpp
#include "Memento.h"
#include <iostream>
using namespace std;
typedef string State;
Originator::Originator() {
    _sdt = "";
    _mt = 0; 
}
Originator::Originator(const State& sdt){
    _sdt = sdt;
    _mt = 0; 
}
Originator::~Originator() {}
Memento* Originator::CreateMemento() {return new Memento(_sdt); }
State Originator::GetState() {return _sdt; }
void Originator::SetState(const State& sdt){ _sdt = sdt;}
void Originator::PrintState() { cout<<this->_sdt<<"....."<<endl; }
void Originator::SetMemento(Memento* men){}
void Originator::RestoreToMemento(Memento* mt) { this->_sdt = mt->GetState();}
//class Memento
Memento::Memento() {}
Memento::Memento(const State& sdt) { _sdt = sdt;}
State Memento::GetState() { return _sdt;}
void Memento::SetState(const State& sdt) { _sdt = sdt; }
```
```cpp
//main.cpp
#include "Memento.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    Originator* o = new Originator();
    o->SetState("old"); //备忘前状态
    o->PrintState();
    Memento* m = o->CreateMemento(); //将状态备忘
    o->SetState("new"); //修改状态
    o->PrintState();
    o->RestoreToMemento(m); //恢复修改前状态
    o->PrintState();
    return 0;
}
```

代码说明:
Memento模式的关键就是friend class Originator;我们可以看到，Memento的接口都声明为private，而将Originator声明为Memento的友元类。我们将Originator的状态保存在Memento类中，而将Memento接口private起来，也就达到了封装的功效。
在Originator类中我们提供了方法让用户后悔：RestoreToMemento(Memento* mt)我们可以通过这个接口让用户后悔。

## Mediator(中介模式)

Mediator模式将对象间的交互和通讯封装在一个类中，各个对象间的通信不必显示去声明和引用，将多对多的通信转化为一对多的通信，大大降低了系统的复杂性能。
通过Mediator，各个Colleage就不必维护各自通信的对象和通信协议，降低了系统的耦合性。
Mediator模式还有一个很显著额特点就是将控制集中，集中的优点就是便于管理，也正符合了OO设计中的每个类的职责要单一和集中的原则
Mediator模式结构图
![mediator](/assets/images/patterns/mediator.png)

```cpp
//Colleage.h
#ifndef _COLLEAGE_H_ 
#define _COLLEAGE_H_
#include <string> 
using namespace std;
class Mediator;
class Colleage {
public:
    virtual ~Colleage();
    virtual void Aciton() = 0;
    virtual void SetState(const string& sdt) = 0;
    virtual string GetState() = 0; 
protected:
    Colleage();
    Colleage(Mediator* mdt);
    Mediator* _mdt;
};
class ConcreteColleageA:public Colleage {
public:
    ConcreteColleageA();
    ConcreteColleageA(Mediator* mdt);
    ~ConcreteColleageA();
    void Aciton();
    void SetState(const string& sdt);
    string GetState();
private: 
    string _sdt;
};
class ConcreteColleageB:public Colleage {
public: 
    ConcreteColleageB();
    ConcreteColleageB(Mediator* mdt);
    ~ConcreteColleageB();
    void Aciton();
    void SetState(const string& sdt);
    string GetState();
private: 
    string _sdt;
};
#endif
```
```cpp
//Colleage.cpp
#include "Mediator.h"
#include "Colleage.h"
#include <iostream> 
using namespace std;
Colleage::Colleage() {
//_sdt = " "; 
}
Colleage::Colleage(Mediator* mdt) {
    this->_mdt = mdt;
    //_sdt = " "; 
}
Colleage::~Colleage() {}
ConcreteColleageA::ConcreteColleageA(){}
ConcreteColleageA::~ConcreteColleageA() {}
ConcreteColleageA::ConcreteColleageA(Mediator* mdt):Colleage(mdt) {}
string ConcreteColleageA::GetState() {return _sdt;}
void ConcreteColleageA::SetState(const string& sdt) { _sdt = sdt;}
void ConcreteColleageA::Aciton() {
    _mdt->DoActionFromAtoB();
    cout<<"State of ConcreteColleageB:"<<" "<<this->GetState()<<endl; 
}
ConcreteColleageB::ConcreteColleageB(){}
ConcreteColleageB::~ConcreteColleageB() {}
ConcreteColleageB::ConcreteColleageB(Mediator* mdt):Colleage(mdt){}
void ConcreteColleageB::Aciton() {
    _mdt->DoActionFromBtoA();
    cout<<"State of ConcreteColleageB:"<<" "<<this->GetState()<<endl; 
}
string ConcreteColleageB::GetState() { return _sdt; }
void ConcreteColleageB::SetState(const string& sdt) { _sdt = sdt; }
```
```cpp
//Mediator.h
#ifndef _MEDIATOR_H_ 
#define _MEDIATOR_H_
class Colleage;
class Mediator {
public: 
    virtual ~Mediator();
    virtual void DoActionFromAtoB() = 0;
    virtual void DoActionFromBtoA() = 0;
protected: 
    Mediator();
};
class ConcreteMediator:public Mediator{
public:
    ConcreteMediator();
    ConcreteMediator(Colleage* clgA,Colleage* clgB);
    ~ConcreteMediator();
    void SetConcreteColleageA(Colleage* clgA);
    void SetConcreteColleageB(Colleage* clgB);
    Colleage* GetConcreteColleageA();
    Colleage* GetConcreteColleageB();
    void IntroColleage(Colleage* clgA,Colleage* clgB);
    void DoActionFromAtoB();
    void DoActionFromBtoA();
private: 
    Colleage* _clgA;
    Colleage* _clgB;
};
#endif
``
```cpp
//Mediator.cpp
#include "Mediator.h"
#include "Colleage.h"
Mediator::Mediator() {}
Mediator::~Mediator(){}
ConcreteMediator::ConcreteMediator(){}
ConcreteMediator::~ConcreteMediator() {}
ConcreteMediator::ConcreteMediator(Colleage* clgA,Colleage* clgB) {
    this->_clgA = clgA;
    this->_clgB = clgB;
}
void ConcreteMediator::DoActionFromAtoB() {
    _clgB->SetState(_clgA->GetState());
}
void ConcreteMediator::SetConcreteColleageA(Colleage* clgA) {this->_clgA = clgA; }
void ConcreteMediator::SetConcreteColleageB(Colleage* clgB) {this->_clgB = clgB;}
Colleage* ConcreteMediator::GetConcreteColleageA(){return _clgA; }
Colleage* ConcreteMediator::GetConcreteColleageB(){return _clgB;}
void ConcreteMediator::IntroColleage(Colleage* clgA,Colleage* clgB) {
    this->_clgA = clgA;
    this->_clgB = clgB; 
}
void ConcreteMediator::DoActionFromBtoA() {
    _clgA->SetState(_clgB->GetState()); 
}
```
```cpp
//main.cpp
#include "Mediator.h"
#include "Colleage.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    ConcreteMediator* m = new ConcreteMediator();
    ConcreteColleageA* c1 = new ConcreteColleageA(m); 
    ConcreteColleageB* c2 = new ConcreteColleageB(m);

    m->IntroColleage(c1,c2);

    c1->SetState("old"); 
    c2->SetState("old"); 
    c1->Aciton(); 
    c2->Aciton();
    cout<<endl;

    c1->SetState("new"); 
    c1->Aciton(); 
    c2->Aciton(); 
    cout<<endl;

    c2->SetState("old"); 
    c2->Aciton(); 
    c1->Aciton();
    return 0;
}
```

## Command(命令模式)

Command模式通过将请求封装到一个对象（Command）中，并将请求的接受者存放到具体的ConcreteCommand类中（Receiver）中，从而实现调用操作的对象和操作的具体实现者之间的解耦。  
Command 模式结构图：
![command](/assets/images/patterns/command.png)

```cpp
//Receiver.h
#ifndef _RECEIVER_H_ 
#define _RECEIVER_H_
class Reciever {
public: 
    Reciever();
    ~Reciever();
    void Action();
};
#endif
```
```cpp
//Receiver.cpp
#include "Receiver.h"
#include <iostream>
Reciever::Reciever() {}
Reciever::~Reciever(){}
void Reciever::Action() {
    std::cout<<"Reciever action......."<<std::endl;
}
```
```cpp
//Command.h
#ifndef _COMMAND_H_ 
#define _COMMAND_H_
class Reciever;
class Command {
public: 
    virtual ~Command();
    virtual void Excute() = 0;
protected: 
    Command();
};
class ConcreteCommand:public Command {
public: 
    ConcreteCommand(Reciever* rev);
    ~ConcreteCommand();
    void Excute();
private: 
    Reciever* _rev; 
};
#endif
```
```cpp
//Command.cpp
#include "Command.h"
#include "Receiver.h"
#include <iostream>
Command::Command() {}
Command::~Command() {}
void Command::Excute(){}
ConcreteCommand::ConcreteCommand(Reciever* rev) { this->_rev = rev;}
ConcreteCommand::~ConcreteCommand() { delete this->_rev; }
void ConcreteCommand::Excute() {
    _rev->Action();
    std::cout<<"ConcreteCommand..."<<std::endl;
}
```
```cpp
//Invoker.h
#ifndef _INVOKER_H_ 
#define _INVOKER_H_
class Command;
class Invoker {
public: 
    Invoker(Command* cmd);
    ~Invoker();
    void Invoke();
private: 
    Command* _cmd;
};
#endif
```
```cpp
//Invoker.cpp
#include "Invoker.h"
#include "Command.h"
#include <iostream>
Invoker::Invoker(Command* cmd) { _cmd = cmd;}
Invoker::~Invoker() {delete _cmd;}
void Invoker::Invoke() { _cmd->Excute(); }
```
```cpp
//main.cpp
#include "Command.h" 
#include "Invoker.h" 
#include "Receiver.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    Reciever* rev = new Reciever();
    Command* cmd = new ConcreteCommand(rev);
    Invoker* inv = new Invoker(cmd);
    inv->Invoke();
    return 0; 
}
```
将请求接收者的处理抽象出来作为参数传给Command对象，实际也就是回调的机制（Callback）来实现这一点，也就是说将处理操作方法地址（在对象内部）通过参数传递给Command对象（涉及到C++成员函数指针的概念）。简单示例:

```cpp
//Receiver.h
#ifndef _RECEIVER_H_ 
#define _RECEIVER_H_
class Reciever {
public: 
    Reciever();
    ~Reciever();
    void Action();
};
#endif
```
```cpp
//Receiver.cpp
#include "Receiver.h"
#include <iostream>
Reciever::Reciever() {}
Reciever::~Reciever(){}
void Reciever::Action() {
    std::cout<<"Reciever action......."<<std::endl;
}
```
```cpp
//Command.h
#ifndef _COMMAND_H_ 
#define _COMMAND_H_
class Command {
public: 
    virtual ~Command(){}
    virtual void Excute() = 0;
protected:
    Command(){}
};

template<class Reciever> 
class SimpleCommand:public Command{
public: 
    typedef void (Reciever::* Action)();
    SimpleCommand(Reciever* rev,Action act) {
        _rev = rev;
        _act = act; 
    }
    virtual void Excute() { (_rev->* _act)();}
    ~SimpleCommand() { delete _rev;}
private: 
    Reciever* _rev;
    Action _act; 
};
#endif
```
```cpp
//main.cpp
#include "Command.h"
#include "Receiver.h"
#include <iostream>
using namespace std;
int main(int arc,char* argv[]) {
    Reciever* rev = new Reciever();
    Command* cmd = new SimpleCommand<Reciever>(rev,&Reciever::Action);
    cmd->Excute();
    return 0; 
}
```

## Visitor(访问者模式)

Visitor模式：将更新（变更）封装到一个类中（访问操作），并由待更改类提供一个接收接口，则可在不破坏类的前提下，为类提供增加新的新操作。
Visitor模式结构图
![visitor](/assets/images/patterns/visitor.png)
Visitor模式的关键是双分派（Double-Dispatch）的技术：Accept（）操作是一个双分派的操作，具体调用哪个Accept（）操作，有两个决定因素

+ Element类型
+ Visitor类型。

```cpp
//Visitor.h
#ifndef _VISITOR_H_
#define _VISITOR_H_ 
class Element; 
class Visitor{
public:
    virtual ~Visitor();
    virtual void VisitConcreteElementA(Element* elm) = 0;
    virtual void VisitConcreteElementB(Element* elm) = 0;
protected:
    Visitor();
};
class ConcreteVisitorA:public Visitor{
public:
    ConcreteVisitorA();
    virtual ~ConcreteVisitorA();
    virtual void VisitConcreteElementA(Element* elm);
    virtual void VisitConcreteElementB(Element* elm);
}; 
class ConcreteVisitorB:public Visitor{
public:
    ConcreteVisitorB();
    virtual ~ConcreteVisitorB();
    virtual void VisitConcreteElementA(Element* elm);
    virtual void VisitConcreteElementB(Element* elm);
}; 
#endif
```
```cpp
//Visitor.cpp
#include "Visitor.h"
#include "Element.h" 
#include <iostream>
using namespace std;

Visitor::Visitor(){} 
Visitor::~Visitor(){} 
ConcreteVisitorA::ConcreteVisitorA(){}
ConcreteVisitorA::~ConcreteVisitorA(){}
void ConcreteVisitorA::VisitConcreteElementA(Element* elm){
    cout<<"i will visit ConcreteElementA..."<<endl;
} 
void ConcreteVisitorA::VisitConcreteElementB(Element* elm){
    cout<<"i will visit ConcreteElementB..."<<endl;
} 
ConcreteVisitorB::ConcreteVisitorB(){}
ConcreteVisitorB::~ConcreteVisitorB(){} 
void ConcreteVisitorB::VisitConcreteElementA(Element* elm){
    cout<<"i will visit ConcreteElementA..."<<endl;
} 
void ConcreteVisitorB::VisitConcreteElementB(Element* elm){
    cout<<"i will visit ConcreteElementB..."<<endl;
}
```
```cpp
//Element.h
#ifndef _ELEMENT_H_
#define _ELEMENT_H_ 
class Visitor; 
class Element{
public:
    virtual ~Element();
    virtual void Accept(Visitor* vis) = 0;
protected:
    Element();
}; 
class ConcreteElementA:public Element{
public:
    ConcreteElementA();
    ~ConcreteElementA();
    void Accept(Visitor* vis);
}; 

class ConcreteElementB:public Element
{
public:
    ConcreteElementB();
    ~ConcreteElementB();
    void Accept(Visitor* vis);
}; 
#endif
```
```cpp
//Element.cpp
#include "Element.h"
#include "Visitor.h" 
#include <iostream>
using namespace std; 

Element::Element(){}
Element::~Element(){}
void Element::Accept(Visitor* vis){} 
ConcreteElementA::ConcreteElementA(){} 
ConcreteElementA::~ConcreteElementA(){} 
void ConcreteElementA::Accept(Visitor* vis){
    vis->VisitConcreteElementA(this);
    cout<<"visiting ConcreteElementA..."<<endl;
} 
ConcreteElementB::ConcreteElementB(){}
ConcreteElementB::~ConcreteElementB(){} 
void ConcreteElementB::Accept(Visitor* vis){
    cout<<"visiting ConcreteElementB..."<<endl;
    vis->VisitConcreteElementB(this);
}
```
```cpp
//main.cpp
#include "Element.h"
#include "Visitor.h" 
#include <iostream>
using namespace std; 
int main(int argc,char* argv[]){
    Visitor* vis = new ConcreteVisitorA();
    Element* elm = new ConcreteElementA();
    elm->Accept(vis); 
    return 0;
}
```
Visitor模式的缺点

+ 破坏了封装性
+ ConcreteElement的扩展很困难：每增加一个Element的子类，就要修改Visitor的接口，使得可以提供给这个新增加的子类的访问机制。

## Chain of Responsibility(职责链模式)

Chain of Responsibility模式:将可能处理一个请求的对象链接成一个链，并将请求在这个链上传递，直到有对象处理该请求（可能需要提供一个默认处理所有请求的类，例如MFC中的CwinApp类）。
Chain of Responsibility Pattern 结构图：
![chain](/assets/images/patterns/chain_of_responsibility.png)

```cpp
//Handle.h
#ifndef _HANDLE_H_ 
#define _HANDLE_H_
class Handle {
public: 
    virtual ~Handle();
    virtual void HandleRequest() = 0;
    void SetSuccessor(Handle* succ);
    Handle* GetSuccessor();
protected: 
    Handle();
    Handle(Handle* succ);
private: 
    Handle* _succ; 
};

class ConcreteHandleA:public Handle {
public: 
    ConcreteHandleA();
    ~ConcreteHandleA();
    ConcreteHandleA(Handle* succ);
    void HandleRequest();
};

class ConcreteHandleB:public Handle { 
public: 
    ConcreteHandleB();
    ~ConcreteHandleB();
    ConcreteHandleB(Handle* succ);
    void HandleRequest();
};

#endif
```
```cpp
//Handle.cpp
#include "Handle.h"
#include <iostream>
using namespace std;
Handle::Handle(){ _succ = 0;}
Handle::~Handle() { delete _succ; }
Handle::Handle(Handle* succ) { this->_succ = succ; }
void Handle::SetSuccessor(Handle* succ) { _succ = succ;}
Handle* Handle::GetSuccessor() {return _succ;}
void Handle::HandleRequest() {}
ConcreteHandleA::ConcreteHandleA() {}
ConcreteHandleA::ConcreteHandleA(Handle* succ):Handle(succ) {}
ConcreteHandleA::~ConcreteHandleA() {}
void ConcreteHandleA::HandleRequest() {
    if (this->GetSuccessor() != 0) {
        cout<<"ConcreteHandleA 我把处理权给后继节点....."<<endl; 
        this->GetSuccessor()->HandleRequest(); 
    } else { 
        cout<<"ConcreteHandleA 没有后继了，我必须自己处理...."<<endl; 
    } 
}
ConcreteHandleB::ConcreteHandleB() {}
ConcreteHandleB::ConcreteHandleB(Handle* succ):Handle(succ) {}
ConcreteHandleB::~ConcreteHandleB() {}
void ConcreteHandleB::HandleRequest() {
    if (this->GetSuccessor() != 0) {
        cout<<"ConcreteHandleB 我把处理权给后继节点....."<<endl; 
        this->GetSuccessor()->HandleRequest(); 
    } else { 
        cout<<"ConcreteHandleB 没有后继了，我必须自己处理...."<<endl;
    }
}
```
```cpp
//main.cpp
#include "Handle.h"
#include <iostream>
using namespace std;
int main(int argc,char* argv[]) {
    Handle* h1 = new ConcreteHandleA();
    Handle* h2 = new ConcreteHandleB();
    h1->SetSuccessor(h2);
    h1->HandleRequest();
    return 0; 
}
```

## Iterator(迭代器模式)

Iterator模式用来解决对一个聚合对象的遍历问题，将对聚合的遍历封装到一个类中进行，这样就避免了暴露这个聚合对象的内部表示的可能.  
Iterator 模式结构图
![iterator](/assets/images/patterns/iterator.png)

```cpp
//Aggregate.h
#ifndef _AGGREGATE_H_ 
#define _AGGREGATE_H_
class Iterator;
typedef int Object;
class Interator;
class Aggregate {
public: 
    virtual ~Aggregate();
    virtual Iterator* CreateIterator() = 0;
    virtual Object GetItem(int idx) = 0;
    virtual int GetSize() = 0;
protected: 
    Aggregate();
};
class ConcreteAggregate:public Aggregate {
public: 
    enum { SIZE = 3};    
    ConcreteAggregate();
    ~ConcreteAggregate();
    Iterator* CreateIterator();
    Object GetItem(int idx);
    int GetSize();
private: 
    Object _objs[SIZE];
};
#endif
```
```cpp
//Aggregate.cpp
#include "Aggregate.h" 
#include "Iterator.h"
#include <iostream> 
using namespace std;
Aggregate::Aggregate() {}
Aggregate::~Aggregate() {}
ConcreteAggregate::ConcreteAggregate(){
    for (int i = 0; i < SIZE; i++) 
        _objs[i] = i; 
}
ConcreteAggregate::~ConcreteAggregate(){}
Iterator* ConcreteAggregate::CreateIterator(){
    return new ConcreteIterator(this);
}
Object ConcreteAggregate::GetItem(int idx) {
    if (idx < this->GetSize())  return _objs[idx];
    else return -1; 
}
int ConcreteAggregate::GetSize() { return SIZE; }
```
```cpp
//Iterator.h
#ifndef _ITERATOR_H_ 
#define _ITERATOR_H_
class Aggregate;
typedef int Object;
class Iterator {
public: 
    virtual ~Iterator();
    virtual void First() = 0;
    virtual void Next() = 0;
    virtual bool IsDone() = 0;
    virtual Object CurrentItem() = 0;
protected: 
    Iterator();
};
class ConcreteIterator:public Iterator{
public: 
    ConcreteIterator(Aggregate* ag , int idx = 0);
    ~ConcreteIterator();
    void First();
    void Next();
    bool IsDone();
    Object CurrentItem();
private:
    Aggregate* _ag;
    int _idx;
};
#endif
```
```cpp
//Iterator.cpp
#include "Iterator.h" 
#include "Aggregate.h" 
#include <iostream> 
using namespace std;
Iterator::Iterator() {}
Iterator::~Iterator() {}
ConcreteIterator::ConcreteIterator(Aggregate* ag , int idx){
    this->_ag = ag;
    this->_idx = idx; 
}
ConcreteIterator::~ConcreteIterator() {}
Object ConcreteIterator::CurrentItem(){return _ag->GetItem(_idx); }
void ConcreteIterator::First(){ _idx = 0; }
void ConcreteIterator::Next() {
    if (_idx < _ag->GetSize()) 
        _idx++;
}
bool ConcreteIterator::IsDone() { return (_idx == _ag->GetSize());}
```
```cpp
//main.cpp
#include "Iterator.h" 
#include "Aggregate.h"
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]) {
    Aggregate* ag = new ConcreteAggregate();
    Iterator* it = new ConcreteIterator(ag);
    for (; !(it->IsDone()) ; it->Next()) { 
        cout<<it->CurrentItem()<<endl; 
    }
    return 0; 
}
```

## Interpreter(解释器模式)

Interpreter模式的目的就是提供一个一门定义语言的语法表示的解释器，然后通过这个解释器来解释语言中的句子。  
Interpreter模式结构图：
![interpreter](/assets/images/patterns/interpreter.png)

```cpp
//Context.h
#ifndef _CONTEXT_H_ 
#define _CONTEXT_H_
class Context {
public:
    Context();
    ~Context();
};
#endif
```
```cpp
//Context.cpp
#include "Context.h"
Context::Context() {}
Context::~Context() {}
```
```cpp
//Interpret.h
#ifndef _INTERPRET_H_
#define _INTERPRET_H_
#include "Context.h" 
#include <string> 
using namespace std;
class AbstractExpression {
public: 
    virtual ~AbstractExpression();
    virtual void Interpret(const Context& c);
protected: 
    AbstractExpression();
};
class TerminalExpression:public AbstractExpression {
public: 
    TerminalExpression(const string& statement);
    ~ TerminalExpression();
    void Interpret(const Context& c);
private: 
    string _statement; 
};

class NonterminalExpression:public AbstractExpression {
public:
    NonterminalExpression(AbstractExpression* expression,int times);
    ~ NonterminalExpression();
    void Interpret(const Context& c);
private: 
    AbstractExpression* _expression;
    int _times; 
};
#endif
```
```cpp
//Interpret.cpp
#include "Interpret.h" 
#include <iostream> 
using namespace std;
AbstractExpression::AbstractExpression() {}
AbstractExpression::~AbstractExpression(){}
void AbstractExpression::Interpret(const Context& c) {}
TerminalExpression::TerminalExpression(const string& statement){
    this->_statement = statement; 
}
TerminalExpression::~TerminalExpression(){}
void TerminalExpression::Interpret(const Context& c) {
    cout<<this->_statement<<" TerminalExpression"<<endl;
}
NonterminalExpression::NonterminalExpression(AbstractExpression* expression,int times){
    this->_expression = expression;
    this->_times = times; 
}
NonterminalExpression::~NonterminalExpression() {}
void NonterminalExpression::Interpret(const Context& c) {
    for (int i = 0; i < _times ; i++) {
        this->_expression->Interpret(c); 
    } 
}
```
```cpp
//cpp
#include "Context.h" 
#include "Interpret.h" 
#include <iostream> 
using namespace std;
int main(int argc,char* argv[]){
    Context* c = new Context();
    AbstractExpression* te = new TerminalExpression("hello");
    AbstractExpression* nte = new NonterminalExpression(te,2);
    nte->Interpret(*c);
    return 0; 
}
```
