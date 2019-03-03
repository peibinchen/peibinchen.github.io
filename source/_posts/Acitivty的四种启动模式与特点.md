---
title: Acitivty的四种启动模式与特点
date: 2017-02-23 20:56:25
tags:
    - 四大组件
---

<div style="text-align: center"> <img src="http://i1.piimg.com/567571/8b3b6a7e8cf42e0c.png"/> </div>


**问题1：standard模式Android5.0之后如果被启动的Activity所在的进程已经启动，那么是加入已经存在的Task还是重新创建一个Task**  
答案是重新创建一个Task。

**问题2：singleTop模式是不是意味着只要栈顶有目标Activity就可以不用重新创建新的Activity了？**  
不是，singleTop模式不用创建新的Activity有两个条件：  
①目标Activity和调用者的Activity位于同一个Task中  
②目标Activity位于栈顶

**问题3：singleTask模式启动另一个进程的Activity，并且在另一个进程的Task中确实已经存在这个Activity的实例，那么按返回键时，能否立刻回到原来的进程Activity？**  
在singleTask的目标Activity已经启动后，如果目标Activity的Task中只有一个Activity，那么可以立刻回到原来的进程Activity；
但是如果目标Activity的Task中不只有目标Activity，那么就要不断按返回键直到Task中所有Activity全都销毁了才能回到原来的进程Activity。

参考：http://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/
