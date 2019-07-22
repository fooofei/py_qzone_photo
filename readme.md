﻿
# 使用约束

自己的 QQ 与目标 QQ 是好友。

能够允许抓取的相册是对方允许访问的。

以上理解为，通过手工通过浏览器访问的照片，都是通过程序自动化完成手工的这些步骤，

而其他的手工无权限完成的事情，程序也无法完成。


# Go 版本的使用方法


### 20190714 更新

方案依旧可用

小幅修改说明，整理代码目录结构。暂时对 Golang 如何安排目录结构还存在疑惑。

### 20190208 更新

上一个老办法是 Python 版本，是一个整体解决方案。

Python 负责登录 Qzone，Python负责获取照片 URI， Python负责启动多线程下载照片。

在步骤Python登录Qzone中，因为腾讯产品政策的变更，导致Python第三方库 qqlib 无法成功登录Qzone，所以解决方案失效了。

在这次的办法中，将这三个步骤拆分，每个步骤交给最擅长的工具去完成。

文件放在 `/goqzone` 目录。

第一步骤，使用Chrome（或其他浏览器，需要自己去适配）登录 Qzone，此时浏览器记录了Qzone登录需要的 Cookies

第二步骤，使用Go语言编写的程序，借助上一步骤 Chrome 的Cookies，

获取所有相册（也可以自行修改程序增加过滤需求）中所有照片的 URI ，组装为 aria2 格式。

第三步骤，使用 aria2 下载照片，下载使用的配置文件是上一步骤的。

有工具 http://aria2c.com/ 查看下载进度和实时下载速度。


## 步骤详解

### 登录

使用Chrome登录Qzone就是我们的常规使用方式。

如果你使用的是其他浏览器，那么你需要自己适配程序代码中读取浏览器 Cookies 部分。

### 取照片 URI

获取照片程序编译命令

直接运行 
```shell
go build -v -mod=vendor
```

运行命令为 
```
./main -self <你的QQ> -target <目标QQ>
```
运行时需要读取 Chrome Cookies 文件，需要一定权限。在MacOS 表现为弹窗需要输入笔记本密码。

获取照片程序运行后，生成两个给`Aria2`使用的文件
```
$ tree
.
├── aria2.conf
├── qzone.aria2.session

```
同时也会给出你要继续运行的命令，直接拷贝运行即可，比如
```
aria2c --conf-path=/xxx/git_src/qzone_photo/goqzone/src/aria2.conf
```

获取照片URI可能遇到以下错误：

```
got err= 对不起，您尚未登录或者登录超时。”，说明你的浏览器Cookies登录过期了。
```
需要刷新在浏览器中的登录，然后重新运行程序获取照片。

各平台(Windows, Linux, MacOS) 的Chrome Cookies 路径可能不相同，还缺乏适配工作。

目前支持 MacOS Chrome。

### Arai2 下载

aria2 是下载工具中的佼佼者，我们这次也要借助这个工具完成照片的下载。

在实现中，我曾借助 Golang 中便利的并发便利条件，使用 Golang 原生HTTP下载照片，下载效果很不理想。

难以应付死链，下载慢。

aria2 支持我们照片下载中最需要的断点续传需求，同时还支持同一个文件的并发下载。

下载是一个耗时任务，我们可能因为网速不理想下载慢，或者需要断网一段时间后重新恢复下载，因此 aria2 是我们的理想工具。

我曾试图寻找一个 libarai2 的 Golang 绑定(bindings)，来解决多步骤间的衔接问题，但是不理想。

目前的解决方案还是人工手动启动 arai2 下载，人工来辨识下载任务的完成来终结 aria2 的运行。

下载中避免不了死链，有的照片URI就是无法下载了，通过浏览器访问该地址都是超时的，

你可以在 `qzone.aria2.session` 文件中搜索URI，找到该照片的原图URI和正常图片URI，来做进一步处理。

下载文件命名格式为 `<相册名字>_<照片名字>_<照片在相册中的索引>.png` 。

`aria2.conf` 文件中 `load-cookies=` 这一行是使用的浏览器 Cookies 路径，与上面使用的方式一致。

考虑到 Aria2 下载耗时久，如果遇到 Cookies 失效，可以自己重新在 Chrome 中登录一次，然后重新运行下载，

即可断点续传，或者更高级的，更换了 Cookies 读取路径也是可以的。 

------------------------------
# Python 版本的使用方法

- 向脚本填写自己的 QQ 账号，填写 1 个或 多个他人 QQ 号码，批量下载对方所有相册。

- 运行脚本所需的模块都在脚本头部注明。

- 此脚本学习了保存在 GitHub 中多个作者的相册获取方法，成功用此脚本成功获取到指定人的所有相册原图。

- 这种 hack 的方式获取 Qzone 文件是存在有效期的。
    
    - update 2018.01.07 失效
      
      失败位置 qqlib 登陆失败，返回错误 "提交参数错误，请检查"
      
      qqlib 相关 issue https://github.com/gera2ld/qqlib/issues/23
      
      猜测原因是 API 有所调整，相关新闻 
      [永别了塞班，已无法登录QQ和微信](http://www.sohu.com/a/214881879_350699)
    
    - update 2017.05.09 有效
    - update 2017.05.01 有效
    - update 2017.02.20 有效

- 如果遇到无效的，可能是 Qzone API 发生变化，则需要更改代码重新测试。
