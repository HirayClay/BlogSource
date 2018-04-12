---
title: 关于AndroidStudio下的idea目录
date: 2018-01-24 16:38:30
tags: 
    - AndroidStudio
---
这个目录一开始看名字只知道和我们工程的工作区间有关，到底是啥我没仔细看过。直到有一天，我不小心把这个文件的内容给删除了（可能是午睡趴在键盘上了），然后会导致工程一直报这个文件的错，于是乎我干脆把这个文件都删了。导致的结果是每次重新打开工程，之前打开过的文件全都不会自动打开，得一个个的点开，瞬间我就有点知道这个文件干嘛的了。其实就是记录我们最近的文件操作，比如你上次退出前打开过的文件，鼠标在哪个位置等等，比如：
![](https://github.com/HirayClay/draft/blob/master/as-workspace.png?raw=true)
我最后鼠标停在红色标记处，也就是整个文件的第1行，第25个文字处

那么在workspace.xml文件中会产生这么一条记录：
![](https://github.com/HirayClay/draft/blob/master/as-workspace-record-shot.png?raw=true)
应该是下标从0开始的原因，记录的line = 0,column =24
可能没有这条记录，ctrl+s强制保存一次就有了

当然workspace文件里面保存的还有其他信息，比如当然使用的gradle版本等信息

所以最后我自然就去其他工程拷贝了一份直接放到idea目录下就ok了
