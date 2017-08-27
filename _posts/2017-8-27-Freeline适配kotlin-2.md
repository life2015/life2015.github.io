---
layout:     post
title:      "Freeline适配kotlin-2"
subtitle:   "kotlin增量实现"
date:       2017-08-27
author:     "Retrox"
header-img: "img/阿咏的图hhh.jpg"
tags:
- Android
- Freeline
typora-copy-images-to: ../img
---

# Freeline适配kotlin-2 源码修改

>  在上一部分我们梳理了java增量的逻辑 

整体来讲就是：

- 扫描变化的java文件
- 对它们进行单独的javac编译
- 然后打包成增量dex
- merge进去

> 其实现在大家基本上已经有思路了 
>
> javac换成kotlinc就ok了嘛

# 磨刀霍霍向牛羊

打开我心爱的pycharm 

## 1.增加对kotlin修改的扫描

```python
self._changed_files[module_name] = {'libs': [], 'assets': [], 'res': [], 'src': [], 'manifest': [],
                                    'config': [], 'so': [], 'cpp': [], 'kotlin': []}
# kotlin占坑
```

现在保存修改文件状态的map里面增加kotlin字段 准备放置修改的kotlin的文件路径

然后我们修改扫描代码文件修改的方法

```python
# scan src
src_dirs = self._config['project_source_sets'][module_name]['main_src_directory']
for src_dir in src_dirs:
    if os.path.exists(src_dir):
        for dirpath, dirnames, files in os.walk(src_dir):
            if re.findall(r'[/\\+]androidTest[/\\+]', dirpath) or '/.' in dirpath:
                continue
            for fn in files:
              # 之前只有对java后缀名文件的检查
                if fn.endswith('java'):
                    if fn.endswith('package-info.java') or fn.endswith('BuildConfig.java'):
                        continue
                    fpath = os.path.join(dirpath, fn)
                    if self.__check_changes(module_name, fpath, module_cache):
                        self._changed_files[module_name]['src'].append(fpath)
                # 添加kotlin的增量检查
                elif fn.endswith('kt'):
                    fpath = os.path.join(dirpath, fn)
                    if self.__check_changes(module_name, fpath, module_cache):
                        self._changed_files[module_name]['kotlin'].append(fpath)
```

>  `check_changes`这个方法我们在上一节就已经介绍了 先检查修改时间在检查md5

这些操作后 我们修改一个kotlin文件 然后跑一遍freeline的调试模式

```json
{
    "build_info": {
        "last_clean_build_time": 1503822329.0,
        "is_root_config_changed": false
    },
    "projects": {
        "app": {
            "src": [],
            "so": [],
            "assets": [],
            "kotlin": [
                "/Users/retrox/AndroidStudioProjects/One/app/src/main/java/com/twtstudio/one/view/VActivity.kt"
            ],
            "libs": [],
            "res": [],
            "config": [],
            "cpp": [],
            "manifest": []
        }
    }
}
```

在kotlin数组里面已经出现了我们修改的文件了 

我们下一步要做的就是：对它进行kotlinc

## 对kt文件单独增量编译

> 流程类似于对java的增量

新建一个TaskCommand类用于对kotlin的增量编译

```python
class GradleIncKotlincCommand(IncKotlincCommand):
    def __init__(self, module_name, invoker):
        IncKotlincCommand.__init__(self, module_name, invoker)

    def execute(self):
        self._invoker.check_r_md5()  # check if R.java has changed
        # self._invoker.check_other_modules_resources()
        should_run_kotlinc_task = self._invoker.check_kotlinc_task()
        if not should_run_kotlinc_task:
            self.debug('no need to execute kotlinc ')
            return

        self.debug('start to execute kotlinc command...')
        self._invoker.append_r_file()
        self._invoker.fill_classpaths()
        self._invoker.clean_dex_cache()
        self._invoker.run_kotlinc_task()
```

其实和java增量没有什么区别 因为没有加入kotlin的复杂特性支持（如kapt）所以某种角度看比java增量还简单

然后我们看看`run_kotlinc_task`这个方法

```python
#运行增量kotlinc
def run_kotlinc_task(self):
    kotlincargs = self._generate_kotlin_compile_args()
    self.debug('kotlinc exec: ' + ' '.join(kotlincargs))
    output, err, code = cexec(kotlincargs, callback=None)

    if code != 0:
        raise FreelineException('incremental kotlinc compile failed.', '{}\n{}'.format(output, err))
    
```

这个方法只贴了核心代码 当然还有一些R文件的增量什么的需要处理下（我只展示和kotlin相关的部分）

到了`_generate_kotlin_compile_args`这个方法

```python
#kotlin增量命令的生成 暂时直接kotlinc了
def _generate_kotlin_compile_args(self, extra_javac_args_enabled=False):
    # javacargs = [self._javac]
    # test environment kotlinc
    kotlincargs = ['kotlinc']
    arguments = []
    arguments.append('-cp')
    # todo  适配classpath
    self._classpaths.append('${projectDir}/app/build/tmp/kotlin-classes/debug')
    arguments.append(os.pathsep.join(self._classpaths))

    for fpath in self._changed_files['kotlin']:
        arguments.append(fpath)

    arguments.append('-d')
    arguments.append(self._finder.get_patch_classes_cache_dir())

    kotlincargs.extend(arguments)
    return kotlincargs
```

> kotlin的classpath要增加一些 因为kotlin的字节码文件是单独存放的 需要把那些字节码文件也纳入到classpath中来

然后把kotlin增量的操作添加到freeline执行的操作链上

```python
class GradleCompileCommand(CompileCommand):
    def __init__(self, module, invoker):
        self._module = module
        CompileCommand.__init__(self, 'gradle_{}_compile_command'.format(module), invoker)

    def _setup(self):
        # 加一个kotlin即可
        self.add_command(GradleIncKotlincCommand(self._module, self._invoker))
        self.add_command(GradleIncJavacCommand(self._module, self._invoker))
        self.add_command(GradleIncDexCommand(self._module, self._invoker))
```

其实就已经很清晰了 增量kotlinc然后把字节码存放到freeline文件夹 然后再统一dex打增量包 推送到手机

## Dex增量打包

Dex增量工具会自己把字节码文件夹的生成字节码打包 而我们适配kotlin要做的是

设置标志位 为什么呢？

> 因为之前增量dex的标志位只针对了java的情况 修改kotlin并不能触发标志位的变化
>
> 所以只改kotlin的话 dex是拒接增量打包推送的

所以我们要小小的修改下

```python
def _mark_changed_flag(self):
    info = self._changed_files.values()
    cache_dir = self._config['build_cache_dir']
    for bundle in info:
        if not android_tools.is_src_changed(cache_dir) and len(bundle['src']) > 0:
            android_tools.mark_src_changed(cache_dir)
        if not android_tools.is_res_changed(cache_dir) and len(bundle['res']) > 0:
            android_tools.mark_res_changed(cache_dir)
        # kotlin增量flag
        if not android_tools.is_src_changed(cache_dir) and len(bundle['kotlin']) > 0:
            android_tools.mark_src_changed(cache_dir)
```



然后修改下kotlin 跑下freeline  **成功增量！**

> freeline兼容适配之路还有很长要走
>
> 不如kapt kotlin和java混写的相互依赖什么的 还有各种蜜汁问题
>
> 不过一个优秀的增量工具就是这样子一步步走下来的
>
> 希望这篇文章可以对你有所启发