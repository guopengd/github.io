---
layout: _posts
categories: linux
tags: [shell,ssh]
title: github配置ssh秘钥报错Permission denied (publickey)
date: 2019-06-16 00:34:31
---
## SSH报错Permission denied (publickey)
以前一直使用的Ubuntu系统，所以使用配置ssh秘钥连接github一直没有出现什么问题。但是今天偶然在Windows系统下给github配置一个秘钥，令人惊奇的事情发生了。github一直报错Permission denied (publickey)，我仔细的检查了下我的.ssh目录，我的秘钥赫然躺在文件夹中。  
![ssh文件夹](https://i.loli.net/2019/06/16/5d0518fe41dcd69999.png)
当时我的内心是崩溃的，what the f**k，果然还是Linux大法好，什么破Windows，净出些幺蛾子。但是问题出来了，总是要去解决吧.于是Github搜了一圈下来，发现[github帮助](https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#adding-your-ssh-key-to-the-ssh-agent)里面有一篇关于这个描述。按照以下步骤即可解决报错。
```bash
$ eval $(ssh-agent -s)
# 也可以换成下面的命令,两个效果一样
# $ ssh-agent $SHELL

$ ssh-add ~/.ssh/y
Enter passphrase for /c/Users/Administrator/.ssh/y:
Identity added: /c/Users/Administrator/.ssh/y (648276765@qq.com)

$ ssh -T git@github.com
Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
```
但是，我们怎么能只满足于怎么解决呢，于是我就开始了研究之路，通过搜索看了不少博客，了解了大致的原因。

### SSH agent
{% blockquote @faner https://www.jianshu.com/p/1246cfdbe460 %}
ssh-agent命令是一种控制用来保存公钥身份验证所使用的私钥的程序。在Linux中ssh-agent 在X会话或登录会话之初启动，所有其他窗口或程序则以客户端程序的身份启动并加入到ssh-agent程序中。通过使用环境变量，可定位代理并在登录到其他使用ssh机器上时使用代理自动进行身份验证。
其实ssh-agent就是一个密钥管理器，运行ssh-agent以后，使用ssh-add命令将私钥交给ssh-agent保管，其他程序需要身份验证的时候可以将验证申请交给ssh-agent来完成整个认证过程。
另外如果您的私钥使用密码短语来加密了的话，每一次使用 SSH 密钥对进行登录的时候，您都必须输入正确的密码短语。而 SSH agent 程序能够将您的已解密的私钥缓存起来，在需要的时候提供给您的 SSH 客户端。这样子，您就只需要将私钥加入 SSH agent 缓存的时候输入一次密码短语就可以了。这为您经常使用 SSH 连接提供了不少便利。
{% endblockquote %}
通过上面的解释，我们知道我在Ubuntu系统中我们开机就能自动启动ssh-agent，所以我们无需启动ssh-agent就能能直接使用ssh来连接我的github了。但是ssh-agent是无法运行在Windows中的，所以我们需要用git的bash来启动我们的ssh-agent服务，这样才能管理我们秘钥进行ssh操作。同时，通过这个解释，我们还能发现一个很绝望的事情，那就是你每次重新打开一个bash就要再次输入上面的命令才能连接ssh。

### ssh-agent $SHELL和eval $(ssh-agent -s)的区别
* ssh-agent $SHELL
这种方式会在当前shell中启动一个默认shell作为当前shell的子shell，ssh-agent程序会在子shell中运行。$SHELL变量名代表系统的默认shell，如果自己知道系统使用的是哪一种shell也可以直接指定，如ssh-agent bash,ssh-agent csh.退出当前子shell使用exit。
* eval $(ssh-agent -s)
 eval $(ssh-agent -s)本质上等同于上一个命令，ssh-agent -s只会输出你需要连接的环境变量，因此我们需要加上eval来将这些环境变量加载进来。至于为什么不设置ssh-agent命令本身将环境变量加载进来呢？那是因为在Unix中，进程只能修改自己的环境变量，并将它们传递给子进程。它无法修改其父进程的环境，因为系统不允许它。这是非常基本的安全设计。
 
所以，在这两个命令里面大家选一个自己看着顺眼的用就好，虽然解决了使用ssh连接我的github，但是每次都要这么繁琐的步骤。综上，我选择图形化界面。
 
 那么问题来了，我搞这么就弄清楚这些都是为了啥！！！开个玩笑啦，学无止境，其实还是不亏的，至少又学习到了一点新的知识呢。