# root用户配置
```bash
apt install -y vim sudo
apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common

touch /etc/sudoers.d/nopasswd4sudo
echo "debian ALL=(ALL) NOPASSWD : ALL" | tee -a /etc/sudoers.d/nopasswd4sudo
```
# debian用户配置脚本
```bash
#!/bin/bash

current_time=$(date +"%Y-%m-%d_%H:%M:%S")

echo "blacklist i2c_piix4" | sudo tee -a /etc/modprobe.d/blacklist.conf
echo "blacklist pcspkr" | sudo tee -a /etc/modprobe.d/blacklist.conf

sudo systemctl stop systemd-timesyncd.service
sudo systemctl disable systemd-timesyncd.service
sudo systemctl stop apparmor
sudo systemctl disable apparmor

# 验证禁用透明大页
# cat /sys/kernel/mm/transparent_hugepage/enabled
# 输出结果应该是:
# always madvise [never]
# rm /etc/systemd/system/disable-thp.service
sudo touch /etc/systemd/system/disable-thp.service
sudo tee /etc/systemd/system/disable-thp.service > /dev/null << EOF
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo never | tee /sys/kernel/mm/transparent_hugepage/enabled > /dev/null"
[Install]
WantedBy=basic.target
EOF

sudo systemctl daemon-reload
sudo systemctl start disable-thp
sudo systemctl enable disable-thp

touch ~/.vimrc
tee ~/.vimrc > /dev/null << EOF
set nocompatible
syntax on
highlight Comment ctermfg=LightCyan
set tabstop=4
set softtabstop=4
set shiftwidth=4
set expandtab
set autoindent
filetype indent on
set showmode
set encoding=utf-8
set fileencodings=utf-8
set fileformats=unix,dos
set binary noeol
set ignorecase
set smartcase
set hlsearch
set incsearch
set showmatch
set ruler
set nowrap
EOF

sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak

sudo tee /etc/apt/sources.list > /dev/null << EOF
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
EOF

# sudo apt update
# sudo apt -y upgrade
# sudo update-initramfs -u -k all
# sudo apt install -y build-essential git wget unzip zip
```
