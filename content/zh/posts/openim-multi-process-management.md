---
title: '我是如何优雅设计 OpenIM 多进程管理策略的'
description: '本文详述了在 OpenIM 中实施多进程管理的策略和技术，OpenIM 的 System 部署设计； 探讨如何通过进程隔离和资源控制来提升系统稳定性和扩展性。'
ShowRssButtonInSectionTermList: true
date: 2023-08-16T15:55:39+08:00
draft: false
showtoc: true
tocopen: true
type: posts
author: '熊鑫伟，我'
keywords: ['OpenIM', '多进程管理', '系统稳定性', '资源控制', '系统扩展']
tags: ['OpenIM', '多进程 (Multi-Processing)', '性能优化 (Performance Optimization)', '系统稳定性 (System Stability)', '资源管理 (Resource Management)']
categories: ['开发 (Development)']
---

## 主要模块

这篇文章将会从 OpenIM 最基本的单进程前台运行开始，到 nohup 后台运行，到 system 系统进程，然后再到 Supervisord，容器化管理，kubernetes 健康检测。


## 目前问题

阅读：https://github.com/cubxxw/Open-IM-Server/blob/refactor/feat-enhance/scripts/install/openim-crontask.sh 源码

目前 OpenIM 之前的进程管理策略是通过 `nohup` 在后台启动的。

整个项目由多个进程共同运行，现在需要一个可靠的保活机制，以便能够在进程崩溃的时候能够快速把它拉起来。openim 的解决方案无非就是写个保活脚本，在后台一直运行，如果发现某进程被关闭了，那么由脚本拉起来，或者是通过  docker compose 健康检测机制：

```bash
healthcheck:
  test: ["CMD-SHELL", "./scripts/check-all.sh"]
  interval: 30s
  timeout: 10s
  retries: 5
```

但是脚本它自己挂掉怎么办？(总不能使用脚本继续保活保活脚本套娃吧)。此外，另一个办法就是配置出来一个服务，让`Linux`操作系统帮你守护进程，显然，这种办法完全不需要担心守护进程自己挂掉，毕竟是`systemd`帮你守护，如果它挂掉了，操作系统应该也没了。

我们适配通用的 Ubuntu 和 CentOS 即可，因为 其他 的操作系统比如说 `alpine` , Alpine Linux 并不使用 `systemd` 作为其默认的初始化系统。相反，Alpine Linux 使用 `OpenRC` 作为其默认的初始化系统。这也是许多人选择 Alpine Linux 的原因，因为它是轻量级的，并且没有引入 `systemd`。

现在新版的 Ubuntu 和 CentOS 都支持的，旧版的`linux`使用`service httpd start`启动服务，新版的`linux`使用`systemctl start httpd`来启动服务，此外使用`initd`作为初始化系统的操作系统添加服务是在`/etc/init.d/`中添加脚本，而使用`systemd`作为初始化系统的操作系统只需要在`/etc/systemd/system/`文件夹中添加配置文件就好了。



## 前台进程

前台进程：是在终端中运行的命令，那么该终端就为进程的控制终端，一旦这个终端关闭，这个进程也随之消失，这时就把Shell给占据了，我们无法进行其他操作。对于那些没有交互的进程，我们希望将其在后台启动，可以在启动参数的时候加一个’&’实现这个目的。

后台进程：也叫守护进程（Daemon），是运行在后台的一种特殊进程，不受终端控制，它不需要终端的交互；Linux的大多数服务器就是使用守护进程实现的。比如Web服务器的httpd等。


### 解决方案

**1. 使用&后台运行程序：**

结果会输出到终端

+ 使用Ctrl + C发送SIGINT信号，程序**免疫**

+ 关闭session发送SIGHUP信号，程序**关闭**

**2. 使用nohup运行程序：**

结果默认会输出到nohup.out

+ 使用Ctrl + C发送SIGINT信号，程序**关闭**

+ 关闭session发送SIGHUP信号，程序**免疫**

因此，平日线上经常使用nohup和&配合来启动程序：**可以同时免疫SIGINT和SIGHUP信号**

**3. Systemctl：**

Systemctl是一个systemd工具，主要负责控制systemd系统和服务管理器。

在终端中输入 `ps ax | grep systemd`，看到第一行，其中的数字 1 表示它的进程号是1，也就是说它是 Linux 内核发起的第一个程序。因此，内核一旦检测完硬件并组织好了内存，就会运行 `/usr/lib/systemd/systemd` 可执行程序，这个程序会按顺序依次发起其他程序。（ 在还没有 Systemd 的日子里，内核会去运行 `/sbin/init`，随后这个程序会在名为 SysVinit 的系统中运行其余的各种启动脚本。）



## System

详细讲解一下 Linux 很重要的部分 system

+ 参考文章：https://blog.51cto.com/babylater/1895056

### 单元

系统初始化需要做的事情非常多。需要启动后台服务，比如启动 SSHD 服务；需要做配置工作，比如挂载文件系统。这个过程中的每一步都被 systemd 抽象为一个配置单元，即 unit。可以认为一个服务是一个配置单元；一个挂载点是一个配置单元；一个交换分区的配置是一个配置单元；等等。systemd 将配置单元归纳为以下一些不同的类型。然而，systemd 正在快速发展，新功能不断增加。所以配置单元类型可能在不久的将来继续增加。

1. service：代表一个后台服务进程，比如 mysqld。这是最常用的一类。

2. socket：此类配置单元封装系统和互联网中的一个 套接字 。当下，systemd 支持流式、数据报和连续包的 AF_INET、AF_INET6、AF_UNIX socket 。每一个套接字配置单元都有一个相应的服务配置单元 。相应的服务在第一个"连接"进入套接字时就会启动(例如：nscd.socket 在有新连接后便启动 nscd.service)。

3. device：此类配置单元封装一个存在于 Linux 设备树中的设备。每一个使用 udev 规则标记的设备都将会在 systemd 中作为一个设备配置单元出现。

4. mount：此类配置单元封装文件系统结构层次中的一个挂载点。Systemd 将对这个挂载点进行监控和管理。比如可以在启动时自动将其挂载；可以在某些条件下自动卸载。Systemd 会将 `/etc/fstab` 中的条目都转换为挂载点，并在开机时处理。

5. automount：此类配置单元封装系统结构层次中的一个自挂载点。每一个自挂载配置单元对应一个挂载配置单元 ，当该自动挂载点被访问时，systemd 执行挂载点中定义的挂载行为。

6. swap: 和挂载配置单元类似，交换配置单元用来管理交换分区。用户可以用交换配置单元来定义系统中的交换分区，可以让这些交换分区在启动时被激活。

7. target：此类配置单元为其他配置单元进行逻辑分组。它们本身实际上并不做什么，只是引用其他配置单元而已。这样便可以对配置单元做一个统一的控制。这样就可以实现大家都已经非常熟悉的运行级别概念。比如想让系统进入图形化模式，需要运行许多服务和配置命令，这些操作都由一个个的配置单元表示，将所有这些配置单元组合为一个目标(target)，就表示需要将这些配置单元全部执行一遍以便进入目标所代表的系统运行状态。 (例如：multi-user.target 相当于在传统使用 SysV 的系统中运行级别 5)

8. timer：定时器配置单元用来定时触发用户定义的操作，这类配置单元取代了 atd、crond 等传统的定时服务。


每个配置单元都有一个对应的配置文件，系统管理员的任务就是编写和维护这些不同的配置文件，比如一个 MySQL 服务对应一个 `mysql.service` 文件。这种配置文件的语法非常简单，用户不需要再编写和维护复杂的系统 5 脚本了。



### 依赖关系

虽然 systemd 将大量的启动工作解除了依赖，使得它们可以并发启动。但还是存在有些任务，它们之间存在天生的依赖，不能用"套接字激活"(socket activation)、D-Bus activation 和 autofs 三大方法来解除依赖（三大方法详情见后续描述）。比如：挂载必须等待挂载点在文件系统中被创建；挂载也必须等待相应的物理设备就绪。为了解决这类依赖问题，systemd 的配置单元之间可以彼此定义依赖关系。

Systemd 用配置单元定义文件中的关键字来描述配置单元之间的依赖关系。比如：unit A 依赖 unit B，可以在 unit B 的定义中用"require A"来表示。这样 systemd 就会保证先启动 A 再启动 B。



### Systemd 的并发启动原理

如前所述，在 Systemd 中，所有的服务都并发启动，比如 Avahi、D-Bus、livirtd、X11、HAL 可以同时启动。乍一看，这似乎有点儿问题，比如 Avahi 需要 syslog 的服务，Avahi 和 syslog 同时启动，假设 Avahi 的启动比较快，所以 syslog 还没有准备好，可是 Avahi 又需要记录日志，这岂不是会出现问题？



## Systemd  的使用

开发人员需要了解 systemd 的更多细节。比如你打算开发一个新的系统服务，就必须了解如何让这个服务能够被 systemd 管理。这需要你注意以下这些要点：

1. 后台服务进程代码不需要执行两次派生来实现后台精灵进程，只需要实现服务本身的主循环即可。

2. 不要调用 setsid()，交给 systemd 处理

3. 不再需要维护 pid 文件。

4. Systemd 提供了日志功能，服务进程只需要输出到 stderr 即可，无需使用 syslog。

5. 处理信号 SIGTERM，这个信号的唯一正确作用就是停止当前服务，不要做其他的事情。

6. SIGHUP 信号的作用是重启服务。

7. 需要套接字的服务，不要自己创建套接字，让 systemd 传入套接字。

8. 使用 sd_notify()函数通知 systemd 服务自己的状态改变。一般地，当服务初始化结束，进入服务就绪状态时，可以调用它。



### Unit 文件的编写

对于开发者来说，工作量最大的部分应该是编写配置单元文件，定义所需要的单元。

举例来说，开发人员开发了一个新的服务程序，比如 httpd，就需要为其编写一个配置单元文件以便该服务可以被 systemd 管理，类似 UpStart 的工作配置文件。在该文件中定义服务启动的命令行语法，以及和其他服务的依赖关系等。

此外我们之前已经了解到，systemd 的功能繁多，不仅用来管理服务，还可以管理挂载点，定义定时任务等。这些工作都是由编辑相应的配置单元文件完成的。我在这里给出几个配置单元文件的例子。

**服务的配置单元文件，服务配置单元文件以 `.service` 为文件名后缀。**

openim 也写了关于自己的服务的一些配置文件 ，比如说 `openim-api.service`



## Supervisord

Supervisord 是用 Python 实现的一款的进程管理工具，supervisord 要求管理的程序是非 daemon 程序，supervisord 会帮你把它转成 daemon 程序，因此如果用 supervisord 来管理进程，进程需要以非daemon的方式启动。

例如：管理nginx 的话，必须在 nginx 的配置文件里添加一行设置 daemon off 让 nginx 以非 daemon 方式启动

安装：

```bash
apt install supervisor || yum install supervisor
```

安装后，可以修改一下最后一行：

```bash
files = /etc/supervisor/conf.d/*.conf
```

换到 ` /opt/supervisord.d/` 目录下：

```bash
files = /opt/supervisord.d/*.ini   
```

**上述地址可以自定义, 会读取/opt/supervisord.d 文件夹下 所有以 ini结尾的文件 作为配置读取**

**然后再/opt/supervisord.d/ 下新建一个配置 例如 test.ini**

```ini
;进程名称 即项目名
[program:test]

;脚本目录 运行的进程文件目录
directory=/opt/ytgMateriel/materialBackend

;启动命令 此处为java的jar 启动命令
command=/opt/jdk1.8.0_171/bin/java -Xms512m -Xmx1024m -jar -Dspring.profiles.active=prd -Djava.io.tmpdir=./tmp -Dloader.path=lib ytg-material-backend.jar

;停止进程的命令 默认 quit
stopsignal=KILL

;supervisor启动的时候是否随着同时启动，默认True   
;当程序exit的时候，这个program不会自动重启,默认unexpected，设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected和true。如果为false的时候，无论什么情况下，都不会被重新启动，如果为unexpected，只有当进程的退出码不在下面的exitcodes里面定义的 
autorestart=true

;这个选项是子进程启动多少秒之后，此时状态如果是running，则我们认为启动成功了。默认值为1
startsecs=3

;日志
stdout_logfile=/opt/ytgMateriel/materialBackend/logs/ytg-material-backend.log

;stdout日志文件大小，默认 50MB
stdout_logfile_maxbytes=100MB

;stdout日志文件备份数
stdout_logfile_backups=50
user = root

;把stderr重定向到stdout，默认 false
redirect_stderr=true
```



**启动supervisord**

使用supervisor进程管理命令之前先启动 `supervisord`，否则程序报错。

```bash
supervisord -c /etc/supervisord.conf
```

常用命令

```cpp
supervisorctl status        //查看所有进程的状态
supervisorctl stop 进程名       //停止es
supervisorctl start 进程名      //启动es
supervisorctl restart 进程名      //重启es
supervisorctl update        //配置文件修改后使用该命令加载新的配置
supervisorctl reload        //重新启动配置中的所有程序
```



## Systemd与Supervisord对比

无论是Systemd和Supervisord都可以用于控制进程，实现进程的分组管理和崩溃自动重启.

无论 Supervisord 还是 Systemd，都采用 ini 作为配置文件的格式。跟 Supervisord 不同的是，Systemd 每个程序都要单独开一个 unit 文件。

Supervisord 可以同时启动/停止配置文件中所以的进程（或者某个进程组配置中的进程）。

也就是说，Supervisord 使用“进程组”这个概念来控制多个进程, Systemd 则用用依赖来实现这一点。

我们来看一个简单的Systemd配置文件，实现了进程组管理

```ini
; group.target
[Unit]
Description=Application
Wants=prog1.service prog2.service
```

第一个：

```ini
; prog1.service
[Unit]
Description=prog1
PartOf=group.target

[Service]
ExecStart=/usr/bin/prog1
Restart=on-failure
```

第二个：

```ini
; prog2.service
[Unit]
Description=prog2
PartOf=group.target

[Service]
ExecStart=/usr/bin/prog2
Restart=on-failure
```

`systemctl start group.target` ，prog1 和 prog2 也会启动。`systemctl restart group.target`，prog1 和 prog2 也会跟着重启



相比起来Supervisord的管理方式就要明了一些了:

```ini
[program:prog1]
command=python /home/user/myapp/prog1.py

[program:prog2]
command=python /home/user/myapp/prog2.py

[group:prog]
programs=prog1,prog2
```

`supervisorctl start prog:*`可以启动prog组下的所有进程

不过 supervisord 有一个优点。如果你不知道哪些程序的配置改变了，简单地执行 `supervisorctl update`，所有涉及的进程都会被重启。

supervisor与[launchd](https://en.wikipedia.org/wiki/Launchd)，[daemontools](http://cr.yp.to/daemontools.html)，[runit](http://smarden.org/runit/)等程序有着相同的功能，与某些程序不同的是，它并不作为“id 为 1的进程”而替代init。相反，它用于控制应用程序，像启动其它程序一样，通俗理解就是，把Supervisor服务管理的进程程序，它们作为supervisor的子进程来运行，而supervisor是父进程。supervisor来监控管理子进程的启动关闭和异常退出后的自动启动。



### 多进程模板

对于 `Supervisord` 来说，只需要维护一个配置文件而已，而对于 `systemd` 来说，需要维护很多个配置文件，配置项也比较复杂。

如果需要创建很多服务，但是服务的配置文件只有 `ExecStart` 项有细微区别，那么可以考虑使用模板功能。

比如，创建服务文件`/etc/systemd/system/ping@.service`

```ini
[Unit]
Description=Ping service %i

[Service]
Type=simple
ExecStart=/usr/bin/ping %i

[Install]
WantedBy=multi-user.target
```

启动进程可以这样`systemctl start ping@127.0.0.1.service`，实际上，启动服务可以省略`.service`后缀，也即是`systemctl start ping@127.0.0.1`，如果要一次启动多个服务，可以`systemctl start ping@127.0.0.1 ping@127.0.0.2`，停止服务也是类似。

这样，如果想要`ping`别的地址，只需要修改命令中`@`之后的字符串就好了。

也支持如下这样拼接字符串

```ini
[Unit]
Description=Ping service %i

[Service]
Type=simple
ExecStart=/usr/bin/ping 127.0.0.%i

[Install]
WantedBy=multi-user.target
```

`systemctl start ping@1`就能执行`ping 127.0.0.1`服务

对于配置文件中的`%i`其实是有大小写区别的，`%i`是转义之后的字符串 `%I`是不转义的字符串，对于完整的说明符列表，见[systemd.unit (www.freedesktop.org)](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)



### 链式启动（服务依赖）

有这么一种情况，需要同时启动多个服务，并且他们有启动顺序的限制。那么可以像下面这么配置

假设有 `A进程`、 `B进程`、 `C进程`，想要按顺序依次启动，那么可以这么配置

```bash
# cat /etc/systemd/system/C.service
[Unit]
Description=C Process
Requires=B.service
After=B.service

[Service]
Type=simple
ExecStart=/export/CProgram

[Install]
WantedBy=multi-user.target
```

对于 B 服务

```ini
# cat /etc/systemd/system/B.service
[Unit]
Description=B Process
Requires=A.service
After=A.service

[Service]
Type=simple
ExecStart=/export/BProgram

[Install]
WantedBy=multi-user.target
```

对于 A 服务

```ini
# cat /etc/systemd/system/A.service
[Unit]
Description=A Process

[Service]
Type=simple
ExecStart=/export/AProgram

[Install]
WantedBy=multi-user.target
```

效果是，`systemctl start C.service`之后几个进程会依次启动，`Requires`指定了几个服务之间的依赖关系，因为通过`After`选择指定了服务间启动顺序，所以几个服务是依次启动的。如果没有`After`，**启动顺序不被保证**

如果，此时`systemctl stop A.service`，那么几个服务都会被关闭，因为`Requires`要求了前置服务必须存在，否则自身也不应该启动。如果不想自身服务被关闭，那么可以把`Requires`(要求)替换成`Wants`(想要)。

如果要形成依赖链，除了`After`也可以使用`Before`



### 查看服务输出 - journalctl

`systemd`不仅用来运行服务，它同时也有日志服务，用于取代老系统的`syslog`。

运行的服务标准输出和错误输出会被交给`journald`管理，查看某个服务可以使用这样的命令`journalctl -u ping@1` 带`-e`参数可以跳到最新一行 `-f`参数可以看到实时输出，`-n`参数可以指定输出的行数，`-r`反序输出。

例如`journalctl -u ping@1 -e` 或者 `journalctl -u ping@1 -f`

如果服务的输出太多，那么可以在`.servive`文件中的`[Unit]`节区配置`StandardOutput=null`

也可以通过`systemctl status xx.service`查看服务的部分输出

```bash
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```





## OpenIM  配置 System 

在上面的多重考虑中， OpenIM 最终选择的还是 System ，因为我们已经有一套后台的方案。

首先，考虑将各个配置模板化，这是我们第一步要做的。

使用模板，一个模板单元(unit)文件可以创建多个实例化的单元文件，从而简化系统配置。

模板单元文件的文件名中包含一个@符号，@位于单元基本文件名和扩展名之间，比如:

```
example@.service
```

当从模板单元文件创建实例单元文件时，在@符号和单元扩展名(包括符号.)之前加入实例名,比如：

```bash
example@instance1.service
```

表明实例单元文件[example@instance1.service](https://mail.google.com/mail/?view=cm&fs=1&tf=1&to=example@instance1.service)实例化自模板单元文件example@.service，其实例名为instance1

实例单元文件一般是模板单元文件的一个符号链接，符号链接命中包含实例名，systemd就会传递实例名给模板单元文件。

在相应的target中创建实例单元文件符号链接之后，需要运行一下命令将其装载：

```bash
$ sudo systemctl daemon-reload
```

**模板标识符/参数**

模板单元文件中可以使用一些标识符，当被实例化为实例单元文件并运行时，systemd会将标识符的实际值传递给对应的标识符，比如在模板单元文件中是用`%i`，实际运行实例单元文件时，会将实例名传递给 `%i` 标识符。(中文意思就是`@`之后的字符，`.service`之前的字符)

| 占位符 | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| %n     | 在模板文件中出现时，会插入完整的单元名称。                   |
| %N     | 与上面相同，但会反转任何转义，例如在文件路径模式中存在的转义。 |
| %p     | 这引用单元名称前缀。这是单元名称中 @ 符号之前的部分。        |
| %P     | 与上面相同，但会反转任何转义。                               |
| %i     | 这引用实例名称，即在实例单元中 @ 之后的标识符。这是最常用的指定符之一，因为它保证是动态的。使用此标识符鼓励使用配置重要的标识符。例如，服务运行的端口可以用作实例标识符，模板可以使用此指定符来设置端口规范。 |
| %I     | 这个指定符与上面相同，但会反转任何转义。                     |
| %f     | 这将被替换为未转义的实例名称或前缀名称，前面加上一个 /。     |
| %c     | 这将指示单元的控制组，移除了标准的父层次结构 `/sys/fs/cgroup/ssytemd/`。 |
| %u     | 配置为运行单元的用户的名称。                                 |
| %U     | 与上面相同，但作为数字 UID 而不是名称。                      |
| %H     | 运行该单元的系统的主机名称。                                 |
| %%     | 用于插入文字的百分比符号。                                   |



## 实现

以 `openim-crontask` 为例：

```bash
#!/usr/bin/env bash

# Copyright © 2023 OpenIM. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# 
# OpenIM CronTask Control Script
# 
# Description:
# This script provides a control interface for the OpenIM CronTask service within a Linux environment. It supports two installation methods: installation via function calls to systemctl, and direct installation through background processes.
# 
# Features:
# 1. Robust error handling leveraging Bash built-ins such as 'errexit', 'nounset', and 'pipefail'.
# 2. Capability to source common utility functions and configurations, ensuring environmental consistency.
# 3. Comprehensive logging tools, offering clear operational insights.
# 4. Support for creating, managing, and interacting with Linux systemd services.
# 5. Mechanisms to verify the successful running of the service.
#
# Usage:
# 1. Direct Script Execution:
#    This will start the OpenIM CronTask directly through a background process.
#    Example: ./openim-crontask.sh openim::crontask::start
# 
# 2. Controlling through Functions for systemctl operations:
#    Specific operations like installation, uninstallation, and status check can be executed by passing the respective function name as an argument to the script.
#    Example: ./openim-crontask.sh openim::crontask::install
# 
# Note: Ensure that the appropriate permissions and environmental variables are set prior to script execution.
# 

OPENIM_ROOT=$(cd "$(dirname "${BASH_SOURCE[0]}")"/../.. && pwd -P)
[[ -z ${COMMON_SOURCED} ]] && source "${OPENIM_ROOT}"/scripts/install/common.sh

SERVER_NAME="openim-crontask"

function openim::crontask::start()
{
    openim::log::info "Start OpenIM Cron, binary root: ${SERVER_NAME}"
    openim::log::status "Start OpenIM Cron, path: ${OPENIM_CRONTASK_BINARY}"

    openim::util::stop_services_with_name ${SERVER_NAME}

    openim::log::status "start cron_task process, path: ${OPENIM_CRONTASK_BINARY}"
    nohup ${OPENIM_CRONTASK_BINARY} >> ${LOG_FILE} 2>&1 &
    openim::util::check_process_names ${SERVER_NAME}
}

###################################### Linux Systemd ######################################
SYSTEM_FILE_PATH="/etc/systemd/system/${SERVER_NAME}.service"

# Print the necessary information after installation
function openim::crontask::info() {
cat << EOF
openim-crontask listen on: ${OPENIM_CRONTASK_HOST}
EOF
}

# install openim-crontask
function openim::crontask::install()
{
  pushd "${OPENIM_ROOT}"

  # 1. Build openim-crontask
  make build BINS=${SERVER_NAME}
  openim::common::sudo "cp ${OPENIM_OUTPUT_HOSTBIN}/${SERVER_NAME} ${OPENIM_INSTALL_DIR}/bin"

  openim::log::status "${SERVER_NAME} binary: ${OPENIM_INSTALL_DIR}/bin/${SERVER_NAME}"

  # 2. Generate and install the openim-crontask configuration file (openim-crontask.yaml)
  echo ${LINUX_PASSWORD} | sudo -S bash -c \
    "./scripts/genconfig.sh ${ENV_FILE} deployments/templates/${SERVER_NAME}.yaml > ${OPENIM_CONFIG_DIR}/${SERVER_NAME}.yaml"
  openim::log::status "${SERVER_NAME} config file: ${OPENIM_CONFIG_DIR}/${SERVER_NAME}.yaml"

  # 3. Create and install the ${SERVER_NAME} systemd unit file
  echo ${LINUX_PASSWORD} | sudo -S bash -c \
    "./scripts/genconfig.sh ${ENV_FILE} deployments/templates/init/${SERVER_NAME}.service > ${SYSTEM_FILE_PATH}"
  openim::log::status "${SERVER_NAME} systemd file: ${SYSTEM_FILE_PATH}"

  # 4. Start the openim-crontask service
  openim::common::sudo "systemctl daemon-reload"
  openim::common::sudo "systemctl restart ${SERVER_NAME}"
  openim::common::sudo "systemctl enable ${SERVER_NAME}"
  openim::crontask::status || return 1
  openim::crontask::info

  openim::log::info "install ${SERVER_NAME} successfully"
  popd
}


# Unload
function openim::crontask::uninstall()
{
  set +o errexit
  openim::common::sudo "systemctl stop ${SERVER_NAME}"
  openim::common::sudo "systemctl disable ${SERVER_NAME}"
  openim::common::sudo "rm -f ${OPENIM_INSTALL_DIR}/bin/${SERVER_NAME}"
  openim::common::sudo "rm -f ${OPENIM_CONFIG_DIR}/${SERVER_NAME}.yaml"
  openim::common::sudo "rm -f /etc/systemd/system/${SERVER_NAME}.service"
  set -o errexit
  openim::log::info "uninstall ${SERVER_NAME} successfully"
}

# Status Check
function openim::crontask::status()
{
  # Check the running status of the ${SERVER_NAME}. If active (running) is displayed, the ${SERVER_NAME} is started successfully.
  systemctl status ${SERVER_NAME}|grep -q 'active' || {
    openim::log::error "${SERVER_NAME} failed to start, maybe not installed properly"
    return 1
  }

  # The listening port is hardcode in the configuration file
  if echo | telnet 127.0.0.1 7070 2>&1|grep refused &>/dev/null;then
    openim::log::error "cannot access health check port, ${SERVER_NAME} maybe not startup"
    return 1
  fi
}

if [[ "$*" =~ openim::crontask:: ]];then
  eval $*
fi
```



**service:**

```bash
[Unit]
Description=OPENIM OPENIM CRONTASK
Documentation=Manages the OpenIM CronTask service, with both direct and systemctl installation methods.
Documentation=https://github.com/OpenIMSDK/Open-IM-Server/blob/main/deployment/init/README.md

[Service]
WorkingDirectory=${OPENIM_DATA_DIR}/openim-crontask
ExecStartPre=/usr/bin/mkdir -p ${OPENIM_DATA_DIR}/openim-crontask
ExecStartPre=/usr/bin/mkdir -p ${OPENIM_LOG_DIR}
ExecStart=${OPENIM_INSTALL_DIR}/bin/openim-crontask -c=${OPENIM_CONFIG_DIR}
Restart=always
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```
