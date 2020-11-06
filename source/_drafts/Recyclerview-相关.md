---
title: Recyclerview 相关
tags:
---

https://www.jianshu.com/u/00b25c511dd3


[Android中当item数量超过一定大小RecyclerView高度固定
](https://blog.csdn.net/qqq2830/article/details/79868552)

[RecyclerView实现设置最大高度maxHeight](https://blog.csdn.net/z226688/article/details/83722638)--自己目前采用了此方案


[RecyclerView 系列](https://blog.csdn.net/zxt0601/article/category/9267588)

[RecyclerView 系列](https://blog.csdn.net/zxt0601/article/category/6398860)



----

1. rv.setAdapter

    setAdapterInternal

RecyclerViewDataObserver[]







Recycler#tryBindViewHolderByDeadline(@NonNull ViewHolder holder, int offsetPosition,int position, long deadlineNs)
Adapter#bindViewHolder(@NonNull VH holder, int position)
    Adapter#onBindViewHolder
        abstract onBindViewHolder