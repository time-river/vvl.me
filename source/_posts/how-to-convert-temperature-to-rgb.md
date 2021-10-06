---
title: 译文：怎么把温度(K)转换成 RGB：算法与示例代码
date: 2017-05-27 23:28:32
categories: style
tags: translation
---

## 摘要

如果你不知道啥是“色温”，看[这里](http://en.wikipedia.org/wiki/Color_temperature)。

使用[PhotoDemo](http://www.tannerhelland.com/photodemon/)这个工具的时候，我竭尽全力尝试找到一个简单、直接有效的算法在温度（Kelven 表示）与 RGB 值之间转换。这看起来是一个很容易找到的算法，因为在后期处理中，许多图片编辑器提供了纠正图片色彩温度的工具，每一个现代的摄像机——包括智能机——提供了一种基于光照条件调节白平衡的方法。

![Example of a camera white balance screen. Image courtesy of http://digitalcamerareviews2011online.blogspot.com](http://www.tannerhelland.com/wp-content/uploads/camera_white_balance-300x235.jpg)
<small>Example of a camera white balance screen. Image courtesy of [http://digitalcamerareviews2011online.blogspot.com](http://digitalcamerareviews2011online.blogspot.com/2011/08/understanding-white-balance-setting-on.html)</small>

或许是我知道得太少，但真的很难找到一种精确的公式把温度转换为 RGB。诚然，有一些算法，但他们大都知识把温度转换到 XYZ 色彩空间，之后你可以添加自己的 RGB 转换。这些算法看样子是基于 AR Robertson 的方法，一种实现在[这里](http://www.brucelindbloom.com/index.html?Eqn_XYZ_to_T.html)，另一种在[这里](http://www.fourmilab.ch/documents/specrend/specrend.c)。

不幸的是，那种方法并不是真正的数学公式——这仅仅是查表插值。在某些特定的情况下，这可能是一个合理的解决方法，但它需要额外的 XYZ -> RGB 转换，这对于简单的实时色温调节太慢了。

因此我设计了我自己的算法，它工作得非常好。下面是我怎么做到的。

## 使用这个算法的注意事项

注意事项1: 我的算法虽然提供了高精度的近似值，但它对严谨的科学来说并不够精确。它主要被用于图片操作——因此不要把它尝试用到天文学与医学图片中。

注意事项2: 因为它相对简单，这个算法对合理大小的图片来说，实时性足够好（我使用 1200 万像素的图片测试过），但为了更好的结果，你最好为你使用的语言做些数值优化。我在这里并不提供数值优化，以避免过度的复杂。

注意事项3: 这算法被设计用于 1000K 到 40000K 的温度范围内，它的在摄影上的光谱非常好（通常情况下，这比大多数的摄影情况还要好一些）。在这温度范围外的模拟，它的效果要差电。

## 特别感谢 Mitchell Charity

First off, I owe a big debt of gratitude to the source data I used to generate these algorithms – Mitchell Charity’s raw blackbody datafile at [http://www.vendian.org/mncharity/dir3/blackbody/UnstableURLs/bbr_color.html](http://www.vendian.org/mncharity/dir3/blackbody/UnstableURLs/bbr_color.html). Charity provides two datasets, and my algorithm uses the [CIE 1964 10-degree color matching function](http://cvrl.ioo.ucl.ac.uk/database/text/cmfs/ciexyz64.htm). A discussion of the CIE 1931 2-degree CMF with Judd Vos corrections versus the 1964 10-degree set is way beyond the scope of this article, but you can [start here](http://www.vendian.org/mncharity/dir3/blackbody/parameters.html) for a more comprehensive analysis if you’re so inclined.

## 算法：简单输出

1000K 到 40000K 范围的输出：

![Output of my algorithm from 1000 K to 40000 K. The white point occurs at 6500-6600 K, which is perfect for photo manipulation purposes on a modern LCD monitor.](http://www.tannerhelland.com/wp-content/uploads/Temperature_to_RGB_1000_to_40000.png)
<small>Output of my algorithm from 1000 K to 40000 K. The white point occurs at 6500-6600 K, which is perfect for photo manipulation purposes on a modern LCD monitor.</small>

一个更小范围内的效果，1500K 至 15000K 之间：

![Same algorithm, but from 1500 K to 15000 K](http://www.tannerhelland.com/wp-content/uploads/Temperature_to_RGB_1500_to_15000.png)
<small>Same algorithm, but from 1500 K to 15000 K</small>

正如你所看到的，条纹很小——这与前面所提到的查表方法相比，有很大的改进。这个算法还在这方面做得很好：保留了微弱的黄色，直至全白，这对图片的后期处理操作中模仿日光很重要。

## 在这个算法中我是怎么做到的

在逆向工程中我的第一步是根据可靠的公式绘制 [Charity 的原始黑体值](http://www.vendian.org/mncharity/dir3/blackbody/UnstableURLs/bbr_color.html)。你可以在[这里](http://www.tannerhelland.com/code/Temperature_to_RGB_Worksheet.ods)下载我整个的工作表。

![Mitchell Charity’s original Temperature (K) to RGB (sRGB) data, plotted in LibreOffice Calc. Again, these are based off the CIE 1964 10-degree CMFs. The white point, as desired, occurs between 6500 K and 6600 K (the peak on the left-hand side of the chart). (Source: http://www.vendian.org/mncharity/dir3/blackbody/UnstableURLs/bbr_color.html)](http://www.tannerhelland.com/wp-content/uploads/Raw_temperature_vs_RGB_chart.png)
<small>Mitchell Charity’s original Temperature (K) to RGB (sRGB) data, plotted in LibreOffice Calc. Again, these are based off the CIE 1964 10-degree CMFs. The white point, as desired, occurs between 6500 K and 6600 K (the peak on the left-hand side of the chart). (Source: [http://www.vendian.org/mncharity/dir3/blackbody/UnstableURLs/bbr_color.html](http://www.vendian.org/mncharity/dir3/blackbody/UnstableURLs/bbr_color.html))</small>

对于这些数据，很容易注意到存在一些极值，这让我们的算法变得更加容易实现。特别的：

* 温度低于 6600K，R 值总是 255
* 温度低于 2000K，B 值总是 0
* 温度高于 6500K，B 值总是 255

这对于拟合由这些数据组成的曲线很重要，绿色最好被视为由两条独立的曲线组成——温度低于 6600K 是一条，高于 6600K 则是另一条。

从这里也可以看到，我把数据分割成独立的颜色组件（不包括恒 0 与恒 255 这两段）。更完美的情况下，曲线会拟合到每一点，但不幸的是绝不会那么简单。因为在这张图中 X 与 Y 值之间存在很大的差距—— x 值的返回从 0 到 1000，中间只有 100 个点，y 值 从 0 到 255 —— 这必须做一些优化，令 x 值更好地拟合这曲线。因此，我把每个颜色的 x 值除以 100，之后再适当地做些减法，以便更好地拟合。下面是拟合的结果:

![modified temperature to red plot](http://www.tannerhelland.com/4435/convert-temperature-rgb-algorithm-code/modified_temperature_to_red_plot/)  
![modified temperature to green plot 1](http://www.tannerhelland.com/4435/convert-temperature-rgb-algorithm-code/modified_temperature_to_green_plot/)  
![modified temperature to green plot 2](http://www.tannerhelland.com/wp-content/uploads/Modified_Temperature_to_Green_plot_2.png)  
![modified temperature to blue plot](http://www.tannerhelland.com/wp-content/uploads/Modified_Temperature_to_Blue_plot.png)  

Apologies for the horrifically poor font kerning and hinting in those charts. I love LibreOffice for many things, but its inability to do font aliasing on charts is downright shameful. I also don’t like having to extract charts from screenshots because they don’t have an export option, but that’s a rant best saved for some other day.

正如你所看到的，曲线拟合得很好，R 值都高于 0.987。我可以花费更多的时间来调整曲线，但对于图片处理来说，这已经足够好了。外行人看不出这些曲线不完全复合原始的理想化的黑体观测值，不是吗？

## 算法

使用这些数据，我设计了下面的算法。

第一步，伪代码：

```md
    Start with a temperature, in Kelvin, somewhere between 1000 and 40000.  (Other values may work,
     but I can't make any promises about the quality of the algorithm's estimates above 40000 K.)
    Note also that the temperature and color variables need to be declared as floating-point.

    Set Temperature = Temperature \ 100

    Calculate Red:

    If Temperature <= 66 Then
        Red = 255
    Else
        Red = Temperature - 60
        Red = 329.698727446 * (Red ^ -0.1332047592)
        If Red < 0 Then Red = 0
        If Red > 255 Then Red = 255
    End If

    Calculate Green:

    If Temperature <= 66 Then
        Green = Temperature
        Green = 99.4708025861 * Ln(Green) - 161.1195681661
        If Green < 0 Then Green = 0
        If Green > 255 Then Green = 255
    Else
        Green = Temperature - 60
        Green = 288.1221695283 * (Green ^ -0.0755148492)
        If Green < 0 Then Green = 0
        If Green > 255 Then Green = 255
    End If

    Calculate Blue:

    If Temperature >= 66 Then
        Blue = 255
    Else

        If Temperature <= 19 Then
            Blue = 0
        Else
            Blue = Temperature - 10
            Blue = 138.5177312231 * Ln(Blue) - 305.0447927307
            If Blue < 0 Then Blue = 0
            If Blue > 255 Then Blue = 255
        End If

    End If
```

在上述的伪码中，记住 [Ln() 是自然对数](http://en.wikipedia.org/wiki/Natural_logarithm)。也需要注意，如果使用在推荐的温度范围内，也可以忽略 `if color < 0`（但仍然需要 `if color > 255`）。

对于实际的代码，下面是我使用在 [PhotoDemon](http://www.tannerhelland.com/photodemon/) 中的 Visual Basic 函数。它并没有做任何的优化（毕竟，这已经比查表要快很多了），但代码足够少，而且健壮。

```vb
'Given a temperature (in Kelvin), estimate an RGB equivalent
Private Sub getRGBfromTemperature(ByRef r As Long, ByRef g As Long, ByRef b As Long, ByVal tmpKelvin As Long)

    Static tmpCalc As Double

    'Temperature must fall between 1000 and 40000 degrees
    If tmpKelvin < 1000 Then tmpKelvin = 1000
    If tmpKelvin > 40000 Then tmpKelvin = 40000

    'All calculations require tmpKelvin \ 100, so only do the conversion once
    tmpKelvin = tmpKelvin \ 100

    'Calculate each color in turn

    'First: red
    If tmpKelvin <= 66 Then
        r = 255
    Else
        'Note: the R-squared value for this approximation is .988
        tmpCalc = tmpKelvin - 60
        tmpCalc = 329.698727446 * (tmpCalc ^ -0.1332047592)
        r = tmpCalc
        If r < 0 Then r = 0
        If r > 255 Then r = 255
    End If

    'Second: green
    If tmpKelvin <= 66 Then
        'Note: the R-squared value for this approximation is .996
        tmpCalc = tmpKelvin
        tmpCalc = 99.4708025861 * Log(tmpCalc) - 161.1195681661
        g = tmpCalc
        If g < 0 Then g = 0
        If g > 255 Then g = 255
    Else
        'Note: the R-squared value for this approximation is .987
        tmpCalc = tmpKelvin - 60
        tmpCalc = 288.1221695283 * (tmpCalc ^ -0.0755148492)
        g = tmpCalc
        If g < 0 Then g = 0
        If g > 255 Then g = 255
    End If

    'Third: blue
    If tmpKelvin >= 66 Then
        b = 255
    ElseIf tmpKelvin <= 19 Then
        b = 0
    Else
        'Note: the R-squared value for this approximation is .998
        tmpCalc = tmpKelvin - 10
        tmpCalc = 138.5177312231 * Log(tmpCalc) - 305.0447927307

        b = tmpCalc
        If b < 0 Then b = 0
        If b > 255 Then b = 255
    End If

End Sub
```

这个函数被用于生成下面的示例图片。

## 示例图片

这是一个非常好的示例。下面的图片—— HBO 发行的 True Blood 系列海报——用作演示光线调节效果非常好。左边是原始图片；右边是经过色温调整后的图片。

![Color temperature adjustments in action.](http://www.tannerhelland.com/wp-content/uploads/True_Blood_Color_Temperature_Demo-600x450.jpg)

我在 [PhotoDemon](http://www.tannerhelland.com/photodemon/) 项目中使用的色温调节工具如下所示：

![PhotoDemon’s Color Temperature tool.](http://www.tannerhelland.com/wp-content/uploads/PhotoDemon_50_Color_Temperature_Tool.jpg)

点击[这里](http://photodemon.org/download)下载。

## addendum October 2014

[Renaud Bédard](https://twitter.com/renaudbedard) has put together a great online demonstration of this algorithm. [Check it out here](https://www.shadertoy.com/view/lsSXW1), and thanks to Renaud for sharing!

## addendum April 2015

Thank you to everyone who has suggested improvements to the original algorithm. I know there are a lot of comments on this article, but they’re worth reading if you’re planning on implementing your own version.

I’d like to call out two specific improvements. First, Neil B has helpfully provided a better version of the original curve-fitting functions, which results in slightly modified temperature coefficients. [His excellent article](http://www.zombieprototypes.com/?p=210) describes the changes in detail.

Next, Francis Loch has added some comments and sample images below, which are very helpful if you want to apply these corrections to a photograph. His modifications produce a much more detailed image, as [his sample images demonstrate](http://www.tannerhelland.com/4435/convert-temperature-rgb-algorithm-code/#comment-24753).

## Similar Posts

* [Simple algorithms for adjusting image temperature and tint](http://www.tannerhelland.com/5675/simple-algorithms-adjusting-image-temperature-tint/)  
* [Announcing PhotoDemon 5.0 – Everything is Faster, Everything is Better](http://www.tannerhelland.com/4407/photodemon-5-0-update/)  
* [Image Dithering: Eleven Algorithms and Source Code](http://www.tannerhelland.com/4660/dithering-eleven-algorithms-source-code/)  
* [Blur Filter performance: PhotoDemon vs GIMP vs Paint.NET](http://www.tannerhelland.com/5109/performance-photodemon-gimp-paintnet/)  
