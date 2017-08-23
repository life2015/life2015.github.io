---
layout:     post
title:      "Freeline迁移记录"
subtitle:   "修复Gradle Plugin崩溃"
date:       2017-08-23
author:     "Retrox"
header-img: "img/home-bg.jpg"
tags:
- Android
- Freeline
- Gradle
---

# Freeline 迁移记录-1 修复Gradle Plugin崩溃

> Freeline是我认为Android Application增量最快的 在成功增量的情况下，可以保证时间在10s及以内

### 痛点

- 不支持Android Studio3.0的gradle plugin
- databinding的支持是烂的


## 分析问题

> 项目已配置好freeline

```groovy
buildscript {
    repositories {
        jcenter()
//        mavenLocal()
        google()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.0-beta2'
        classpath 'com.antfortune.freeline:gradle:0.8.7'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

然后在Debug模式下运行Freeline

Boom在意料之中

追查log

```shell
> build failed with script: ./gradlew :app:assembleDebug -P freelineBuild=true --stacktrace

StackTrace: 
-> Caused by: java.lang.IllegalStateException: Resolving configuration 'annotationProcessor' directly is not allowed
-> 追查Freeline源码部分
-> 
  at com.antfortune.freeline.FreelinePlugin.findAptConfig(FreelinePlugin.groovy:644)
	at com.antfortune.freeline.FreelinePlugin$_apply_closure4$_closure13$_closure19.doCall(FreelinePlugin.groovy:176)
	at com.antfortune.freeline.FreelinePlugin$_apply_closure4$_closure13.doCall(FreelinePlugin.groovy:156)
	at com.antfortune.freeline.FreelinePlugin$_apply_closure4.doCall(FreelinePlugin.groovy:51)
```

**追查结果 ： 在findAptConfig里面出现了 不允许直接分析依赖的问题**

于是跑到Freeline源码里面

定位到这个方法里面的两行代码

```groovy
 def annotationProcessorConfig = project.configurations.findByName("annotationProcessor")
 def isAnnotationProcessor = annotationProcessorConfig != null && !annotationProcessorConfig.empty
// annotationProcessorConfig的直接访问导致抛出异常
// 因为最新的Gradle Plugin修改了Api
```

> 那怎么办呢？？？

于是 > 追查annotationProcessorConfig的类型及其源码

在源码分析到

```groovy
package org.gradle.api.artifacts;
public interface Configuration extends FileCollection, HasConfigurableAttributes<Configuration> 
```

查看这个接口相关的方法，顺藤摸瓜

```groovy
    @Incubating
    void setCanBeResolved(boolean var1);
```

顾名思义

因此我们在Freeline源码中添加一行

```groovy
def annotationProcessorConfig = project.configurations.findByName("annotationProcessor")
@+++
        annotationProcessorConfig.setCanBeResolved(true)
@+++
        def isAnnotationProcessor = annotationProcessorConfig != null && !annotationProcessorConfig.empty
```

**问题解决 ！**

于是我们开心的再次跑起了Freeline

Boom!

又来了

```groovy
Caused by: groovy.lang.MissingPropertyException: Could not get unknown property 'apkvariantData' for object of type com.android.build.gradle.internal.api.ApplicationVariantImpl.
```

查看栈回溯 Freeline相关代码

```groovy
                boolean multiDexEnabled = variant.apkvariantData.variantConfiguration.isMultiDexEnabled()

```

很显然...这个变量或者这个方法没了

从外层闭包开始看

```groovy
project.android.applicationVariants.each { variant -> ... }
```

看Foreach里面的varient类型

```groovy
public DomainObjectSet<ApplicationVariant> getApplicationVariants() {
        return this.applicationVariantList;
    }
```

然后去ApplicationVariant

```groovy
public interface ApplicationVariant extends ApkVariant, TestedVariant {
}
```

```groovy
public class ApplicationVariantImpl extends ApkVariantImpl implements ApplicationVariant {
    private final ApplicationVariantData variantData;
    private TestVariant testVariant = null;
    private UnitTestVariant unitTestVariant = null;

    public ApplicationVariantImpl(ApplicationVariantData variantData, ObjectFactory objectFactory, AndroidBuilder androidBuilder, ReadOnlyObjectProvider readOnlyObjectProvider, NamedDomainObjectContainer<BaseVariantOutput> outputs) {
        super(objectFactory, androidBuilder, readOnlyObjectProvider, outputs);
        this.variantData = variantData;
    }

  //看见没...名字改了
    public ApkVariantData getVariantData() {
        return this.variantData;
    }

   ......
}

```

于是改源码一行:

```groovy
                boolean multiDexEnabled = variant.variantData.variantConfiguration.isMultiDexEnabled()
//之前是apkvariantData
```

成功解决

当然....不可能这么容易跑通

下一个问题 在AndroidManifest的获取上

Gradle Android最新版Plugin不允许直接通过Api获取Manifest的地址（取消了那个api）

但是因为Android Manifest编译中在build目录的地址是相对固定的

原代码

```groovy
if (applicationProxy) {
                    variant.outputs.each { output ->
                        output.processManifest.outputs.upToDateWhen { false }
                        output.processManifest.doLast {
                            def manifestOutFile = output.processManifest.manifestOutputFile
                            if (manifestOutFile.exists()) {
                                println "find manifest file path: ${manifestOutFile.absolutePath}"
                                replaceApplication(manifestOutFile.absolutePath as String)
                            }
                        }
                    }
                }
```

修改后

```groovy
output.processManifest.doLast {
                            def path = "${project.buildDir}/intermediates/manifests/full/debug/AndroidManifest.xml"
                            def manifestFile = new File(path)
                            if (manifestFile.exists()){
                                println "find manifest file path: ${manifestFile.absolutePath}"
                                replaceApplication(manifestFile.absolutePath as String)
                            }
                        }
```



目前修复了Gradle Plugin的兼容性问题...

但是还没有结束




