---
layout: post
title:  "ATT&CK-DLL Side-Loading"
date:   2019-12-09 20:17:40 +0800
categories: jekyll update
---
# 前言

本篇文章主要是分析DLL Side-Loading（DLL侧加载技术），在说到DLL Side-loading技术前，先了解下`动态静态链接库（DLL）`、`DLL加载的方式`以及`DLL加载顺序原理`。

# 动态链接库（DLL）

> 官方解释：https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-libraries
>

百度百科：

DLL(Dynamic Link Library)文件为动态链接库文件，又称“应用程序拓展”，是软件文件类型。在Windows中，许多应用程序并不是一个完整的可执行文件，它们被分割成一些相对独立的动态链接库，即DLL文件，放置于系统中。当我们执行某一个程序时，相应的DLL文件就会被调用。一个应用程序可使用多个DLL文件，一个DLL文件也可能被不同的应用程序使用，这样的DLL文件被称为共享DLL文件。

DLL 是一个包含可由多个程序同时使用的代码和数据的库。例如，在 Windows 操作系统中，Comdlg32 DLL 执行与对话框有关的常见函数。因此，每个程序都可以使用该 DLL 中包含的功能来实现“打开”对话框。这有助于促进代码重用和内存的有效使用。

**在这里不免要啰嗦下动态链接库和静态链接库的区别：（引用简书-作者：烟雨随风）**

在软件开发的过程中，大家经常会或多或少的使用别人编写的或者系统提供的动态库或静态库，但是究竟是使用静态库还是动态库呢？他们的适用条件是什么呢？

简单的说，静态库和应用程序编译在一起，在任何情况下都能运行，而动态库是动态链接，顾名思义就是在应用程序启动的时候才会链接，所以，当用户的系统上没有该动态库时，应用程序就会运行失败。再看它们的特点：

**动态库：**

- 类库的名字一般是 libxxx.so
- 共享：多个应用程序可以使用同一个动态库，启动多个应用程序的时候，只需要将动态库加载到内存一次即可；
- 开发模块好：要求设计者对功能划分的比较好。
- 动态函数库的改变并不影响你的程序，所以动态函数库的升级比较方便。

**静态库：**

- 类库的名字一般是libxxx.a
- 代码的装载速度快，执行速度也比较快，因为编译时它只会把你需要的那部分链接进去。
- 应用程序相对比较大，如果多个应用程序使用的话，会被装载多次，浪费内存。
- 如果静态函数库改变了，那么你的程序必须重新编译。

如果你的系统上有多个应用程序都使用该库的话，就把它编译成动态库，这样虽然刚启动的时候加载比较慢，但是多任务的时候会比较节省内存；如果你的系统上只有一到两个应用使用该库，并且使用的API比较少的话，就编译成静态库吧，一般的静态库还可以进行裁剪编译，这样应用程序可能会比较大，但是启动的速度会大大提高。

简单来说，动态链接库是主程序（EXE）运行时根据导入表的内容去调用对应的DLL文件，其特点是运行时调用，动态链接库的方式符合开发的“高内聚，低耦合”的思路，同时便于进行版本控制。静态链接库是编译文件时已经将对应的静态库编译进最终程序，应用程序相对比较大，如果需要更新版本得重新编译替换主程序。

# 动态链接库的加载顺序

> 官方链接：https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order
> 本次讨论的动态链接库加载顺序不包含Windows应用商店应用，只限于常用的桌面应用程序

**DLL查找路径的基础：**
应用程序可以通过以下方式控制一个DLL的加载路径：使用全路径加载、使用DLL重定向、使用manifest文件。如果上述三种方式均未指定，系统查找DLL的顺序将按照本部分描述的顺序进行。

对于以下两种情况的DLL，系统将不会查找，而是直接加载：

a. 对于已经加载到内存中的同名DLL，系统使用已经加载的DLL，并且忽略待加载DLL的路径。（注意对某个进程而言，系统已经加载的DLL一定是唯一的存在于某个目录下。）

b. 如果该DLL存在于某个Windows版本的已知DLL列表（unkown DLL）中，系统使用已知DLL的拷贝（包括已知DLL的依赖项）。已知DLL列表可以从如下注册表项看到：HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs。

这里有个比较坑的地方，对于有依赖项的DLL（即使使用全路径指定DLL位置），系统查找其所依赖DLL的方法是按照实际的模块名称来的，因此如果加载的DLL不在系统查找顺序目录下，那么动态加载该DLL（LoadLibrary）会返回一个"找不到模块"的错误。


看了一圈资料发现从Windows不同版来了解DLL的加载顺序比较顺畅
**2. 系统标准DLL查找顺序**
> https://www.cnblogs.com/tocy/p/windows_dll_searth_path.html
> http://sh1yan.top/2019/06/16/The-Principle-and-Practice-of-DLL-Hijacking/

##  Windows XP SP2之前

Windows查找DLL的目录以及对应的顺序（Windows XP下，"安全DLL查找模式"默认是禁用的，需要启用该项的话，在注册表中新建一个SafeDllSearchMode子项，并赋值为1即可。"安全DLL查找模式"从Windows XP SP2开始，默认是启用的）：

1. 进程对应的应用程序所在目录；（也就是运行的EXE所在的目录）
2. 当前目录（Current Directory）（用win+R打开CMD看到的初始目录就是当前目录）；> https://stackoverflow.com/questions/57901463/current-directory-vs-directory-from-which-the-application-loaded
3. 系统目录（通过 GetSystemDirectory 获取）（％SystemRoot％\ system32）；
4. 16位系统目录；（现在基本消失了，可以忽略）
5. Windows目录（通过 GetWindowsDirectory 获取）（通常是系统盘\Windows）；
6. PATH环境变量中的各个目录；

##  Windows xp sp2（包含）以上
Windows查找DLL的目录以及对应的顺序（SafeDllSearchMode 默认会被开启）：
默认注册表为：HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\SafeDllSearchMode，其键值为1

1. 进程对应的应用程序所在目录；（也就是运行的EXE所在的目录）
2. 系统目录（通过 GetSystemDirectory 获取）（％SystemRoot％\ system32）；
3. 16位系统目录；（现在基本消失了，可以忽略）
4. Windows目录（通过 GetWindowsDirectory 获取）（通常是系统盘\Windows）；
5. 当前目录（Current Directory）（用win+R打开CMD看到的初始目录就是当前目录)；> https://stackoverflow.com/questions/57901463/current-directory-vs-directory-from-which-the-application-loaded
6. PATH环境变量中的各个目录；

## Windows 7 以上
系统没有了SafeDllSearchMode 而采用KnownDLLs，那么凡是此项下的DLL文件就会被禁止从EXE自身所在的目录下调用，而只能从系统目录即SYSTEM32目录下调用，其注册表位置：
计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs

![微信截图_20191210171651](C:\Users\liwei\Dropbox\Github\Boomboomdunce.github.io\image\微信截图_20191210171651.png)
那么最终Windows2003以上以及win7以上操作系统通过“DLL路径搜索目录顺序”和“KnownDLLs注册表项”的机制来确定应用程序所要调用的DLL的路径，之后，应用程序就将DLL载入了自己的内存空间，执行相应的函数功能。
1. 进程对应的应用程序所在目录；（也就是运行的EXE所在的目录）
2. 系统目录（通过 GetSystemDirectory 获取）（％SystemRoot％\ system32）；
3. 16位系统目录；（现在基本消失了，可以忽略）（即%windir%system，即%windir%）
4. Windows目录（通过 GetWindowsDirectory 获取）（通常是系统盘\Windows）；
5. 当前目录（Current Directory）（用win+R打开CMD看到的初始目录就是当前目录)；> https://stackoverflow.com/questions/57901463/current-directory-vs-directory-from-which-the-application-loaded
6. PATH环境变量中的各个目录；

# 动态链接库的加载方式

# 动态链接库黑客技术

# 动态链接库的安全解决方法