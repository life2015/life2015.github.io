---
layout:     post
title:      "Freeline迁移记录2"
subtitle:   "修复classpath崩溃"
date:       2017-08-24
author:     "Retrox"
header-img: "img/home-bg.jpg"
tags:
- Android
- Freeline
- Gradle
---

# 新版迁移过程中修复Classpath问题

> Freeline踩坑日记

## 追查表象的蛛丝马迹

> 结果前一部分的修改
>
> 基本上做到了编译的通过 
>
> 然后在增量的时候 报出了错误

类似这样子的错误

```groovy
/Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java:3: error: package android.annotation does not exist
import android.annotation.SuppressLint;
                         ^
/Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java:4: error: package android.content does not exist
import android.content.Context;
                      ^
/Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java:5: error: package android.content does not exist
import android.content.Intent;
                      ^
/Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java:6: error: package android.os does not exist
import android.os.Build;
                 ^
/Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java:7: error: package android.support.annotation does not exist
import android.support.annotation.RequiresApi;
                                 ^
...
```

很显然是classpath的缺失

于是查阅Freeline的log ， 也分析了Freeline的增量构建流程

这里几段Python代码是运行了增量javac

```python
javacargs = self._generate_java_compile_args(extra_javac_args_enabled=extra_javac_args_enabled)

        self.debug('javac exec: ' + ' '.join(javacargs))
        output, err, code = cexec(javacargs, callback=None)
```

于是追查javac的命令 查看它的传入的classpath

```shell
 /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/bin/javac 
 -encoding UTF-8 -g -target 1.7 -source 1.7 
 -cp 
 /Users/jichenyang/AndroidStudioProjects/ONE/app/build/freeline/app/classes:
 /Users/jichenyang/AndroidStudioProjects/ONE/app/build/intermediates/classes/debug:
 /Users/jichenyang/android-sdk-mac_x86/platforms/android-25/android.jar 
 /Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java 
 /Users/jichenyang/AndroidStudioProjects/ONE/app/build/freeline/app/backup/com/liuzh/one/R.java -d /Users/jichenyang/AndroidStudioProjects/ONE/app/build/freeline/app/classes
```

通过观察这几个目录可以发现的是 这些classpath并不能满足需要 

缺少了一些第三方库的依赖 

根据：“控制变量法” 我切换到了稳定版的Gradle Plugin跑了一遍 查看它的log

> PS:大家看长度就可以~

```shell
javac exec: /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/bin/javac 
-encoding UTF-8 -g -target 1.7 -source 1.7 -cp 
/Users/jichenyang/AndroidStudioProjects/ONE/app/build/freeline/app/classes:
/Users/jichenyang/AndroidStudioProjects/ONE/app/build/intermediates/classes/debug:
/Users/jichenyang/android-sdk-mac_x86/platforms/android-25/android.jar:
/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.google.code.gson/gson/2.6.1/b9d63507329a7178e026fc334f87587ee5070ac5/gson-2.6.1.jar:
/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.retrofit2/retrofit/2.2.0/41e67dba73c3347e4503761642c39d0e06ca1f2/retrofit-2.2.0.jar:
/Users/jichenyang/.android/build-cache/edb8a7ac2177cd2946b58128ce36f08bc9b38f89/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/1533df315d42d7f4ff00b9f8b7de5c201256284e/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/6aefc3619c39e6e4ce3118e695f213d148de053e/output/jars/classes.jar:/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.picasso/picasso/2.5.2/7446d06ec8d4f7ffcc53f1da37c95f200dcb9387/picasso-2.5.2.jar:/Users/jichenyang/.android/build-cache/dffbaa523b9008a1896eec5b373120bb1c6de17c/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/4aaff6c08351f15c642176ed10e6377844d7046c/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/b20f53c3c1cdc7f0f1b4d0dbb35408b94ce38903/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/ba30abeac3972b144fefa6124554f582fa60df5c/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/d4cdfb05df67f451bb9aba52161bc413bc43c9ea/output/jars/classes.jar:/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.okio/okio/1.11.0/840897fcd7223a8143f1d9b6f69714e7be34fd50/okio-1.11.0.jar:/Users/jichenyang/.android/build-cache/3936d4bae4a86548d80c6de52bbd92ee7a284855/output/jars/classes.jar:/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.okhttp3/okhttp/3.6.0/69edde9fc4b01c9fd51d25b83428837478c27254/okhttp-3.6.0.jar:/Users/jichenyang/.android/build-cache/f159250e4dfb4e3d4150d31319640f34f7ff3388/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/64331bf8b484ce0281624b5b84622ed7a08fbba2/output/jars/classes.jar:/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.retrofit2/converter-gson/2.0.2/f8d87f15b94b8d74e7ccf61d7eedb558811cdb30/converter-gson-2.0.2.jar:/Users/jichenyang/.android/build-cache/11cf9470b93863a1118d758231db813ac8dbc99c/output/jars/classes.jar:/Users/jichenyang/.android/build-cache/28277db1bdc4921dded9cbd9bc2991b9198a9093/output/jars/classes.jar:/Users/jichenyang/android-sdk-mac_x86/extras/android/m2repository/com/android/support/support-annotations/25.3.1/support-annotations-25.3.1.jar:/Users/jichenyang/.android/build-cache/837dc24219f090432e4c44c4b5da23f93591bb19/output/jars/classes.jar: /Users/jichenyang/AndroidStudioProjects/ONE/app/src/main/java/com/liuzh/one/activity/MovieActivity.java -d /Users/jichenyang/AndroidStudioProjects/ONE/app/build/freeline/app/classes
```

相比之下大概是什么呢：

> 正常情况下 应该还囊括到Gradle编译过去中释放aar产生的jar包依赖 加入classpath
>
> 然而在AS3.0中 这部分依赖并没有被加入进来

Freeline的源码中有一段注释可以很清楚的表面Classpath的来源

```python
    def fill_classpaths(self):
        # classpaths:
        # 1. patch classes
        # 2. dependent modules' patch classes
        # 3. android.jar
        # 4. third party jars
        # 5. generated classes in build directory
        ...
```

### 发现表象查源头

> 问：为什么这部分Classpath丢失了

查看Freeline增量编译Javac部分Classpath添加的方法

```python
    def fill_classpaths(self):
        # classpaths:
        # 1. patch classes
        # 2. dependent modules' patch classes
        # 3. android.jar
        # 4. third party jars
        # 5. generated classes in build directory
        patch_classes_cache_dir = self._finder.get_patch_classes_cache_dir()
        self._classpaths.append(patch_classes_cache_dir)
        self._classpaths.append(self._finder.get_dst_classes_dir())
        for module in self._module_info['local_module_dep']:
            finder = GradleDirectoryFinder(module, self._module_dir_map[module], self._cache_dir)
            self._classpaths.append(finder.get_patch_classes_cache_dir())

        # add main module classes dir to classpath to generate databinding files
        main_module_name = self._config['main_project_name']
        if self._name != main_module_name and self._is_databinding_enabled:
            finder = GradleDirectoryFinder(main_module_name, self._module_dir_map[main_module_name], self._cache_dir,
                                           config=self._config)
            self._classpaths.append(finder.get_dst_classes_dir())

        self._classpaths.append(os.path.join(self._config['compile_sdk_directory'], 'android.jar'))
        #重点来了 (下面)
        self._classpaths.extend(self._module_info['dep_jar_path'])

        # remove existing same-name class in build directory
        srcdirs = self._config['project_source_sets'][self._name]['main_src_directory']
...
```

注释写的很清晰 顾名思义

> 可以用Debug的方法 一句一句的跑 然后在Debugger里面看Classpath的变量值

`   self._classpaths.extend(self._module_info['dep_jar_path'])`这句话就是添加第三方依赖的操作

然后这条线就到了下一个工序  —> `self._module_info['dep_jar_path']`是哪里来的？

通过调试+全局搜索的骚操作

定位在了

```python
def get_project_info(config):
    Logger.debug("collecting project info, please wait a while...")
    project_info = {}
    if 'modules' in config:
        modules = config['modules']
    else:
        modules = get_all_modules(os.getcwd())

#从这个json文件中取出来了依赖路径
    jar_dependencies_path = os.path.join(config['build_cache_dir'], 'jar_dependencies.json')
    jar_dependencies = []
    if os.path.exists(jar_dependencies_path):
        jar_dependencies = load_json_cache(jar_dependencies_path)

    for module in modules:
        if module['name'] in config['project_source_sets']:
            module_info = {}
            module_info['name'] = module['name']
            module_info['path'] = module['path']
            module_info['relative_dir'] = module['path']
            #把他们放进去
            module_info['dep_jar_path'] = jar_dependencies
            module_info['packagename'] = get_package_name(
                config['project_source_sets'][module['name']]['main_manifest_path'])

```

看上面代码的注释部分 就可以知道 这些依赖是从一个json里面取出来的

于是我们去翻这个json文件

```json
[
    "/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.google.code.gson/gson/2.6.1/b9d63507329a7178e026fc334f87587ee5070ac5/gson-2.6.1.jar",
    "/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.retrofit2/retrofit/2.2.0/41e67dba73c3347e4503761642c39d0e06ca1f2/retrofit-2.2.0.jar",
    "/Users/jichenyang/.android/build-cache/edb8a7ac2177cd2946b58128ce36f08bc9b38f89/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/1533df315d42d7f4ff00b9f8b7de5c201256284e/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/6aefc3619c39e6e4ce3118e695f213d148de053e/output/jars/classes.jar",
    "/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.picasso/picasso/2.5.2/7446d06ec8d4f7ffcc53f1da37c95f200dcb9387/picasso-2.5.2.jar",
    "/Users/jichenyang/.android/build-cache/dffbaa523b9008a1896eec5b373120bb1c6de17c/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/4aaff6c08351f15c642176ed10e6377844d7046c/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/b20f53c3c1cdc7f0f1b4d0dbb35408b94ce38903/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/ba30abeac3972b144fefa6124554f582fa60df5c/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/d4cdfb05df67f451bb9aba52161bc413bc43c9ea/output/jars/classes.jar",
    "/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.okio/okio/1.11.0/840897fcd7223a8143f1d9b6f69714e7be34fd50/okio-1.11.0.jar",
    "/Users/jichenyang/.android/build-cache/3936d4bae4a86548d80c6de52bbd92ee7a284855/output/jars/classes.jar",
    "/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.okhttp3/okhttp/3.6.0/69edde9fc4b01c9fd51d25b83428837478c27254/okhttp-3.6.0.jar",
    "/Users/jichenyang/.android/build-cache/f159250e4dfb4e3d4150d31319640f34f7ff3388/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/64331bf8b484ce0281624b5b84622ed7a08fbba2/output/jars/classes.jar",
    "/Users/jichenyang/.gradle/caches/modules-2/files-2.1/com.squareup.retrofit2/converter-gson/2.0.2/f8d87f15b94b8d74e7ccf61d7eedb558811cdb30/converter-gson-2.0.2.jar",
    "/Users/jichenyang/.android/build-cache/11cf9470b93863a1118d758231db813ac8dbc99c/output/jars/classes.jar",
    "/Users/jichenyang/.android/build-cache/28277db1bdc4921dded9cbd9bc2991b9198a9093/output/jars/classes.jar",
    "/Users/jichenyang/android-sdk-mac_x86/extras/android/m2repository/com/android/support/support-annotations/25.3.1/support-annotations-25.3.1.jar",
    "/Users/jichenyang/.android/build-cache/837dc24219f090432e4c44c4b5da23f93591bb19/output/jars/classes.jar",
    ""
]
```

很显然了已经 （对比之下 3.0的环境下 这个json的数组是空的）

> 测试的时候一个比较hack的操作是 我把2.3.3环境下跑出的json文件内容 复制到3.0环境下
>
> 然后增量javac成功
>
> 这也就证明了javac环节的问题 同时消去了我的一个担忧：是不是Dexmerge的时候会烂

### 穷追不舍 修复问题

> 那么
>
> 为什么这个json文件会是空的？
>
> 为什么2.3的时候这个json文件可以被赋值？
>
> 这个json文件的数据是在什么时候被存入的？

带着问题 我们继续追查源码

> 一招骚操作 我在Freeline源码里面全局搜索
>
> Command + shift + F
>
> 就搜它： "jar_dependencies.json"

果然 它还在Freeline的Gradle Plugin里面出现了

> Freeline的取出jar的逻辑是：
>
> 把处理jar的Task取出来
>
> 然后遍历他的inputs，取出那些jar包的地址 放在json里面

```groovy
String manifest_path = project.android.sourceSets.main.manifest.srcFile.path
                    if (getMinSdkVersion(variant.mergedFlavor, manifest_path) < 21 && multiDexEnabled) {
                        classesProcessTask = project.tasks.findByName("transformClassesWithJarMergingFor${variant.name.capitalize()}")
                        multiDexListTask = project.tasks.findByName("transformClassesWithMultidexlistFor${variant.name.capitalize()}")
                    } else {
                        classesProcessTask = project.tasks.findByName("transformClassesWithDexFor${variant.name.capitalize()}")
                    }
```

```groovy
project.task(hackClassesBeforeDex) << {
                    def jarDependencies = []
  //就是这里偷出jar依赖地址
                    classesProcessTask.inputs.files.files.each { f ->
                        if (f.isDirectory()) {
                            f.eachFileRecurse(FileType.FILES) { file ->
                                backUpClass(backupMap, file as File, backUpDirPath as String, modules.values())
                                FreelineInjector.inject(excludeHackClasses, file as File, modules.values())
                                if (file.path.endsWith(".jar")) {
                                    jarDependencies.add(file.path)
                                }
                            }
                        } else {
                            backUpClass(backupMap, f as File, backUpDirPath as String, modules.values())
                            FreelineInjector.inject(excludeHackClasses, f as File, modules.values())
                            if (f.path.endsWith(".jar")) {
                                jarDependencies.add(f.path)
                            }
                        }
                    }

                    if (preDexTask == null) {
                        def providedConf = project.configurations.findByName("provided")
//                        providedConf.setCanBeResolved(true) //适配3.0 但是这里不行
                        if (providedConf) {
                            def providedJars = providedConf.asPath.split(File.pathSeparator)
                            jarDependencies.addAll(providedJars)
                        }

                        jarDependencies.addAll(addtionalJars)
                        // add all additional jars to final jar dependencies
                        def json = new JsonBuilder(jarDependencies).toPrettyString()
                        project.logger.info(json)
                        FreelineUtils.saveJson(json, FreelineUtils.joinPath(FreelineUtils.getBuildCacheDir(project.buildDir.absolutePath), "jar_dependencies.json"), true);
                    }
                }
```

这是它之前的代码

> 那为什么无法兼容新版3.0呢？

因为`"transformClassesWithJarMergingFor${variant.name.capitalize()}"`这个task在3.0上...

不存在了...

不存在了...

![](http://i1.muzisoft.com/uploads/image/20170508/20170508154655_16220.jpg)

显然Android studio改了编译流程的task

不过它也不能跳过这一步 肯定有替代品 ...

![](http://ws3.sinaimg.cn/large/9150e4e5ly1fh5rohl95kg208c08cmx9.gif)

果然

```groovy
classesProcessTask = project.tasks.findByName("transformClassesWithDexBuilderFor${variant.name.capitalize()}")
```

在Plugin中增加对3.0的判断 然后替换成这个task就可以了

>很幸运的是...这个task的相关从inputs里面取jar相关的api都没有变...
>
>没有给我太大的折磨

改完相关代码 install到MavenLocal

跑一下增量 OK！

> 如果有问题的话....那估计是3.0自带的问题
>
> 坑还是要慢慢踩