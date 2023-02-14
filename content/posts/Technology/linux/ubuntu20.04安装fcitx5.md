---
title: "Ubuntu20.04安装fcitx5"
subtitle: ""
date: 2022-07-12T15:57:22+08:00
lastmod: 2022-07-12T15:57:22+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: []

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

<!--more-->

主要参考了[这篇文章](https://ouyen.github.io/fcitx5-ubuntu/),感谢原作！

## 安装

### 安装flatpak

```
sudo add-apt-repository ppa:flatpak/stable
sudo apt update
sudo apt install flatpak
sudo apt install gnome-software-plugin-flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
sudo reboot
```

### 安装flatub仓库

```
# 添加 flatub 仓库, fcitx5-unstable 也会依赖一些这个仓库中的运行时软件包。
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
# 可选部分: 如果你想要使用不稳定版本的fcitx5，也可以添加 fcitx5 非稳定仓库。
# flatpak remote-add --user --if-not-exists fcitx5-unstable https://flatpak.fcitx-im.org/unstable-repo/fcitx5-unstable.flatpakrepo
```

这个安装完之后，桌面会出现一个另一个应用市场。

### 安装fcitx5

```
# 如果您使用的是旧版flatpak，在安装的时候会需要显示的指定软件仓库名字: flatpak install flathub org.fcitx.Fcitx5
flatpak install org.fcitx.Fcitx5
# 中文输入法引擎
flatpak install org.fcitx.Fcitx5.Addon.ChineseAddons 

# 日文输入法（不用装）
# flatpak install org.fcitx.Fcitx5.Addon.Mozc			 

# 安装fcitx5服务，好像是这么个意思
sudo apt install fcitx5-frontend-gtk2 fcitx5-frontend-gtk3 fcitx5-frontend-qt5
```

这里日文输入不用安装。

### 配置环境变量

编辑`~/.profile`，加入：

```
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
```

然后重启。

### 配置开机自启

### 安装主题

我使用这个[主题](https://github.com/hosxy/Fcitx5-Material-Color)，蛮漂亮的，README里面有详细的安装方法。

```
git clone https://github.com/hosxy/Fcitx5-Material-Color.git ~/.local/share/fcitx5/themes/Material-Color
cd ~/.local/share/fcitx5/themes/Material-Color
ln -sf ./theme-teal.conf theme.conf
```

### 安装词库

[萌娘百科词库](https://github.com/outloudvi/mw2fcitx/releases/tag/20220114)

[维基百科词库](https://github.com/felixonmars/fcitx5-pinyin-zhwiki/releases/tag/0.2.3)

下载*.dict文件, 放置到 ~/.local/share/fcitx5/pinyin/dictionaries

## FAQ

### 钉钉不能输入中文

找了钉钉的技术客服，教了我一招，可以参考[这里](https://linuxacme.cn/559)对fcitx5的配置，但是我这里还不一样。

编辑`/opt/apps/com.alibabainc.dingtalk/files/Elevator.sh`文件，在文件声明`#！/bin/bin/sh`下面加入如下内容：

```
export GTK_IM_MODULE DEFAULT=fcitx
export QT_IM_MODULE DEFAULT=fcitx
export XMODIFIERS DEFAULT=@im=fcitx
export INPUT_METHOD DEFAULT=fcitx
export SDL_IM_MODULE DEFAULT=fcitx
export QT_QPA_PLATFORM=xcb
```

然后重启钉钉即可。

之后电脑重启，老毛病又犯了，然后我又去撩了阿里的大佬，得到了解决办法，打开fcitx5的`配置->全局选项->行为->勾选“默认状态为激活”->共享输入状态选“所有”`,一套下来搞定！！

### 喷脑浆IntelliJ系列IDE输入问题

此问题的根本原因是 IDE 附带的 JBR 不正确，要处理此问题，需要：

1. 前往 [这里](https://github.com/RikudouPatrickstar/JetBrainsRuntime-for-Linux-x64/releases) 下载 jbr 并解压到任意路径
2. 更改 IDE 的 JBR,启动IDE后快捷键ctrl+shift+A , 输入 `Choose Boot Java Runtime for the IDE`，选择Add Custom Runtime option，选择解压出的文件夹。