---
title: View-measure-详述
date: 2017-02-25 00:10:04
tags:
    - View的绘制原理
---
measure()函数的主要流程如下：  
1、判断该View是否和它的ViewGroup是否同属于视觉边界布局或者同不属于视觉边界布局，如果不一致，则调整view的widthMeasureSpec和heightMeasureSpec。  
2、判断是否有必要进行重绘，如果没有必要重绘，则直接结束  
3、判断是否使用缓存，如果是的话，直接调用setMeasureDimensionRaw函数  
4、如果不使用缓存，则进行正常的onMeasure函数，在内部会再次和步骤1一样调整widthMeasureSpec和heightMeasureSpec，再调用setMeasureDimensionRaw函数设置最终的widthMeasureSpec和heightMeasureSpec  

<div style="text-align: center"> <img src="http://p1.bpimg.com/567571/f947c0cb8bae046e.png"/> </div>


**1、measure（int widthMeasureSpec，int heightMeasureSpec）这两个参数从哪里得来？**
如果是View，那么这两个参数是由ViewGroup根据自己的widthMeasureSpec，heightMeasureSpec和child的宽高给出的child建议值。
如果是DecorView，那么这两个参数是根据屏幕的宽高和自身的宽高一同决定的。

**2、PFLAG_FORCE_LAYOUT在哪里设置的？**  
在View中只有两个函数设置了PLAG_FORCE_LAYOUT：requestLayout()和forceLayout()  
requestLayout()和forceLayout()的区别是什么呢？  
（1）requestLayout函数不只是对自己设置PLAG_FORCE_LAYOUT，它还递归地设置了自己的Parent的PLAG_FORCE_LAYOUT，这就导致了从根节点到此view都会重新进行measure。  
（2）forceLayout函数就仅仅对自己设置PLAG_FORCE_LAYOUT，因此只对自己进行了measure。  

因此，如果在改变自己的view后发现如果对其他view有影响，应该调用requestLayout函数，而如果只对自己有影响，那么就调用forceLayout函数。