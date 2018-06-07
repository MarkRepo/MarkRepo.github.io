---
title: 原型模式(ProtoType)
description: 
categories: 设计模式
tags:
---

## prototype 模式结构图

![Prototype](/assets/images/prototype.png)

## 实现

```c++
//prototype.h

#ifndef _PROTOTYPE_H
#define _PROTOTYPE_H

class Prototype
{

	public:
		virtual ~Prototype();
		virtual Prototype* Clone() const = 0;
	protected:
		Prototype();
	private:
};

class ConcretePrototype : public Prototype
{

	public:
		ConcretePrototype();
		ConcretePrototype(const ConcretePrototype &cp);
		~ConcretePrototype();
		Prototype* Clone() const;
	protected:
	private:
};

#endif
```

```c++
//prototype.cpp

#include "prototype.h"
#include <iostream>
using namespace std;

Prototype::Prototype()
{

}

Prototype::~Prototype()
{

}

Prototype* Prototype::Clone() const
{
	return 0;
}

ConcretePrototype::ConcretePrototype()
{

}

ConcretePrototype::~ConcretePrototype()
{

}

ConcretePrototype::ConcretePrototype(const ConcretePrototype& cp)
{
	cout<<"ConcretePrototype copy..."<<endl;
}

Prototype* ConcretePrototype::Clone()const
{
	return new ConcretePrototype(*this);
}

```