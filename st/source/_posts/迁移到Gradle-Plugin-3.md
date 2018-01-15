---
title: 迁移到Gradle3.0
date: 2018-01-11 11:04:03
tags:
    - AndroidStudio 
    - Gradle3.x 
    - Gradle
---
最近Gradle plugin更新到3.0之后出现了不少问题，不过官方也给出了迁移到3.0之后这些问题
的[解决办法](https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html?utm_source=android-studio#update_gradle)。
无法翻墙的可以访问[国内站点](https://developer.android.google.cn/studio/build/gradle-plugin-3-0-0-migration.html?utm_source=android-studio#update_gradle)

更新到3.0之后，对应的gradle也必须更新到4.1，简单修改一下gradle.properties文件里面的
gradle地址为：https\://services.gradle.org/distributions/gradle-4.1-all.zip就可以了
值得注意的是 gradle plugin的更新现在在google仓库，所以得把google仓库加上
```gradle
    repositories {
        ...
        // You need to add the following repository to download the
        // new plugin.
        google()
    }
```

### 变体区分依赖管理
翻译的不太合适，大意就是现在3.0以及之后都是采用新的依赖管理机制，比如module A依赖了 module B，那么在构建A 的debug变体时也会自动依赖B的debug版本，不像以前得这么写一句<br>
_debugCompile project(path: ':data', configuration: 'debug')_

1. 定义风味
  如果要构建多种flavor(实在不想翻译成风味，意会就行)，那么必须给每种flavor指定flavorDimension

2.构建错误
如果你给app 增减了一个叫做staging的 buildType,但是其依赖并没有叫做staging的buildType，那么就会报错：
```groovy
    Error:Failed to resolve: Could not resolve project :mylibrary.
Required by:
    project :app
```
那么可以使用matchingFallbacks来指定合适的替代依赖,依赖debug,qa或者release，会依赖第一个找到的依赖
```groovy
 buildTypes {
        debug {}
        release {}
        staging {
            // Specifies a sorted list of fallback build types that the
            // plugin should try to use when a dependency does not include a
            // "staging" build type. You may specify as many fallbacks as you
            // like, and the plugin selects the first build type that's
            // available in the dependency.
            matchingFallbacks = ['debug', 'qa', 'release']
        }
    }
```
与之类似的是，如果flavorDimension缺失，也可以指定替代的flavorDimension,用法如下
```groovy
    android {
    defaultConfig{
        missingDimensionStrategy 'minApi', 'minApi18', 'minApi23'
        missingDimensionStrategy 'abi', 'x86', 'arm64'
    }
    flavorDimensions 'tier'
    productFlavors {
        free {
            dimension 'tier'
            missingDimensionStrategy 'minApi', 'minApi23', 'minApi18'
        }
        paid {}
    }
}
```


### 为本地Module改变依赖配置

由于区分变体依赖的原因，现在不需要像以前那样配置具体的依赖变体了，像debugImplementation debugCompile之类的用法都可以废弃了，如果遇到一下错误那都是这个原因

```groovy
    Error:Unable to resolve dependency for ':app@debug/compileClasspath':
        Could not resolve project :library.
    Error:Unable to resolve dependency for ':app@release/compileClasspath':
        Could not resolve project :library.
```
应该改为以下配置：

```groovy
    dependencies {
    // 该用法对本地module不再生效
    // debugImplementation project(path: ':library', configuration: 'debug')
    // releaseImplementation project(path: ':library', configuration: 'release')

    // 简单的一句implementation就可以启用"变体区分"来自动需找对应的本地依赖变体
    implementation project(':library')

    //不过对于外部依赖仍然可以使用这种方式
    debugImplementation 'com.example.android:app-magic:12.3'
}
```

注意的是，尽管这些手动指定依赖的API仍然还可以用，但是最好不要用。因为使用project() dsl语法提供的依赖 必须和 使用依赖者 在buildType和flavor 以及其他属性上匹配。举个栗子，比如让一个debug变体去使用一个release变体是不可能