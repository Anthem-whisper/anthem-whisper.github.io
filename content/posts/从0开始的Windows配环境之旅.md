---
title: "从0开始的Windows配环境之旅"
date: 2023-02-08T16:06:36+08:00
draft: false
image: 
description: 
comments: true
license: true
math: false
categories:
- Misc
tags:
---



用了2.5年的matebook实在是有点小卡，总共就512G的硬盘也爆炸了，瞧了瞧M2 pro的牙膏，心想还是再苟一苟，等等党永远不亏，所以就下狠心直接重装了Windows。

## 重装Windows

这没啥好说的，[wepe](https://www.wepe.com.cn/)，[NEXT, ITELLYOU](https://next.itellyou.cn/)，[傲梅分区助手（wepe自带）](https://www.disktool.cn/)

十多分钟搞定，主要是配环境和其他的问题。

## 配置各种环境

语言倒不是很麻烦，各种语言能上多版本管理工具的都上了，版本之间环境隔离，用起来挺舒服

- **PHP**：[PHPstudy](https://www.xp.cn/)，打过CTF的都说好，还自带MySQL，nginx等多版本管理的功能，谁用谁知道。不过要注意低版本以前出过后门事件，使用的时候留意一下。
- **Node.js**：[nvm-windows](https://github.com/coreybutler/nvm-windows)，Go写的nodejs多版本管理工具，通过软链接和环境变量实现版本隔离，每个版本的npm也是隔离的，node默认安装在nvm同级目录，也不用担心占用C盘空间。
- **Golang**：[g(gvm)](https://github.com/voidint/g)，各平台通用的golang版本管理工具，也是通过软链接实现版本切换。默认会在用户目录下创建`.g`目录，担心占用C盘的可以在cmd下用`mklink /J `链接到其他地方去。
- **Java**：无工具，直接通过修改`JAVA_HOME`环境变量实现版本切换，担心maven仓库占用C盘的用`mklink /J`命令把`.m2`目录链接到D盘去
- **Python**：没多版本切换的需求，只有Python2，3并存的需求，直接去官网下载安装包分别装到两个目录就行了，然后把python2的`python.exe`复制一份重命名为`python2.exe`，原来的`python.exe`别删，删了不好使。配环境变量的时候把python3目录放到python2目录之前，就可以保证使用命令`python`的时候执行python3。对于pip，2.7版本可以直接在换源之后`curl https://bootstrap.pypa.io/pip/2.7/get-pip.py | python2`来安装。

IDE

- [**IntelliJ IDEA**](https://www.jetbrains.com/zh-cn/idea/)，宇宙最强Java IDE，没啥好说的
- [**PhpStorm**](https://www.jetbrains.com/phpstorm/)，调php代码用
- [**Visual Studio Code**](https://code.visualstudio.com/)，除了Java和php其他都用这个，插件多，生态丰富

Docker和wsl肯定不可或缺，装docker之前先装wsl2

- **WSL**：照着[知乎](https://zhuanlan.zhihu.com/p/386590591)的装，只不过我在store里面选择的Kali，这个Kali还可以装GUI，体验很不错。照着[ms的文档](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config)配置性能参数
- **Docker**：[Docker desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)，照着官网来，别忘了换源啥的。
- **迁移wsl和docker的数据**：担心占C盘用WSL管理工具[LxRunOffline](https://github.com/DDoSolitary/LxRunOffline)，我用的mingw版本，可以把Docker和WSL的硬盘都迁移到D盘去

Terminal、powershell7和美化

- **Terminal**直接在store里面搜，powershell去[Github release](https://github.com/PowerShell/powershell/releases)下msi安装

- **美化**

  - 字体使用[Nerd Fonts](https://www.nerdfonts.com/font-downloads)，我用的Hack，看起来是[这样](https://www.programmingfonts.org/#hack)

  - Oh-my-posh现在已经不支持powershell module了，按照[官方文档](https://ohmyposh.dev/docs/installation/windows)照着装

  - 主题我用的stelbent，可以在powershell里输入`notepad $profile`来设置，分享下我的：

    ```
    oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\stelbent.minimal.omp.json" | Invoke-Expression
    Import-Module posh-git
    ```

其他

- Git、[GitHub Desktop](https://desktop.github.com/)

- [utools](https://www.u.tools/)，这个强烈推荐，里面的ctool超级好用，一般人我不告诉他。

- [Clash For Windows](https://github.com/Fndroid/clash_for_windows_pkg/releases)，懂得都懂
- [V2rayN](https://github.com/2dust/v2rayN/releases)，也很棒，多一个代理防止打内网的时候不能访问Google
- [geek uninstaller](https://geekuninstaller.com/)，卸载软件用
- [PicGo](https://picgo.github.io/PicGo-Doc/zh/guide/)，配合GitHub图床，写博客用
- VMware Workstation Pro，网上直接找激活码

## 对抗国产流氓软件

### 火绒高级防护

Windows Defender作为杀软确实不错，我的那些工具一下就给我吞了，但是在整治国产流氓软件这方面还得看火绒，推荐火绒的"高级防护"功能，可以自定义防护规则，防止创建目录和读取浏览器历史之类的。

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202302081710228.png)

规则可以在[火绒论坛](https://bbs.huorong.cn/forum-45-1.html)里找到，我用的是[火绒安全自定义规则 v29.0.zip](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202302081713589.zip)，导入的时候需要同时导入规则和自动处理。

### 根治流氓软件

因为我习惯了tim和360压缩，导入规则之后，火绒就疯狂给我告警。

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202302081716497.png)]

但是由于一些原因，我有时候需要关闭火绒，这就给了这些软件可乘之机。

之前打awd的时候杀不死马可以创建同名目录来防止文件再生，这里我迁移一下，创建一个同名文件来防止目录创建

<img src="https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202302081718691.png" style="zoom: 50%;" /><img src="https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202302081719560.png" style="zoom: 50%;" />

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202302081720198.png)

根据火绒安全日志里面的目录，一个个创建，注册表同理，创建一个键然后权限全部关掉。

然后世界就清净了。

## 小结

一套下来，C盘占用不到50G，D盘占不到100个G，四舍五入又拥有了一台新电脑。



