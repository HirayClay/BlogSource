---
title: Gradle的一些小知识(不定期更新)
date: 2018-01-05 13:20:07
tags:
    - Gradle
    - Android
---

### 资源分包
一次不小心点进去sourceSet进去，发现可以定义资源路径的;自己按照main目录下的结构一样创建了一个debug的SourceSet(新建的其他名字的都不行，debug和main可以)，然后在module的gradle中加入如下配置就等于是有两个res目录，这样可以让资源分类更清晰
```groovy
        sourceSets {
            debug {
                res.srcDirs("src/debug/res_debug", "src/debug/res")
            }

        }
```
记得sync一下
![](https://github.com/HirayClay/draft/blob/master/GradleSourceSet.png?raw=true)
最后生成apk的时候两个sourceSet的东西会合并