---
title: zsh不存储失败命令
date: '2023-02-16T00:00:00+08:00'
url: /post/bJEPQlrpl/
tags:
- linux
feature: https://learnubuntu.com/content/images/2022/09/zsh-ubuntu.png
hidden: true
---

使用 [oh-my-zsh](https://ohmyz.sh/) 的朋友经常会用到 [zsh-autosuggestions](https://github.com/tarruda/zsh-autosuggestions) 这个插件，它可以在你开始敲命令的时候，自动根据输入过的历史命令提供建议。但作为普通个人用户，经常尝试命令、经常出错。

比如我想要输入 `git commit`，偶尔错误输入 `git comit`，这个自动建议就会反复出现，反而降低了效率。

于是我想有没有办法能够让 zsh 不记录失败了的命令。我在[这里](https://superuser.com/questions/902241/how-to-make-zsh-not-store-failed-command)找到了答案。

你可以在 zsh 的配置文件中增加以下命令，能够在单行单个命令的简单情形中不记录失败命令。可以是这些配置文件 `~/.zshrc` 或 `~/.oh-my-zsh/oh-my-zsh.sh` 或 `~/.oh-my-zsh/custom/<yourcustom>.zsh`。

```
zshaddhistory() { whence ${${(z)1}[1]} >| /dev/null || return 1 }
```

修改配置文件后：

```
source ~/.zshrc
```

单行单个命令是指诸如 `git comit`，而非 `echo foo; echooo bar`。需要注意的是，通过 `↑` 方向键你还是可以看到失败的命令。