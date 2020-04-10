---
title: RecyclerView之ItemDecoration
tags:
---


### Recyclerview#ItemDecoration 的方法

**getItemOffsets**
```
void getItemOffsets (Rect outRect, 
                View view, 
                RecyclerView parent, 
                RecyclerView.State state)
```
> Retrieve any offsets for the given item. Each field of outRect specifies the number of pixels that the item view should be inset by, similar to padding or margin. The default implementation sets the bounds of outRect to 0 and returns.

通过设置 outRect 的 left、top、right、bottom 为 Item 所在的 View 设置边界偏移量，设置效果与 padding、margin 相似，默认值为 0。可以调用 `getChildAdapterPosition(View)` 获得 Item 的 position，效果示意图如下：

![](/../images/2019_08_09_02.png)


**onDrawOver**
```
void onDrawOver (Canvas c, 
                RecyclerView parent, 
                RecyclerView.State state)
```              
> Draw any appropriate decorations into the Canvas supplied to the RecyclerView. Any content drawn by this method will be drawn after the item views are drawn and will thus appear over the views.

在 Item 所在的 View 绘制后进行绘制，所以出现在 View 的上面，示意图如下：


![](/public/images/2019_08_09_01.png)

**onDraw**
```
void onDraw (Canvas c, 
                RecyclerView parent, 
                RecyclerView.State state)
```

> Draw any appropriate decorations into the Canvas supplied to the RecyclerView. Any content drawn by this method will be drawn before the item views are drawn, and will thus appear underneath the views.

对 Canvas 的任何绘制动作都会应用到 Recyclerview 上，但是由于此方法在 Item 的 View 的之前绘制，所以在 Item 的 View 的下方，示意图如下：

![](/public/images/2019_08_13_01.png)



----
**知识链接**

[RecyclerView之ItemDecoration由浅入深](https://www.jianshu.com/p/b46a4ff7c10a)