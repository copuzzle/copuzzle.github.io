
# Systemd 使用简介

## 1.简介 

Systemd 是 linux 上新一代的系统和服务管理软件。它的优缺点如下。 

优点： 

- 用描述性的 DSL 替代传统的 rc 系统的脚本，启动速度快
- 多种功能更集成化
- 使用 cgroup 来管理进程(组)，避免了传统 rc 系统的父进程非正常退出时其子进程失控的问题
- 内建 journal 功能

缺点（见仁见智）： 

- 多种功能的紧密集成，破坏了组件可替换的 Unix 传统
- 集成了过多的功能
- 二进制 journal 文件损坏后，处理相对麻烦

systemd 由以下几个部分构成（路径随版本会有差异） 

组件或功能 
说明 
/sbin/init 
符号链接到 /lib/systemd/systemd，系统的 pid=1 进程，是最重要的组件功能 
/lib/systemd/systemd-journald 
处理 journal (即 log)。需要注意的是，它会接受 syslog 
/lib/systemd/systemd-udevd 
代替原本的 udevd。它监听内核消息，执行 udev 规则来处理设备文件 
/lib/systemd/systemd-logind 
处理用户登录、会话，支持 multi-seat 功能。例如 X 登录及控制台登录 
/lib/systemd/systemd --user
systemd 用户实例，用户服务管理进程。这是本文档要描述的核心部分，也是我们将使用的主要功能 

## 2.用户服务管理进程 


用户服务管理进程是管理用户的服务进程的进程。以传统的 rc 系统为例，简要描述功能和优缺点 

sysvinit 

- /sbin/init 进程启动后，会读取 /etc/inittab，根据规则直接管理一些进程。这些进程退出后，其对应程序会被重新运行(respawn)
- 根据一些特殊规则，init 会响应一些事件，例如键盘组合键
- init 有状态 (runlevel)，init 会在不同 runlevel 执行指定的脚本 (/etc/rcN.d)，实际上会派生很多服务进程
- 通过 runlevel 脚本启动的服务，init 不直接管理，服务进程异常退出不会 respawn

daemontools 

- daemontools  对 sysvinit 进行补充。它的主进程 svscan 可以从 inittab 中由 init 启动并直接管理，也可以由 runlevel 脚本启动，需要注意的是，runlevel 脚本启动的 svscan 退出后不会重新运行
- svscan 扫描指定的目录下的子目录(通常是 /etc/service/<subdir> )，每个子目录启动一个 supervise 进程，这个进程会监控子目录，执行子目录下的 run 脚本
- supervise 使用 wait() 系统调用，在子进程退出后 respawn。所以 run 脚本最后要用 exec 执行程序，执行的程序不能 daemon 化
- supervise 使用该子目录下的特别文件来与 svscan 、命令行工具 svc交互
- supervise 被杀死后，其管理的子进程会失控
- svscan 和 supervise 是弱耦合的，svscan 被杀死不影响 supervise，可以重新启动 svscan，这是优势，也是劣势
- svstat 这个状态查看工具，没有内建的列出服务的功能，需要用一些 unix 工具从文件系统上查，容易出错
- 由于 supervise 会在子目录中创建一些文件，在一些情况管理操作中是“不干净的”，处理较复杂
- 适合保持那些无状态的或者幂等的服务，对于有状态或非幂等的服务，daemontools 并不适合

## 3.systemd 用户实例的配置 


通常，用户的第一个会话创建时，由 /lib/systemd/systemd-logind 启动 /lib/systemd/systemd --user；最后一个会话结束后，/lib/systemd/systemd --user 进程退出。如果想让 /lib/systemd/systemd --user 在系统启动时启动，之后持续运行，很简单 

# loginctl enable-linger user1 [user2 ...]
  
例如
# loginctl enable-linger root tiger

  

但是，systemd 用户实例的默认设置往往不够，无法达成一些特殊要求，例如 

- 被杀死后自动重启
- 调整各种 limit 值，例如可打开的文件描述符数量

Systemd 用户实例的配置由 /etc/systemd/system 下的 user@ 配置实现，具体而言 

- /etc/systemd/system/user@1000.service.d/*.conf  影响 uid=1000 的用户的 systemd 用户实例
- /etc/systemd/system/user@0.service.d/*.conf 影响 uid=0 即 root 的 systemd 用户实例
- /etc/systemd/system/user@.service.d/*.conf 影响所有用户的 systemd 用户实例

如果这些目录 (user@.service.d 等) 不存在，可创建目录。以下举两个例子 

	/etc/systemd/system/user@1000.service.d/always.conf

[Service]
Restart=always

	/etc/systemd/system/user@1000.service.d/limits.conf

[Service]
LimitNOFILE=100000

  

添加配置后，需要 systemctl daemon-reload 

  

具体可配置参数可参考 man page (systemd.service 、systemd.resource-control 和 systemd.exec) 


## 4.用户服务的配置 


如前所述，systemd 用户实例用于启动用户服务。用户服务的配置有以下目录 

- /usr/lib/systemd/user，系统预装软件所需要的用户服务配置
- /etc/systemd/user ，由系统管理员给所有用户添加的服务
- ~/.config/systemd/user ，用户自定义的服务

这里给一个简单的例子 

	~tiger/.config/systemd/user/sleep.service

[Unit]
Description=a sleep example service
[Service]
ExecStart=/bin/sleep 1200
Restart=always
[Install]
WantedBy=default.target

  

启用使之生效，然后启动 

$ systemctl --user enable sleep[.service]
Created symlink from /home/tiger/.config/systemd/user/default.target.wants/sleep.service to /home/tiger/.config/systemd/user/sleep.service.
$ systemctl --user start sleep[.service]

停止和删除一个服务 

systemctl --user stop sleep[.service]     #停止一个服务
systemctl --user disable sleep[.service]  #删除一个服务

## 5.扩展或修改系统随机带的服务的行为（配置） 


在上面的例子里，我们使用 user@.service.d 中的配置来修改 user.service 的行为。 

如果要扩展或修改随系统安装的服务的行为（配置），不应该直接修改系统配置，而是创建一个以服务的配置文件名后追加 .d 的目录，然后在这个目录下创建任意以 .conf 为后缀的配置文件，指定你需要增加或修改的选项。 

你需要用 systemctl daemon-reload 让修改生效，然后重启服务。 

见 http://www.freedesktop.org/software/systemd/man/systemd.unit.html 


## 6.Systemd的问题


Systemd 管理进程默认是通过 cgroup。当 systemd --user 用户实例被杀掉（本身很稳定，只能是误操作或特意如此）时，其启动的服务进程都将被杀掉。这和 daemontools 的行为不一样。 

daemontools 是 svscan 启动 supervise，由 supervise 启动服务。当杀死 svscan 后，supervise 不受影响，服务不受影响。 

systemd --user 的配置中，可以指定 KillMode 来修改行为。KillMode 是 systemctl stop <service> 的行为。KillMode 的值如下 

KillMode 
行为 
control-group 
同一 cgroup 的所有进程会被 kill 
process 
仅此进程被杀死 
mixed 
此进程被 SIGTERM，同 cgroup 的其它进程被 SIGKILL 
none 
任何进程不会被杀 

但 --user 用户实例只能使用 control-group 和 mixed 两种模式。 

当 systemd --user 用户实例以 Restart=always 启动后，直接 kill 它的操作会使得它管理的处于 enabled 状态的所有服务重启（即使该服务没有指定 Restart=always，没有处于运行状态)。 


## 7.其它配置问题 


### 7.1  su - username 无法执行 systemctl --user 命令 


systemctl --user enable some.service 这样的命令，连接到 systemd --user 所监听的 unix socket 上进行通信，而这个 unix socket 的位置是由 XDG_RUNTIME_DIR 环境变量决定的。 

正常 ssh 登录，这个环境变量会正确设置。而 su - username 不会。这一个 systemd 的 feature/bug 

（见 https://bugs.freedesktop.org/show_bug.cgi?id=70810 和 https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=724731#250 ) 

临时解决办法是 

1

	export XDG_RUNTIME_DIR=/run/user/$(id -u)

然后执行 systemd 相关命令 


### 7.2  Restart=always 不生效 


问题：即使设置了 Restart=always，有时候服务还是不能重启。 

为了防止过于快速的重启对系统造成压力，重启有一个默认的间隔 RestartSec；另外还有一个隐含的检查，如果在 10s 中有多于 5 次重启，则触发 start limit，不能再自动重启（可以/必须手工重启）。如果服务进程的代码或者配置问题造成无法重启，通常会触发 10s 5次的限制。（参考 man systemd.unit 的 StartLimitInterval，以及 man systemd.service 的 RestartSec ） 

比较好的解决的办法是设置 RestartSec=2 或以上。 

[Service]
Restart=always
RestartSec=2s500ms

man systemd.time 可查看指定时间的方式 

 另外一种情况是用 sysv 脚本启动的服务，默认为 RemainAfterExit=yes，即进程退出也是正常状态，不会触发 restart。解决方法 

[Service]
Restart=always
RestartSec=2s500ms
RemainAfterExit=no
PIDFile=/path/to/pidfile

### 7.3  Enable 失败 


systemctl enable 需要 service 文件中包含 [Install] 声明。 

错误信息对应的问题 

错误信息 
问题 
解决办法 
Failed to execute operation: No such file or directory 
	1. 在正常的搜索目录下不存在指定的 service 文件
	2. 存在 service 文件，它是一个符号链接，且链接层次太多导致被判断不存在 (ELOOP)
	3. root下enable非/lib/systemd/ 下的服务，重启后会有这个问题。需要将root下的服务（进程1拉起的服务放到 /lib/systemd/system/ 下）
无需将部署目录的 service 文件链接至 ~/.config/systemd/user 下，直接 enable 部署目录的 service 文件 

### 7.4  可实例化服务(服务模板) 


在一些时候，我们需要起多个相同的服务，只是运行参数不同。systemd 提供了可实例化的服务模板机制。其实在前述 2 章节中的 user@.service 就是模板服务。 

可实例化的服务是定义一个  name@.service，然后 enable 或 start 时指定一个实例名称，形如 name@instance.service， 这个实例名称可以作为参数传递给服务定义，以下是一个例子 

	example@.service

[Unit]
Description=Example Instanced service, sleep %I seconds
After=network.target
 
 
[Service]
ExecStart=/bin/sleep %I

使用 

启动一个 sleep 10000 秒的实例（不一定持久化）
# systemctl start example@10000.service
  
生成一个 1000 秒的实例对应的符号链接（持久化），但不启动
# systemctl enable example@1000.service

服务定义中，可以用 %i (instance name) 或者 %I ( unescaped instance name) 来获取 name@ 和 .service 之间的实例名称。具体可参考 man systemd.unit。 

需要注意的一点是，如果命令行上指定的 instance name 含有“ - ”字符，用 %I 时会把“-” 字符替换为 “/” 字符，此种情况应该用 %i。 

另外，可实例化服务的 .d 配置目录是先  name@instance.service.d 然后 name@.service.d ，即先实例然后模板。 

服务模板的细节可参考 man systemd.unit 


### 7.5  服务不能优雅退出，被强行杀死(SIGKILL) 


服务stop（或者是restart），起始systemd会尝试用SIGTERM来让服务自己优雅退出，如果超时（默认90s）之后，服务仍未退出，那么他就会用SIGKILL杀死服务。可以配置Service段的TimeoutStopSec来延长时间，或者把这个值设置成0（不推荐）来禁止timeout，即永不超时。 


### 7.6  服务的 stdout 和 stderr 


见 man systemd.exec 的 StandardOutput= 和 StandardError= 

例如，将 stderr 丢弃 

[Service]
StandardError=null

使用其它选项前应仔细看文档。 


### 7.7  删除有问题的unit服务 


使用 systemctl reset-failed 命令来清理有问题的unit，systemd 并没有提供直接删除某个 unit 的功能。 

