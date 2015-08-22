---
Title: cocos2d-x 3.0 线程安全
date: 2015-01-07 11:08
status: public
tag: cocos2d-x
title: 'cocos2d-x 3.0 线程安全'
---

在cocos2dx中使用多线程，难免要考虑线程安全的问题。cocos2dx 3.0中新加入了一个专门处理线程安全的函数`performFunctionInCocosThread()`。他是Scheduler类的一个成员函数：
``` c++
void Scheduler::performFunctionInCocosThread(const std::function<void ()> &function)  
/** calls a function on the cocos2d thread. Useful when you need to call a cocos2d function from another thread.  
This function is thread safe.  
@since v3.0  
*/  
```
在其他线程中控制主线程去调用一个函数，这个函数会是线程安全的。
具体定义：
``` c++
void Scheduler::performFunctionInCocosThread(const std::function<void ()> &function)  
{  
    _performMutex.lock();  
    _functionsToPerform.push_back(function);  
    _performMutex.unlock();  
}  
```
使用这个函数就能安全的在其他线程中去控制cocos2dx的一些操作了。但是在使用时也要注意一些问题：
1.同步问题 
因为使用performFunctionInCocosThread将参数函数中的代码放到主线程中去运行，所以就无法知道运行完这段代码需要多少时间，可能线程已经运行完毕退出了而那部分代码还没有执行完毕。
可以做一下测试：
``` c++
void thread_fun()  
{  
    log("new thread create:t_id:0x%x",GetCurrentThreadId());  
  
    Director::getInstance()->getScheduler()->performFunctionInCocosThread([&,this]  
    {  
        for(int i=0;i<=1000;i++){}  
        log("[performFunctionInCocosThread] finished!");  
    });  
    
    log("thread finished:t_id:0x%x",GetCurrentThreadId());  
}  
```
然后在cocos2dx中以这个函数创建一个线程，运行可以看到结果：

![](~/20150107142713.png)

可以看出performFunctionInCocosThread中的代码执行完毕在线程退出之后，所以在使用时可能要加入一些值判断代码是否已经运行完毕。

2.互斥问题
如果在线程中要使用互斥，而又要使用performFunctionInCocosThread的话。我认为应该将performFunctionInCocosThread调用放到线程的最后，然后在performFunctionInCocosThread调用函数的末尾使用mutex.unlock(),这样才能确保互斥。

在自己的线程中对精灵的创建等操作可能会没有用，所以performFunctionInCocosThread还是很有用的。