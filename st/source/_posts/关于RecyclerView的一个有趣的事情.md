---
title: 关于RecyclerView的一个有趣的事情
date: 2018-01-19 15:22:22
tags: - RecyclerView 
      - Android
---
对于RecylcerView ，基本上第一印象就是View重用，但是真的明白怎么重用的吗，最近在写自定义LayoutManager,由此对RecylerView、LayoutManager、ItemAnimator整个之间的关系都比较的熟悉。不过回到标题上来，这个有趣的事情和RV的回收有关。

比如页面上此时刚好显示了前六条Item，那么你肯定觉得第七条还没创建，关于第七条的信息一点也没有，而且会根据对RV重用的印象觉得只有当我把第一条Item稍微滑出去一点就会