# Rocky Linux 9 定制基础镜像

一个在构建阶段就内置了常用运维工具、彩色命令行提示符、history 优化、vim 定制以及若干合理默认配置的 Rocky Linux 9 基础镜像。

> 参考文章：https://www.bravexist.cn/posts/c13020d7.html（原文以 CentOS Stream 9 为主，文中同时给出了 Rocky 9 的换源命令，本 Dockerfile 采用的是 Rocky 9 分支）

## 镜像包含内容

| 类别 | 说明 |
|---|---|
| dnf 源 | 用 `sed` 把 `/etc/yum.repos.d/rocky*.repo` 里的 `mirrorlist` 注释掉，`baseurl` 替换为阿里云镜像 `https://mirrors.aliyun.com/rockylinux`（原文件自动备份为 `.bak`） |
| EPEL 源 | `dnf install -y epel-release`（Rocky 9 的 EPEL 直接走上面换好的阿里云源，不需要像 CentOS Stream 分支那样再单独改 EPEL 的 baseurl） |
| 运维工具 | `tree vim wget bash-completion lrzsz net-tools sysstat iotop iftop htop unzip git lsof nc nmap telnet bc psmisc httpd-tools bind-utils nethogs expect` |
| SSH | 额外装了 `openssh-server`，防止后续升级 openssl 底层库时 sshd 相关组件缺失导致故障（原文提到的坑） |
| 命令行提示符 | 通过 `/etc/profile.d/cli-color.sh` 设置彩色 `PS1`（蓝色用户名、绿色主机名、红色路径） |
| history 配置 | `HISTTIMEFORMAT` 设置为 `"%F %T <用户名> "`；`HISTSIZE` 从 1000 调整为 10000 |
| 别名 | `yy` → `egrep -v '^$|^#'`（读取配置文件时过滤空行和注释），已加入 `/root/.bashrc` |
| vim | 全局 `/etc/vimrc` 配置：新建 `.sh/.bash/.py/.cc/.java` 文件时自动插入文件头模板（作者、组织、版本、描述） |
| 时区 | 直接软链 `/etc/localtime` 到 `Asia/Shanghai`（构建阶段没有 `timedatectl`/systemd，用软链方式等效于原文里的 `timedatectl set-timezone`） |
| SSH 配置调优 | 修改了 `sshd_config`（`UseDNS no`、`GSSAPIAuthentication no`）——仅改配置文件，不重启服务，因为镜像里本来就没有跑 sshd |
| 镜像体积 | 安装时使用 `--setopt=tsflags=nodocs --setopt=install_weak_deps=False`；安装完成后清理了 dnf 缓存、临时文件、日志以及大部分 locale/文档/man 数据 |

## 有意跳过的部分

原文是针对物理机/虚拟机场景的初始化教程，以下内容在容器场景下不适用，所以没有包含进来：

- **firewalld / systemctl / getenforce / SELinux** —— 容器默认不跑 systemd，也没有独立的 SELinux 上下文，这些都归宿主机管
- **hostnamectl / nmcli / 静态 IP / DNS** —— 主机名和网络由 Docker 管理
- **growpart / vgimportdevices / pvresize / lvextend（硬盘扩容）** —— 这是物理磁盘/LVM 层面的操作，容器用的是 overlay 存储，没有对应的分区/LVM 结构
- **`ssh-keygen -A` / 重启 sshd** —— 镜像里 sshd 本来就没启动，没有服务可重启
- **`read -p` 交互式输入** —— 镜像构建过程必须是非交互式的

## 构建

需要 BuildKit（因为用到了 `RUN ... <<'EOF'` heredoc 语法）。

```bash
export DOCKER_BUILDKIT=1
docker build -t qiankong/os:rocky9-v1 .
```

## 运行

```bash
docker run -it --rm qiankong/os:rocky9-v1
```

默认命令是 `/bin/bash -l`，也就是**登录 shell（login shell）**，这是必须的，因为只有登录 shell 才会去加载 `/etc/profile.d/cli-color.sh` 和 `/etc/profile` 里的 history 配置。如果你用别的 `CMD`/`ENTRYPOINT` 覆盖了它（比如直接用普通的 `bash` 而不是 `bash -l`），彩色提示符和 `HISTTIMEFORMAT` 就不会自动生效——这种情况下需要手动 `source /etc/profile`。

## 验证方法

```bash
docker run -it --rm qiankong/os:rocky9-v1

# 此时彩色提示符应该已经能看到了

# 检查换源结果
cat /etc/yum.repos.d/rocky.repo | grep baseurl

# 检查 history 配置
echo $HISTTIMEFORMAT
grep '^HISTSIZE' /etc/profile

# 检查别名
alias yy
yy /etc/ssh/sshd_config

# 检查 vim 头模板
vim test.sh   # 新建文件时应自动插入文件头

# 检查时区
cat /etc/timezone

# 检查工具是否装好
which tree htop iftop nmap telnet expect
```

## 注意事项 / 坑点

- **源是阿里云专属的。** 原文中同时给出了中科大（USTC）和阿里云两种 Rocky 9 换源方式，本 Dockerfile 采用阿里云。如果阿里云源访问不稳定，把 `sed` 里的 `baseurl` 换成 `https://mirrors.ustc.edu.cn/rocky` 即可。
- **EPEL 顺序问题。** 原文（CentOS Stream 分支）是先装 `epel-release`，再单独用 `sed` 把 EPEL 的 baseurl 也指向清华源；但 Rocky 9 的 EPEL 包本身就会用已经改好的 dnf 源解析，所以这里省了一步单独改 EPEL baseurl 的操作，实测装包速度是一致的。如果发现 EPEL 走了官方源导致变慢，可以额外加一步专门替换 `/etc/yum.repos.d/epel*.repo`。
- **时区设置没有用 `timedatectl`。** 因为镜像构建阶段没有 systemd/dbus，`timedatectl` 在容器里通常不可用，改用软链 `/etc/localtime` 的方式，效果等价。
- **SSH 配置修改目前是"死"的**，除非你在派生镜像里或者运行时另外装了并启动了 `sshd`——因为构建时压根没有服务在跑，所以这里不涉及重启操作。装 `openssh-server` 只是为了避免以后升级 openssl 时相关组件跟不上而导致后续手动跑 sshd 出问题。
- **locale 清理**只保留了 `/usr/share/locale` 下的 `en*` 语言包；如果你需要其他语言，把这一步删掉，或者构建完之后再补装回来。



# 最终

编译

```bash
# Rocky 9
cd ../rocky9
export DOCKER_BUILDKIT=1
docker build -t bravexist/os:rocky9 .
```

推送

```
skopeo copy \
  docker-daemon:bravexist/os:rocky9 \
  docker://docker.io/bravexist/os:rocky9 \
  --dest-creds=你的用户名:你的密码或Token
```

