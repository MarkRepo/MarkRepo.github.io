---
title: 单例模式(SingleTon)
description: 设计模式之单例模式 
categories: 设计模式
tag: 
---

>SingleTon不可以被实例化，因此将其构函数声明为protected或者private

```c++
//singleton.h

#ifndef _SINGLETON_H_
#define _SINGLETON_H_

#include <iostream>
using namespace std;

class Singleton
{

	public:
		static Singleton* Instance();

	protected:
		Singleton();

	private:
		static Singleton* _instance;

}

#endif
```

```c++
//singleton.cpp

#include "Singleton.h"

Singleton* singleton::_instance = 0;

Singleton::Singleton()
{
	cout<<"Singleton...."<<endl;
}

Singleton* single::Instance()
{
	if(_instance == 0)
		_instance = new Singleton();

	return _instance;
}

```