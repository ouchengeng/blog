title: 一流的人体工程学1
date: 2016-11-15
tags:
- i3
- urxvt
- tmux
- emacs
- osc52
- copyq
thumbnail: /images/test.gif
categories: ergonomics
---


前言
===

本文是介绍人体工程学开发的第一篇博文，涉及到的软件有，
- Archlinux
- i3wm
- urxvt
- copyq
- emacs, spacemacs
- tmux


<!-- more -->

正文
===

窗口切换
---

我使用i3 workspace绑定每个应用程序，并一一对应热键进行切换。具体的映射关系和应用名称相关，火狐（ctrl+alt+f），win虚拟机（ctrl+alt+w），emacs（ctrl+alt+e）。
<video src="/videos/switch-windows.webm" width="100%" height="100%" controls></video>

研发工作台
---

我的研发工作会在一台远端服务器上进行。这有如下几个好处
1. 服务器长时间运行，工作状态不会被中断（后面会提到高可用配置，保证服务器宕机也能回滚到最近工作状态）
2. 开发编译性能高，可用内存大
3. 与生产环境类似，便于测试
4. 内网部署十分快速

在这个前置要求下，我设计了一套开发方案--研发工作台。该工作台包括两个窗口，使用i3的tab layout叠在一起。他们分别是tmux window和emacs window。如下面视频所示。

<video src="/videos/open-workspace.webm" width="100%" height="100%" controls></video>

tmux与emacs均是daemon模式运行，客户机使用如下命令连接
```
#!/usr/bin/env bash

urxvt -title nobida201 -e ssh nobida201 -t "tmux new -s amos || tmux a -t amos" &
urxvt -title nobida201 -e ssh nobida201 -t "emacsclient -c" &
```

客户机可以随时关闭窗口，不中断开发。

这个方案将开发过程分为两大块，1. 代码编辑，2. 终端操作。emacs基本可以满足代码编辑的全部需求，但是其终端模拟器非常差，而tmux对终端的操作是目前顶级的，它们两的有机结合可以极大地提升开发效率与幸福感。

下面介绍客户机如何同研发工作台无缝互动。

### emacs与tmux切换

这里比较简单的操作是窗口直接切换，通过`i3-msg focus right`即可。然而，很多开发人员在操作终端的时候，都喜欢直接用vim或emacs打开文件编辑，这在tmux下是不合适的，不仅热键会有冲突，而且不利于tmux内部的窗口管理。这里展示一个很高端的技巧，在tmux里执行命令，直接在对应的emacs窗口里打开编辑。更厉害的是，如果tmux里用了ssh到其他服务器，该命令依然有效（只要是在该服务器可达的网络链路，无论多少跳都可以）。

<video src="/videos/open-file.webm" width="100%" height="100%" controls></video>

具体实现如下，

客户机urxvt perl脚本
```perl
#!/usr/bin/env perl

use MIME::Base64;
use Encode;

sub on_osc_seq {
  my ($term, $op, $args) = @_;
  return () unless $op eq 52;
  my ($clip, $data) = split ';', $args, 2;
  if ($data eq '?') {
    my $data_free = $term->selection();
    Encode::_utf8_off($data_free);
    $term->tt_write("\e]52;$clip;".encode_base64($data_free, '')."\a");
  }
  elsif ($clip =~ /x/) {
    my $data_decoded = decode_base64($data);
    Encode::_utf8_on($data_decoded);
    my $host_name = $term->resource('title');
    $data_decoded =~ s/^\/$host_name://;
    $term->exec_async('ssh', $host_name, 'emacsclient -n',  $data_decoded);
    $term->exec_async('i3-msg focus right');
  }
  else {
    my $data_decoded = decode_base64($data);
    Encode::_utf8_on($data_decoded);
    $term->selection($data_decoded, $clip =~ /c/);
    $term->selection_grab(urxvt::CurrentTime, $clip =~ /c/);
  }
  ()
}
```

服务器[sshrc](https://github.com/Russell91/sshrc)配置
```rc
# cannot be used outside tmux
function e() {
  if [ $# -ne 1 ]; then
      echo "Usage: $0 <filename>"
      exit 1
  fi
  max=74994
  file=$1
  if [[ -d $file || -f $file ]]; then
      buf="/`hostname`:`readlink -f $file`"
      esc="\033]52;x;$( printf %s "$buf" | head -c $max | base64 | tr -d '\r\n' )\a"
      esc="\033Ptmux;\033$esc\033\\" # tmux
      printf "$esc"
  else
      echo "Filename is invalid."
  fi
}
```

服务器bashrc
```rc
alias ssh='sshrc'
source /home/amos/.sshrc
```

### 客户机与服务器的剪贴板互动(非X11转发)

在终端操作时遇到错误信息，如何复制出来进行google搜索，是非常频繁的操作。然而在tmux下，这变得有一些困难。这里介绍一个几乎完美的解决方案（有缺憾的地方是，需要osc52的terminal），能够直接将tmux的选择块复制到客户机的剪贴板中。该方案还有一个更厉害的扩展，支持任意终端(ssh多次)emacs，vim直接复制到客户机的剪贴板中。

<video src="/videos/clipboard.webm" width="100%" height="100%" controls></video>

该方案需要前文所述的urxvt perl脚本支持。

tmux配置
```
set -g terminal-overrides '*88col*:colors=88,*256col*:colors=256,xterm*:XT:Ms=\E]52;%p1%s;%p2%s\007,rxvt-uni*:XT:Ms=\E]52;%p1%s;%p2%s\007'
```

emacs配置
```elisp
(defun osc52-select-text (string &optional replace yank-handler)
  (let ((b64-length (+ (* (length string) 3) 2)))
    (if (<= b64-length 100000)
        (send-string-to-terminal
         (concat "\e]52;c;"
                 (base64-encode-string (encode-coding-string string 'utf-8) t)
                 "\07"))
        (message "Selection too long to send to terminal %d" b64-length)
        (sit-for 2))))
(defun osc52-set-cut-function ()
  (if (not window-system) (setq interprogram-cut-function 'osc52-select-text)))
(osc52-set-cut-function)
```

### 工作台高可用

1. tmux部分，使用[tmux-continuum](https://github.com/tmux-plugins/tmux-continuum)插件。
2. emacs部分，使用[spacemacs](https://github.com/syl20bnr/spacemacs)的persistent layout插件。
3. HOME目录做RAID10。

以上是研发工作台的介绍。之后我将分享一些基于该工作台下的开发技巧，包括tmux，emacs以及客户机的窗口管理。

写在最后
===

人体工程学开发是我3年前开始尝试的，起因是右肩使用鼠标永久劳损，导致手臂无法正常使用鼠标，以及颈椎的病态反应，都不断地警醒我，这个职业有一个健康的开发环境是多么的重要。在这几年里，我从硬件到软件进行了非常多的尝试，有成功的，身边的朋友也借鉴参考了不少，当然也有失败的。这个过程耗费了很大的精力，有时候一个解决方案需要研究几天才能有答案。这个系列我分享出来，是真心希望里面有一部分内容能帮到在这方面同样有追求的朋友们，我每天都受益于这个方案，无论是工作效率还是身体机能方面。朋友们如果有自己的人体工程学心得，欢迎留言交流。
