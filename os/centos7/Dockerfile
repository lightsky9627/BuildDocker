# syntax=docker/dockerfile:1
# ============================================================
# CentOS 7 customized base image
# Reference: https://www.bravexist.cn/posts/10087662.html
#
# Build (BuildKit required for heredoc):
#   export DOCKER_BUILDKIT=1
#   docker build -t qiankong/os:v1 .
#
# Skipped (not applicable inside a container):
#   - firewalld / systemctl       : no systemd in the default container
#   - SELinux                     : kernel-level, managed by host
#   - hostname / nmcli / IP / DNS : network managed by Docker
#   - ssh-keygen -A / sshd restart: sshd is not running; nothing to restart
#   - read -p interactive prompts : image build must be non-interactive
# ============================================================

FROM centos:7

LABEL maintainer="qiankong" \
      description="CentOS 7 with common ops tools, prompt, history, vim and alias customizations"

# ---------- 1. Replace yum source (CentOS 7 is EOL since 2024-06-30) ----------
RUN set -eux; \
    mkdir -p /etc/yum.repos.d/backup && \
    mv /etc/yum.repos.d/CentOS-*.repo /etc/yum.repos.d/backup/ 2>/dev/null || true && \
    curl -fsSL -o /etc/yum.repos.d/CentOS-Base.repo \
        https://mirrors.aliyun.com/repo/Centos-7.repo && \
    curl -fsSL -o /etc/yum.repos.d/epel.repo \
        https://mirrors.aliyun.com/repo/epel-7.repo && \
    yum makecache fast

# ---------- 2. Install tools (nodocs + no weak deps to shave size) ----------
RUN yum install -y \
        --setopt=tsflags=nodocs \
        --setopt=install_weak_deps=False \
        tree vim wget bash-completion lrzsz \
        net-tools sysstat iotop iftop htop unzip git \
        nc nmap telnet bc psmisc httpd-tools bind-utils nethogs expect && \
    yum clean all && \
    rm -rf /var/cache/yum /tmp/* /var/tmp/* /var/log/* \
           /usr/share/doc/* /usr/share/man/* /usr/share/info/* && \
    find /usr/share/locale -mindepth 1 -maxdepth 1 -type d \
         ! -name 'en*' -exec rm -rf {} + 2>/dev/null || true

# ---------- 3. Colorful PS1 prompt ----------
# KEY TRICK: quote the heredoc terminator as 'EOF' so the shell does NOT
# expand $, \e, \u, \H, \w, \$ etc. The content lands in the file verbatim.
RUN cat > /etc/profile.d/cli-color.sh <<'EOF'
export PS1='[\[\e[34;1m\]\u@\[\e[0m\]\[\e[32;1m\]\H\[\e[0m\]\[\e[31;1m\] \w\[\e[0m\]]\$ '
EOF

# ---------- 4. History settings ----------
RUN cat >> /etc/profile <<'EOF'

# --- custom history settings ---
export HISTTIMEFORMAT="%F %T `whoami` "
EOF
RUN sed -i 's/^HISTSIZE=1000/HISTSIZE=10000/' /etc/profile

# ---------- 5. Custom aliases ----------
# Note: fixed a typo from the source (`egrep -'...'` -> `egrep -v '...'`)
RUN cat >> /root/.bashrc <<'EOF'

# --- custom aliases ---
alias yy="egrep -v '^$|^#'"
EOF

# ---------- 6. Global vimrc: auto-header for shell scripts ----------
RUN cat >> /etc/vimrc <<'EOF'

set ignorecase

autocmd BufNewFile *.py,*.cc,*.sh,*.java,*.bash exec ":call SetTitle()"

func SetTitle()
    if expand("%:e") =~ 'sh\|bash'
        call setline(1, "#!/bin/bash")
        call setline(2, "##############################################################")
        call setline(3, "# File Name:".expand("%"))
        call setline(4, "# Version:V1.0")
        call setline(5, "# Author:qiankong")
        call setline(6, "# Organization:bravexist.cn")
        call setline(7, "# Desc:")
        call setline(8, "##############################################################")
    endif
endfunc
EOF

# ---------- 7. SSH config tweaks (no service restart; file edits only) ----------
# If you later run sshd inside this image, these take effect.
RUN sed -i -e 's/#UseDNS yes/UseDNS no/g' \
           -e 's/^UseDNS yes/UseDNS no/g' \
           -e 's/^GSSAPIAuthentication yes/GSSAPIAuthentication no/g' \
           /etc/ssh/sshd_config 2>/dev/null || true

# Default shell as login shell so /etc/profile.d/cli-color.sh is sourced
CMD ["/bin/bash", "-l"]
