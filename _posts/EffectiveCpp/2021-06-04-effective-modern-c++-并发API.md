---
title: effective modern c++ 之 并发API
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

条款35 优先选用基于任务而非基于线程的程序设计

条款36 如果异步是必要的，则指定std::launch::async

条款37 使std::thread型别对象在所有路径皆不可联结

条款38 对变化多端的线程句柄析构函数行为保持关注

条款39 考虑针对一次性事件通信使用以void为模板型别实参的期值

条款40 对并发使用std::atomic, 对特种内存使用volatile