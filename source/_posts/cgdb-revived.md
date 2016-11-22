title: cgdb项目复活了
date: 2016-11-14
tags: 
- cgdb
- centos6
thumbnail: /images/cgdb.png
categories: infra
---

vertical split的patch等了7年，终于merge进去了。`cgdb`项目也逐渐活跃了起来。不过`CentOS 6`下却出现了编译问题。

在IRC上和作者聊了之后，由于他没有可重现的环境，这bug就不了了之了。

这里hack了一个版本，用了两套gcc轮番编译链接，弄出了个可用的二进制。具体的方法有点繁琐，这里直接提供编译好的二进制，有需要的请自取。

<!-- more -->

http://pan.baidu.com/s/1jIHgxEY


附我的`$HOME/.cgdb/cgdbrc`
```rc
set arrowstyle=highlight
set autosourcereload
set ignorecase
set tabstop=4
set winminheight=2
set wrapscan
set wso=vertical

map <F5> :step<CR>
map <F6> :next<CR>
map <F7> :finish<CR>
map <F8> :continue<CR>
```
