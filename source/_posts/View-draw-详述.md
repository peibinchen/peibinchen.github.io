---
title: View.draw()详述
date: 2017-02-25 00:02:46
tags:
    - View的绘制原理
---

draw（）步骤如下：
1. Draw the background
2. If necessary, save the canvas' layers to prepare for fading
3. Draw view's content
4. Draw children
5. If necessary, draw the fading edges and restore layers
6. Draw decorations (scrollbars for instance)

<div style="text-align: center"> <img src="http://p1.bqimg.com/567571/5a09802a2a9a8598.png"/> </div>