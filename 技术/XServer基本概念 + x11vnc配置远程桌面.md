> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://blog.csdn.net/lovewangtaotao/article/details/102907540

前言
--

读研发论文难啊，之前所有的 blog 都写在了自己的笔记本上，因为觉的写的太好了浪费时间，自己可以看懂就够了，但是白岩松老师的 "机会大多数取决于别人背后怎么评价你" 和雄子的 “将复杂的问题直观的阐述思想” 还是让我有了很大的触动。所以以后我会多更新 blog，并且尽力将问题讲解透彻。

问题起源
----

Xserver 可以说是第二次碰到了，之前玩 Ubuntu 的时候就是各种问题，但是每次都是以重装系统为结果。这次准备装 x11vnc 来捣鼓远程桌面的时候出现了问题，问题挺简单，也挺常见，也就是 display 变量和 xauthority 文件之类的。但是最关键的是 loggin greeting 界面连接不上，所以就花了一下午时间搞明白了，也解决了问题。

abstract
--------

这篇 blog 解决以下几个问题

1.  Xserver 中一些经典概念
2.  x11vnc 如何配置 - display -auth 参数 (见下文 Authority Process)
3.  gdm3 和 wayland 以及 greeting 界面下如何 x11vnc 配置 (no one logs in) (见下文 display manager + gdm3)

Solution to Abstract
--------------------

首先，Xserver 中有很多概念，这里只给出常见的: X 架构，Xserver，Xclient，Xsession，Xauthority file，MIT-MAGIC-COOKIES，display number，  
**X architecture** : 所谓的 X 架构是一个开源的图像绘制架构，一般的 Linux 操作系统都是靠这个来实现 windows 下的桌面的。X 架构是一个 C/S 架构的软件，为 Xserver 和 Xclient。主要的团队好像是 Xorg，一般 linux 下 X11 或者 Xorg 啥的就是和这个相关的软件或者配置文件。

**Xserver**: 作为 X 架构的中心，Xserver 负责处理处理输入，将输入传送给 XClient，接受 XClient 的绘制请求并且绘图。和 Windows 不一样，Xserver 的绘制服务器可以有多个！！每个对应了一个 Display，这点很关键。

**Xclient** : 简单的讲，就是我们的每一个 Application。例如，gnome-terminal， firefox 等。这些程序包含了具体的绘制代码。学习过 windows 窗口机制就知道，application 绘制代码其实是对不同的 Event 向 Xserver 发送绘制操作，具体绘制的过程是靠 Xserver 实现的。这样的好处是 Xserver 可以给每个 Xclient 维护有效区，app 端就不要考虑这些重叠啥的东西了。

**Display**：每个 Xserver 和一个 display 标志相对应。display 是 $local_host:$display_num.$sceen_num 的格式。例如 ":0.0" , "127.0.0.1:1.0"  
都是合法的。具体 man x 。然后有个 DISPLAY 栏目专门讲的格式。

**Connection Process** 连接和安全是 C/S 架构很关键的一点。这里讲解具体的过程:  
首先，要想运行一个 XClient 程序，必须要启动一个 Xserver，这个过程可以使用多个方法，例如 startx，或者是一个 display manager(下面会讲)。启动之后会有一个 display number，如果你的 Server 是第一个那就是: 0 第二个就是 :1  
然后，XClient 只是负责绘制的，所以需要连接到 Xserver 才可以实现图形的绘制，所以我们需要给 XClient 提供，因此 $DISPLAY 变量作为 XServer 的标识符需要被提供，但是为了解决安全问题，需要一个验证机制 XSecurity（后面 Authority Process 有具体过程），因此为了连接成功，还需要提供要给 xauth 文件（后面 Authority Process 会讲具体如何寻找个文件），这个文件的位置通过 $XAUTHORITY 传递，然后 XClient 通过一定解析，发送给 XServer，如果验证通过，就相当于是通过了验证，这个 XClient 就可以绘图了。如果没有，那么就会有一些无法连接的错误之类的。

**Authority Process** 验证过程有很多个步骤，具体可以看 man xsecurity 。这里讲解一下什么是 MIT-MAGIC-COOKIES 验证和 host 验证。首先 xhost 可以添加和删除信任列表。列表如果是空，那么全都可以信任，否则只有在列表上的才可以进行连接。这个方法只可以控制 host 阶段的一些验证问题，同一个机器的的不同程序就无法判断了，所以有后面的 COOKIES 验证机制。COOKIES 机制是最为关键的，因为很多都是使用这个方法。原理很简单，那就是在每个 XServer 中都有一个可接受的 COOKIES，是个特定格式的字符串。然后每个连接都要附带这个信息，如果存在，那么就可以，否则失败。 XServer 中 Cookies 是在启动的时候指定的一个 xauthority 文件。 X -auth=XXX 中的 auth 参数就是文件，这个文件一般是 Display Manager（后面会讲）生成的，然后由 DM 管理。所以需要得到正确的 COOKIES 需要知道启动 X 的时候 - auth 指定的是哪个 xauthority 文件，这个文件就是包含了 COOKIES 的文件，要是想要做实验的，可以通过 xauth -f FILE_NAME 然后 list 命令来看到具体的 cookies 是什么。 同时我们可以通过 ps aux | grep Xorg 来看到服务器启动时候的命令，来找到最关键的 -auth 文件位置，有了上面的知识，基本上就没问题了，都可以完美运行 x11vnc 或者别的 XClient 了。

**Display Manager** 因为一个主机可有多个 Display，也就是多个 XServer，一般是一个 User 一个，所以有的软件负责管理 Xserver。这些软件就是 Display Manager(DM)，这些软件的主要作用就是：greetting 界面来问候，请求输入密码和账号，然后验证，成功之后启动一个 XServer，并且生成和管理 xauthority 文件，并且设置 $DISPLAY $XAUTHORITY 环境变量。主要目的就是将这些过程对用户透明化。这也是为什么很多人不需要知道这些细节的原因。我们这里关心的就是 Display Manager 是如何生成 xauthority 文件的（因为我们需要，但是其实通过 ps 的方法可以直接找出来，所以也不是很关键。）。这些软件的例子有很多，例如 gdm gdm3 xdm lightdm 都是，而且他们有不同的行为，所以当你发现你的问题在这里的话，一定要记得去看文档，而不是一味的谷歌，文档才是最核心的东西。

**GDM3** 我使用的是 GDM3，所以这里我来解析一下 GDM3 的一些坑点。首先文档我只找到了 GDM 的，而且很不一样。其中一个还可以的是 man gdm3 有一些资料。所以通过一些探索，这里记录一下没有的东西。 最关键的一点就是，GDM3 的登陆界面，可能没有 Xserver。但是 GDM3 确实是图形界面啊，是的，它使用了一个新的 wayland 图形架构，具体可以维基百科一下，简单来说，这个东西的作者和 X 架构是同一个人，然后他为了改良 X 中的一些低效的环节，重新设计了这个架构，这个架构更加简单，但是还在开发测试阶段。所以在 greeting 阶段是没有 Xserver 的，所以 x11vnc 是不可以连接的，也就是当电脑没有用户登陆的时候，不可以通过 x11vnc 远程桌面，一定会显示 can’t open display :0 等的错误。所以需要在 /etc/gdm3/custom.conf 中 uncomment 掉一个 wayland 相关的参数（会有英文提示），然后就可以了。但是注意这样的话，greetting 的 display number 为 :0 ，你登陆之后启动的用户的 Xserver 是 display number 2。所以需要建立两个 x11vnc 监听。

**x11vnc 总体过程**: 先注释掉 wayland，重启，在 greeting 界面的时候，通过远程登陆 / tty / 登陆之后 ps aux | grep -i xorg 看 auth 是哪个 (如果登陆了，有两个，一个是 greet 的，一个是用户的，一般是 1000 是用户，121 是登陆界面的 auth 文件)，然后：

```
x11vnc -auth 上面的file -display :0 #greetting用户
x11vnc -auth 上面的file -display :1 #用户
```

**Xserver 的 port 映射** Xserver 对应的 port（如果开启了 TCP，可以在 gdm 的配置文件中开启，具体看 gdm 文档和 gdm3 节）。6000+display number 是监听端口，可使用 netstat -nap + 来看

_**reference:**_ linux man page
