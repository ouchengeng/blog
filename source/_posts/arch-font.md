title: Archlinux下中文字体问题
date: 2016-11-21
tags: font
categories: tips
---

Archlinux下使用`infinality`显示中文字体在特定字体大小下，字符渲染会出现问题。可通过以下方法解决，
```
sudo cp /usr/share/doc/freetype2-infinality-ultimate/infinality-settings.sh /etc/profile.d/
```
并修改里面的默认配置
```
export INFINALITY_FT_STEM_ALIGNMENT_STRENGTH="0"
export INFINALITY_FT_STEM_FITTING_STRENGTH="0"
```
重启即可。
[思路来源](https://bbs.archlinuxcn.org/viewtopic.php?id=4249)
