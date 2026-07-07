# CentOS 7 定制基础镜像

一个在构建阶段就内置了常用运维工具、彩色命令行提示符、history 优化、vim 定制以及若干合理默认配置的 CentOS 7 基础镜像。

> 参考文章：https://www.bravexist.cn/posts/10087662.html
>
> **注意：** CentOS 7 已于 2024-06-30 停止维护（EOL）。官方上游源已经失效，因此本镜像在安装任何软件包之前，会先把源换成阿里云镜像。

## 镜像包含内容

| 类别 | 说明 |
|---|---|
| yum 源 | 原始 `CentOS-*.repo` 文件备份到 `/etc/yum.repos.d/backup/`，替换为阿里云的 `Centos-7.repo` 和 `epel-7.repo` |
| 运维工具 | `tree vim wget bash-completion lrzsz net-tools sysstat iotop iftop htop unzip git nc nmap telnet bc psmisc httpd-tools bind-utils nethogs expect` |
| 命令行提示符 | 通过 `/etc/profile.d/cli-color.sh` 设置彩色 `PS1`（蓝色用户名、绿色主机名、红色路径） |
| history 配置 | `HISTTIMEFORMAT` 设置为 `"%F %T <用户名> "`；`HISTSIZE` 从 1000 调整为 10000 |
| 别名 | `yy` → `egrep -v '^$|^#'`（读取配置文件时过滤空行和注释），已加入 `/root/.bashrc` |
| vim | 全局 `/etc/vimrc` 配置：新建 `.sh/.bash/.py/.cc/.java` 文件时自动插入文件头模板（作者、组织、版本、描述） |
| SSH | 修改了 `sshd_config`（`UseDNS no`、`GSSAPIAuthentication no`）——仅改配置文件，不重启服务，因为镜像里本来就没有跑 sshd |
| 镜像体积 | 安装时使用 `--setopt=tsflags=nodocs --setopt=install_weak_deps=False`；安装完成后清理了 yum 缓存、临时文件、日志以及大部分 locale/文档/man 数据 |

## 有意跳过的部分

以下内容来自原始的物理机/虚拟机初始化教程，但在容器场景下不适用，所以没有包含进来：

- **firewalld / systemctl** —— 容器默认不跑 systemd
- **SELinux** —— 内核层面的东西，由宿主机管理，不归容器管
- **hostname / nmcli / 静态 IP / DNS** —— 网络由 Docker 管理
- **`ssh-keygen -A` / 重启 sshd** —— 镜像里 sshd 本来就没启动，没有服务可重启
- **`read -p` 交互式输入** —— 镜像构建过程必须是非交互式的

## 构建

需要 BuildKit（因为用到了 `RUN ... <<'EOF'` heredoc 语法）。

```bash
export DOCKER_BUILDKIT=1
docker build -t qiankong/os:v1 .
```

## 运行

```bash
docker run -it --rm qiankong/os:v1
```

默认命令是 `/bin/bash -l`，也就是**登录 shell（login shell）**，这是必须的，因为只有登录 shell 才会去加载 `/etc/profile.d/cli-color.sh` 和 `/etc/profile` 里的 history 配置。如果你用别的 `CMD`/`ENTRYPOINT` 覆盖了它（比如直接用普通的 `bash` 而不是 `bash -l`），彩色提示符和 `HISTTIMEFORMAT` 就不会自动生效——这种情况下需要手动 `source /etc/profile`。

## 验证方法

```bash
docker run -it --rm qiankong/os:v1

# 此时彩色提示符应该已经能看到了

# 检查 history 配置
echo $HISTTIMEFORMAT
grep '^HISTSIZE' /etc/profile

# 检查别名
alias yy
yy /etc/ssh/sshd_config

# 检查 vim 头模板
vim test.sh   # 新建文件时应自动插入文件头

# 检查工具是否装好
which tree htop iftop nmap telnet expect
```

## 注意事项 / 坑点

- **源是阿里云专属的。** 如果以后阿里云的 CentOS 7 归档镜像也失效了（EOL 之后这个概率会越来越高），把第一步里的两个 `curl` 地址换成别的归档源（比如清华/中科大镜像），或者直接指向 `vault.centos.org`。
- **`yy` 别名修复了一个原文的笔误**（原文是 `egrep -'...'`，正确应为 `egrep -v '...'`）——本 Dockerfile 里已经是修正后的版本，这里只是提醒一下，方便有人对照原文章排查差异。
- **SSH 配置修改目前是"死"的**，除非你在派生镜像里或者运行时另外装了并启动了 `sshd`——因为构建时压根没有服务在跑，所以这里不涉及重启操作。
- **locale 清理**只保留了 `/usr/share/locale` 下的 `en*` 语言包；如果你需要其他语言，把这一步删掉，或者构建完之后再补装回来。
