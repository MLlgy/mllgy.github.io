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

![](/../images/2019_08_09_01.png)

