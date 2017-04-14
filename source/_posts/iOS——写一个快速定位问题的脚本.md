title: iOS——写一个快速定位问题的脚本
date: 2017-04-04 11:15:38
tags:
  - iOS
categories:
  - iOS
---

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-12.png)

# 你是否见过？

1. 你是否见过测试人员或者自己在 CI 上 install 了一个版本，发现了 BUG 后，突然忘了自己下的是 CI 上的哪一个 commit 的包？
2. 你是否见过下面这个东西：

![blog_iOS——写一个快速定位问题的脚本-01](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-01.jpg)

![blog_iOS——写一个快速定位问题的脚本-02](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-02.png)

<!-- More -->

# 写一个这样一个脚本

可以写这样一个脚本，它能做到：

1. 在 Build 的过程中在 App Icon 的表面覆盖上 Build 号、分支名、commit version 的 hash 值
2. 不影响原本的 App Icon 图标源文件
3. 区分 `Release` 和 `Debug`，只在 `Debug` 环境下 Build 项目时执行脚本

# Do it

## ImageMagick

做后端的同学们，大多知道 [ImageMagick](http://www.imagemagick.org/script/index.php)。

> 使用 ImageMagick 可以创建、编辑、合成或转换图片。它可以读和写各种格式的图像（超过 200 种格式）包括 PNG、JPEG、JPEG - 2000、GIF、TIFF、DPX、EXR、WebP、Postscript、PDF、SVG。ImageMagick 可以调整、翻转、镜像、旋转、扭曲、剪切和转换图像、图像色彩调整，适用于各种特殊效果,或绘制文本、线、多边形、椭圆和贝塞尔曲线。

通过 shell command 就可以轻易使用以上功能。

-------

让我们来看一些基本的。这是我们准备好的 `120*120` 的原图：

![blog_iOS——写一个快速定位问题的脚本-03](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-03.png)

`cd` 到图片所在的目录，执行以下命令，给图片添加高斯模糊效果：

```bash
convert original.png -blur 10x8 blurred.png
```

完成图：

![blog_iOS——写一个快速定位问题的脚本-04](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-04.png)

-------

继续，执行以下命令从 坐标 `(0,60)` 剪切成 `120*60` 的图片，

```bash
convert blurred.png -crop 120x60+0+60 cropped-blurred.png
```

完成图：

![blog_iOS——写一个快速定位问题的脚本-05](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-05.png)

-------

继续，给图片添加文字水印『zhoulingyu』，参数包括：背景不填充颜色、白色字体、字体大小 12、居中显示文字、文字为『zhoulingyu』：

```bash
convert -background none -fill white -pointsize 12 -gravity center caption:"zhoulingyu" cropped-blurred.png +swap -composite label.png
```

完成图：

![blog_iOS——写一个快速定位问题的脚本-06](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-06.png)

-------

继续，将上面得到的剪切好的带水印的 `label.png` 和 原图 `original.png` 合成在一起：

```bash
composite label.png original.png finished-image.png
```

完成图：

![blog_iOS——写一个快速定位问题的脚本-07](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-07.png)


OK，我们得到了想要的效果图。

## 参数参考

给出一些 ImageMagic 的常用用法：

1. 查看图片信息

```bash
identify original.png 
original.png PNG 120x120 120x120+0+0 8-bit sRGB 46c 2.58KB 0.010u 0:00.000
```

2. 格式转换

```bash
convert original.png original.jpg 
```

3. 编辑图片大小

```bash
convert original.png -resize 200x200 resize-image.png 
```

4. 裁剪

```bash
# 从坐标 (0,0) 裁剪 100*100 的图像
convert original.png -crop 100x100+0+0 crop.png  
```

5. 旋转

```bash
convert original.png -rotate 45 rotate.png 
```

6. 合并图像

```bash
# 给图片添加水印
convert original.png -compose over watermark.png -composite new-image.png  
```

7. 高斯模糊

```bash
convert -blur 80x5 original.jpg blur.png
```
-blur radiusxsigma，两个分别是高斯模糊需要的两个参数，具体可以查看 [blur 参数使用](https://www.imagemagick.org/script/command-line-options.php#blur)

ImageMagick 可以实现 N 多效果，像油画、噪声、散射、旋涡，都不在话下。

除了基本的效果，还有一些比较常用的参数：

| 参数名 | 使用规范 | 说明 | 用例 |
| --- | --- | --- | --- |
| -background | -background color | 设置背景色 | -background white |
| -pointsize | -pointsize value | 设置字体等大小 | -pointsize 12 |
| -gravity | -gravity type | 为其他命令附加 gravity，比如设置文字添加位置居中。 | -gravity Center |
| -geometry | -geometry geometry | 设置即将处理图像的坐标位置 | -geometry +0+60 -geometry Center |

当然这些都可以在 [官方文档](https://www.imagemagick.org/script/command-line-options.php) 找到。

## shell 脚本

### 1. 基本设置

知道 ImageMagic 如何使用，剩下来写脚本就思路清晰多了。

在工程 `Target` -> `Build Phases` 中新建一个 Run Script，我们可以给它起名 `generate auxiliary icon`，这样稍后容易在 `Report Navigator` 观察。

![blog_iOS——写一个快速定位问题的脚本-08](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-08.png)

现在我们可以开始编写我们的脚本 `auxiliary_icon.sh`

### 2. 理思路

写伪代码通常能够帮助我自己更清晰的写代码：

```
// 1. 判断执行 Build 的机器是否安装了 ImageMagic
//    |- 如果没有安装：提示安装，退出脚本
//    |- 如果安装：继续执行
// 2. 获取 commit 号 hash 值、分支名、build 号，并将其拼接成一个字符串
// 3. 判断编译环境
//    |- 如果是 Release 环境：提示当前是 Release 环境，退出脚本
//    |- 如果是非 Release 环境：继续执行
// 4. 获取 Plist 中 CFBundleIconFiles 的数量
// 5. 根据数量循环，执行调用『生成记号图方法』


// 『生成记号图方法』 
// function generateIcon() {
// 1. 模糊图片
// 2. 截取图片下半部分
// 3. 添加 commit+brach+build 组成的字符串在截取图片上
// 4. 合成截取图片和原图
// 5. 清除多余图片
// }
```

伪代码写好了，开始编写正式代码：

1. 判断执行 Build 的机器是否安装了 ImageMagic

which 一下就知道

```bash
convertPath=`which convert`
# 判断 convertPath 文件是否存在
if [ ! -f ${convertPath}]; then
echo "==============
WARNING: 你需要先安装 ImageMagick！！！！:
brew install imagemagick
=============="
exit 0;
fi
```

2. 获取 commit 号 hash 值、分支名、build 号，并将其拼接成一个字符串

PlistBuddy 可以用于读取 Plist 文件，通过描述路径就可以找到你想知道的 Key 对应的 Value。
`${INFOPLIST_FILE}` 是 xcodebuild 提供的变量（具体可以参考 [Build settings reference](http://help.apple.com/xcode/mac/8.0/#/itcaec37c2a6)）提供了编译后 `info.plist` 的路径。

```bash
commit=`git rev-parse --short HEAD`
branch=`git rev-parse --abbrev-ref HEAD`
buildNumber=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${INFOPLIST_FILE}"`
caption="${buildNumber} \n${branch}\n${commit}"
```

3. 判断编译环境

```bash
# Release 不执行
echo "Configuration: $CONFIGURATION"
if [ ${CONFIGURATION} = "Release" ]; then
exit 0;
fi
```

4. 获取 Plist 中 CFBundleIconFiles 的数量

在编译后的 `info.plist` 中，可以找到如下结构：

![blog_iOS——写一个快速定位问题的脚本-09](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-09.png)

这里记录了所有的 Icon files。查看 plist 的原格式，可以看到原始的 key 是什么。通过 PlistBuddy 和 路径 `CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles` 可以输出 value。

`| wc -l` 可以统计输出行数。

```bash
icon_count=`/usr/libexec/PlistBuddy -c "Print CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}" | wc -l`
```

要注意的是， `/usr/libexec/PlistBuddy -c "Print CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}"` 输出结果是这样的：

![blog_iOS——写一个快速定位问题的脚本-10](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-10.png)

输出一共是 **五行**，所以获得的结果是 5。

那么真实的 CFBundleIconFiles count 其实是：

```bash
real_icon_index=$((${icon_count} - 2))
```

刚开始，我也没有注意。可想而知心情如何 =_=。

![blog_iOS——写一个快速定位问题的脚本-11](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%86%99%E4%B8%80%E4%B8%AA%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D%E9%97%AE%E9%A2%98%E7%9A%84%E8%84%9A%E6%9C%AC-11.JPG)

5. 根据数量循环，执行调用『生成记号图方法』

```bash
for ((i=0; i<$real_icon_index; i++)); do
# 找到 icon 名
icon=`/usr/libexec/PlistBuddy -c "Print CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles:$i" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}"`

# 调用 generateIcon 方法，传入 icon 名
generateIcon "${icon}@2x.png"

done
```

6. generateIcon 方法

```bash
function generateIcon() {
    originalImg=$1

    cd "${CONFIGURATION_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}"

    # 验证存在性
    if [[ ! -f ${originalImg} || -z ${originalImg} ]]; then
    return;
    fi

    # 进入编译后的工程目录
    cd "${CONFIGURATION_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/"

    # 添加高斯模糊
    convert ${originalImg} -blur 10x8 blur-original.png

    # 截取下部分
    width=`identify -format %w ${originalImg}`
    height=`identify -format %h ${originalImg}`
    height_0=`expr ${height} / 2`
    height_1=$((${height} - ${height_0}))
    convert blur-original.png -crop ${width}x${height_0}+0+${height_1} crop-blur-original.png

    # 加字
    point_size=$(((8 * $height) / 58))

    convert -background none -fill white -pointsize ${point_size} -gravity center caption:"${caption}" crop-blur-original.png +swap -composite label.png

    # 合成
    composite -geometry +0+${height_0} label.png ${originalImg} ${originalImg}

    # 清除文件
    rm blur-original.png
    rm crop-blur-original.png
    rm label.png
}
```

# 代码清单

最终的代码清单放在了 Github 上：

> [ZLYWatermarkIcon](https://github.com/summertian4/ZLYWatermarkIcon)

# 参考

> [Command-line Tools:Convert](http://www.imagemagick.org/script/convert.php)
> [使用ImageMagick添加图片水印－Linux](http://blog.topspeedsnail.com/archives/7783)
> [krzysztofzablocki/IconOverlaying](https://github.com/krzysztofzablocki/IconOverlaying)
> [Overlaying application version on top of your icon](http://merowing.info/2013/03/overlaying-application-version-on-top-of-your-icon/)
> [我的ImageMagick使用心得](http://www.charry.org/docs/linux/ImageMagick/ImageMagick.html)
> [高斯模糊的算法](http://www.ruanyifeng.com/blog/2012/11/gaussian_blur.html)
> [Build settings reference](http://help.apple.com/xcode/mac/8.0/#/itcaec37c2a6)
> [Shell脚本编程30分钟入门](https://github.com/qinjx/30min_guides/blob/master/shell.md)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

