---
layout: post
title: "在iOS App 图标上绘制版本信息"
date: 2014-02-11T23:42:00
draft: false
comments: true
categories: iOS
published: true
toc: false
---

作为一名 iOS Developer, 想必在日差工作中会频繁的提交 App 的测试版本给 QA 测试, 或者给产品提供演示版本,通常我们并不能直观的通过 App 来判断其版本, 例如我们分发了多个测试版本的话,这样会造成在一个测试设备上会出现多个相同图标的 App, 我们通常区别一个 App 不同版本, 主要通过 App Version 或 Git branch name 来区分不同测试分发版本, 那么在 App 测试版本的 Icon 上添加这些信息话,测试人员就会很容易分别不同版本 App, 即可方便的同时测多个 App 版本.
<!-- more -->

###  工欲善其事, 必先利其器, 我们需要哪些工具呢?

#### [ImageMagick][1]

ImageMagick 是一个功能很强大的开源命令行图片处理程序, 主要用于图片的创建、编辑以及转换等工作.

#### [Ghostscript][2]

Ghostscript 是一套建基于 Adobe、PostScript 及 PDF 的页面描述语言等而编译成的自由软件.这么高大上的东西对于我们来说可能杀鸡用牛刀了, 我们用它来美化我们绘制的文字信息.

在 Mac 下使用 HomeBrew 即可快速安装.

``` bash

brew install imagemagick
brew install ghostscript

```
### 可以在图标上绘制哪些版本信息呢?

* App Short Version
* App Build Version
* Git Branch Name
* Git Commit Hash

这些信息可以简单通过下面 Shell 获取.

``` bash
Version_Short=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${APP}/Info.plist")
Version_Build=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${APP}/Info.plist")
Branch_Name=$(git rev-parse --abbrev-ref HEAD)
Commit_Hash=$(git rev-parse --short HEAD)

```

### 绘制版本信息到图标上

我们使用之前安装的 ImageMagick 软件, 用 Convert 命令在图标的正下方绘制出长 144, 高 40 的矩形,然后使用设定的字体绘制文字到矩形中即可.

``` bash
convert -background '#0008' -fill white -gravity center -size ${width}x40 \
    caption:"${Version_Short} ${Branch_Name} ${Commit_Hash}" \
    ${base_file} +swap -gravity south -composite  "${CONFIGURATION_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/${target_file}"
```

当然你可以随意定制显示区域和字体颜色等, 但从实践上来看使用白色的字体最好看.

### 整合到 Xcode 工程中, 实现自动化

1. 为了防止脚本生成的 Icon 文件出现命名冲突, 我们需要把工程中的所有 Icon 图标添加后缀 "_base"
2. 添加 Run Script 到项目 Target 的 Build Phase 中. 可参照[这里][3]
3. 添加如下脚本到 Run Script 中

``` bash
Version_Short=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${APP}/Info.plist")
Branch_Name=$(git rev-parse --abbrev-ref HEAD)
Commit_Hash=$(git rev-parse --short HEAD)

function processIcon() {
    export PATH=$PATH:/usr/local/bin
    base_file=$1
    base_path=`find ${SRCROOT} -name $base_file`

    if [[ ! -f ${base_path} || -z ${base_path} ]]; then
        return;
    fi

    target_file=`echo $base_file | sed "s/_base//"`
    target_path="${CONFIGURATION_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/${target_file}"

    if [ $CONFIGURATION = "Release" ]; then
    cp ${base_file} $target_path
    return
    fi

    width=`identify -format %w ${base_path}`

    convert -background '#0008' -fill white -gravity center -size ${width}x40\
    caption:"${Version_Short} ${Branch_Name} ${Commit_Hash}"\
    ${base_path} +swap -gravity south -composite ${target_path}
}

processIcon "Icon_base.png"
processIcon "Icon@2x_base.png"
processIcon "Icon-72_base.png"
processIcon "Icon-72@2x_base.png"
```
### 产生的新 Icon 效果图

![Image](http://ww4.sinaimg.cn/large/7853084cjw1f7ayrods46j204704fq34.jpg){: .size-small}


### 或可直接下载完整 Shell 代码
1. [buildIconVersioning.sh][4]  仅在 Build App 时绘制图标, 添加脚本路径到 Scheme 的 Build 下 Pre-actions 的 Run Script 中即可.
2. [archiveIconVersioning.sh][5] 仅在 Archive 时绘制图标, 添加脚本路径到 Scheme 的 Archive 下 Post-actions 的 Run Script 中即可.

[1]: http://www.imagemagick.org/
[2]: http://www.ghostscript.com/
[3]: http://www.runscriptbuildphase.com/
[4]: https://gist.github.com/c0a12efcc06f6cbc616e
[5]: https://gist.github.com/623aee69522f6d747ece

