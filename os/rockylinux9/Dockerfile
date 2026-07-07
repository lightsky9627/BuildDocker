# syntax=docker/dockerfile:1
# ============================================================
# Rocky Linux 9 customized base image
# Reference: https://www.bravexist.cn/posts/c13020d7.html
#
# Build (BuildKit required for heredoc):
#   export DOCKER_BUILDKIT=1
#   docker build -t qiankong/os:rocky9-v1 .
#
# Skipped (not applicable inside a container):
#   - firewalld / systemctl / getenforce / SELinux : no systemd, no SELinux inside the default container
#   - hostnamectl / nmcli / IP / DNS                : network and hostname are managed by Docker
#   - growpart / vgimportdevices / pvresize / lvextend : disk expansion is host/VM-level, containers use overlay storage
#   - ssh-keygen -A / sshd restart                   : sshd is not running; nothing to restart
#   - read -p interactive prompts                     : image build must be non-interactive
# ============================================================
FROM rockylinux:9
LABEL maintainer="qiankong" \
      description="Rocky Linux 9 with common ops tools, prompt, history, vim and alias customizations"

# ---------- 1. Replace yum/dnf source (mirrorlist -> Aliyun baseurl) ----------
RUN set -eux; \
    sed -e 's|^mirrorlist=|#mirrorlist=|g' \
        -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
        -i.bak \
        /etc/yum.repos.d/rocky*.repo

# ---------- 2. EPEL source ----------
RUN dnf install -y epel-release && \
    dnf makecache

# ---------- 3. Install tools (nodocs + no weak deps to shave size) ----------
RUN dnf install -y \
        --setopt=tsflags=nodocs \
        --setopt=install_weak_deps=False \
        tree vim wget bash-completion lrzsz \
        net-tools sysstat iotop iftop htop unzip git lsof \
        nc nmap telnet bc psmisc httpd-tools bind-utils nethogs expect \
        openssh-server && \
    dnf clean all && \
    rm -rf /var/cache/dnf /tmp/* /var/tmp/* /var/log/* \
           /usr/share/doc/* /usr/share/man/* /usr/share/info/* && \
    find /usr/share/locale -mindepth 1 -maxdepth 1 -type d \
         ! -name 'en*' -exec rm -rf {} + 2>/dev/null || true

# ---------- 4. Colorful PS1 prompt ----------
# KEY TRICK: quote the heredoc terminator as 'EOF' so the shell does NOT
# expand $, \e, \u, \H, \w, \$ etc. The content lands in the file verbatim.
RUN cat > /etc/profile.d/cli-color.sh <<'EOF'
export PS1='[\[\e[34;1m\]\u@\[\e[0m\]\[\e[32;1m\]\H\[\e[0m\]\[\e[31;1m\] \w\[\e[0m\]]\$ '
EOF

# ---------- 5. History settings ----------
RUN cat >> /etc/profile <<'EOF'
# --- custom history settings ---
export HISTTIMEFORMAT="%F %T `whoami` "
EOF
RUN sed -i 's/^HISTSIZE=1000/HISTSIZE=10000/' /etc/profile

# ---------- 6. Custom aliases ----------
RUN cat >> /root/.bashrc <<'EOF'
# --- custom aliases ---
alias yy="egrep -v '^$|^#'"
EOF

# ---------- 7. Global vimrc: auto-header for shell scripts ----------
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

# ---------- 8. Timezone ----------
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# ---------- 9. SSH config tweaks (no service restart; file edits only) ----------
# If you later run sshd inside this image, these take effect.
RUN sed -i -e 's/#UseDNS yes/UseDNS no/g' \
           -e 's/^UseDNS yes/UseDNS no/g' \
           -e 's/^GSSAPIAuthentication yes/GSSAPIAuthentication no/g' \
           /etc/ssh/sshd_config 2>/dev/null || true

# Default shell as login shell so /etc/profile.d/cli-color.sh is sourced
CMD ["/bin/bash", "-l"]
