---
title: Hexo与Next主题出现文字间隔空白的解决方案
date: 2020-09-01 09:12:07
categories: 技术
tags: 
- Hexo
---

## 前言

博客上线后，写了两篇文章。虽然其他效果都不错，但是发现有时一行文字之间会有很多的空白出现，很大程度影响布局和观感，尤其是标题或中英文混合起来的那种效果太难看了。在手机上观看更是惨不忍睹的感受。

简单来讲，就是像下面这种样子：

{% cq %}
<p style="text-align: justify;">文本总是被像后面这一个很长很长的，占地方的代码块，比如<code>collections.abc.MutableMapping</code>。或其他的东西分隔开，并添加很多的空格。</p>
{% endcq%}

如果你遇到了同样的问题，请尝试本文提供的方法。
<!-- more -->

## 解决方案

1. **打开Next主题配置文件。** 默认是`/themes/next/_config.yml`。

{% note warning %}
根据Next官方文档所建议的，应该在Hexo目录下创建一个名为`_config.[主题名称].yml`**的副本。  
（避免在更新Next主题时，你自己的配置文件和更新的配置文件发生冲突）。如果你如此照做了，那么你需要编辑的文件是`/_config.next.yml`。

如果你没有创建过这样的主题副本，也可以考虑一下，或忽视本提示。见[Next官方文档](https://theme-next.js.org/docs/getting-started/configuration.html#config-name-yml)。
{% endnote %}

![文件路径](/images/Hexo与Next主题出现文字间隔空白的解决方案/file-path.png)
2. **找到`text_align`选项，将两个子项改为`left`。** 这两项分别定义了在桌面设备(desktop)和移动设备(mobile)上的文本对齐效果。如果你遇到的问题和我相同，那么默认应该是`justify`。

![选项位置](/images/Hexo与Next主题出现文字间隔空白的解决方案/option-position.png)

*将选项修改为left：*

![修改选项](/images/Hexo与Next主题出现文字间隔空白的解决方案/change-option.png)
3. **重新生成Hexo博客并启动服务器或部署，预览结果。** 最终效果应该会类似本博客现在的观感。

## 原因

Next主题默认将CSS的文本对齐样式设置为`justify`，也就是对文章应用了[`text-align: justify;`](https://developer.mozilla.org/en-US/docs/Web/CSS/text-align)的CSS样式。其原意为**左右对齐**。

设置了该样式会导致文本将按行试图对齐容器左右的宽度，对于较长的文字行没有什么问题，但是对于被分割换行的文字（例如被一个`<code>`代码块所影响），这将会在行中添加大量空白。

所以我们将`text_align`选项改为`left`，即左对齐。确切来讲，对于Next主题，是将`.post-body`的`text-align` CSS样式改为`left`。

如果你用的是其他主题或者别人的设置，并遇到了上述问题，也可以找找看主题设置里有没有文本对齐的选项。如果没有也可以自己改动主题的CSS文件，将对齐改为左对齐即可（或者……你想用居中对齐`center`之类的，反正，`justify`不适合博客正文）。
