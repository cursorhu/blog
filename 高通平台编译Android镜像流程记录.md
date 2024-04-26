---
title: 高通平台编译Android系统镜像记录
date: 2019-09-26 17:34:02
tags: Android
categories: Android
---

本文记录在高通开发平台HDK845上编译Android系统镜像的过程

## 一、 搭建Shadowsocks+Privoxy代理

### 1.1为什么需要搭代理

下载Android源码需要访问国外代码源，直接访问会被GFW阻挡，代理服务器（VPS）是未被GFW阻挡的国外服务器，通过代理服务器跳转至目标服务器访问国外代码源。

### 1.2 shadowsocks+privoxy代理架构

使用shadowssocks+privoxy搭建客户端代理，如下图客户端进程发送请求（http/https/git）到privoxy，privoxy将请求转化为socks5请求，发送给shadowsocks客户端，shadowsocks处理socks5请求,将其发送到远端VPS上运行的socks5服务端（shadowsocks server），VPS再将请求转发给目标服务器。

![image001](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281739480.png)

### 1.3 shadowsocks+privoxy代理搭建

#### 1.3.1 shadowsocks

HOST系统：ubuntu 14.04 LTS
安装shadowsock
```
apt-get -y install python-pip 
pip install shadowsocks 
```
配置shadowsocks client
```
gedit /etc/ss.json 
输入以下内容: 
{ 
"server":"176.122.xxx.xx", 
"server_port":8080, 
"local_address":"127.0.0.1", 
"local_port":1080, 
"password":"xxxxx", 
"timeout":100, 
"method":"aes-256-cfb" 
} 

```
运行shadowsocks客户端
```
sslocal -c /etc/ss.json > ss.log 2>&1 &  
查看服务是否起来:  
ps -ef | grep sslocal  
```
若开机启动可写入`/etc/rc.local`

#### 1.3.2 privoxy

下载privoxy稳定版本
```
wget http://www.privoxy.org/sf-download-mirror/Sources/3.0.26%20%28stable%29/privoxy-3.0.26-stable-src.tar.gz 
tar -zxvf privoxy-3.0.26-stable-src.tar.gz 
cd privoxy-3.0.26-stable 
```
privoxy服务需要新建privoxy用户,并添加到privoxy用户组来运行
```
useradd privoxy 
groupadd -g 888 privoxy  
gpasswd -a privoxy privoxy  
```
查看privoxy用户信息
```
id privoxy 
```
安装provoxy
```
apt-get -y install autoconf 
autoheader && autoconf
./configure 
make && make install 
```
设置privoxy监听http/https/git的端口，和privoxy面向socks5的端口
```
gedit /usr/local/etc/privoxy/config 
下面两行取消注释 
listen-address 127.0.0.1:8118 
forward-socks5t / 127.0.0.1:1080 
```
启动privoxy
```
privoxy --user privoxy /usr/local/etc/privoxy/config 
ps -ef | grep sslocal 
```
若开机启动可写入`/etc/rc.local`

#### 1.3.3 设置代理环境变量

http/https/ftp请求的代理端口设置为privoxy的监听端口
```
gedit /etc/profile 
export http_proxy="http://127.0.0.1:8118" 
export https_proxy="http://127.0.0.1:8118" 
export ftp_proxy="http://127.0.0.1:8118" 
```
生效并测试, curl返回大堆json字符串
```
source /etc/profile 
curl http://www.google.com 
```
系统的http(s)等请求的代理配置完成

#### 1.3.4 设置git代理

安装并配置git
```
apt-get install git 
git config --global user.email "yourname@xxx.com"  
git config --global user.name "yourname" 
git config --global http.proxy http://127.0.0.1:8118 
git config --global https.proxy http://127.0.0.1:8118 
```
设置git使用代理
```
apt-get install connect-proxy 
mkdir ~/bin 
echo "connect-proxy -S 127.0.0.1:1080 \"\$@\"" > ~/bin/socks5proxywrapper 
chmod 755 ~/bin/socks5proxywrapper 
git config --global core.gitproxy `echo $HOME`/bin/socks5proxywrapper 
```

## 二、下载编译Android源码

### 2.1 交叉编译的概念

\- 1 本地编译：在当前编译平台下，编译出来的程序只能运行在当前平台。常见的应用软件开发的编译都属于本地编译。 
\- 2 交叉编译：在当前编译平台下，编译出来的程序能运行在另一种体系结构不同的目标平台上，但是编译平台本身却不能运行该程序。 
\- 3 交叉编译工具链：编译过程包括了预处理、编译、汇编、链接等过程。每个子过程都是单独的工具来实现。交叉编译链是为了编译跨平台体系结构的程序代码而形成的由多个子工具构成的一套完整的工具集。 
![image003](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281739906.png)
如上图，交叉编译工具链中最主要的部分包含编译器（如gcc）,汇编器（如as）,连接器（如ld）。通常as和ld及objcopy等其他工具由GNU打包成了binutils（binary utilitys)工具，再加上编译器组成整个工具链。
其中编译器命名规则为：

```
arch-core-kernel-system-compiler 

arch：目标平台架构，如arm, x86_64 
core： 目标平台的CPU Core，如Cortex A8 
kernel： 目标平台所运行的OS，如Linux，Android 
systen：交叉编译链所选择的库函数和目标系统的规范，如gnu，gnueabi等 
compiler: 编译器名，如gcc, g++,clang,clang++ 
```

\- 4 交叉编译架构： 
HOST OS 通常为Linux，包含自身的kernel、glibc基础库和Target程序的依赖库。Toolchain包含C/C++及其他语言编译器和汇编、链接器等组件。Toolchain依赖于HOST的glibc基础库。Target binary是编译出的目标镜像/程序，编译过程依赖于Toolchain及HOST的build essential libs。 
![image005](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281740465.png)

### 2.2 高通Android平台编译概念 

高通平台HDK845推荐的编译环境如下：

| HOST            | Toolchain  | Source code repository | build out Android version |
| --------------- | ---------- | ---------------------- | ------------------------- |
| Ubuntu14.04 LTS | Clang/LLVM | CAF                    | support Android 9 Pie     |

高通平台HDK845推荐的编译流程如下： 
![image006](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281740419.png)

Clang/LLVM编译器介绍   
![clangLLVM](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281748816.png)
CAF和AOSP的介绍  

```
CAF is Code Aurora repository. It's the place where Qualcomm releases source code for their phone processors.  
It's directly supported by Qualcomm and it's generally a more optimized branch for Snapdragon phones.  
Actually, there are two main baselines for support of Qualcomm devices:  
- 1. CodeAurora (CAF) - These are Qualcomm's reference sources for their platform.  
This is what they provide to OEMs, and what nearly all OEMs base their software off of.  
As a result - nearly all non-Nexus devices are running kernels/display HALs/etc. that are derived from a CAF baseline.  
- 2. Google's software baseline(AOSP) - Usually when Google starts working on a new Android version, they'll fork from CAF at the beginning.  
Very often Google will be adding "new" features specific to the new Android version, while Qualcomm will continue with performance enhancements and bugfixes against the "old" baseline.  
- 3. So when a new Android revision comes out, you have two baselines: CAF which is usually "ahead" in performance but "behind" in features,  while AOSP is “behind” in performance (relatively) but “ahead” in features.  
Nowadays, developers are directly compiling the builds from CAF source code which is really difficult as this is what Google does initially before upgrading to a new version,  
and then they add features and the source by the time gets ‘compilable’, it is easier to compile the one on Google Sources than the one which is there on CAF.  
CAF can be considered as Vanilla version of a Vanilla version of Android.  
```
### 2.3 高通Android平台编译流程

\- 1 安装jdk，用于编译Android源码中的java代码： 
```
add-apt-repository ppa:openjdk-r/ppa 
apt-get update 
apt-get -y install openjdk-8-jdk 
update-alternatives --config java 
java -version 
```
\- 2 安装HOST(ubuntu14.04)的build essentials，编译过程依赖这些工具和库 
```
apt-get -y install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip libssl-dev libc6:i386 libstdc++6:i386
```
\- 3 安装repo，用于下载android源码 
```
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo 
chmod +x ~/bin/repo 
export PATH=~/bin:$PATH 
repo --help 
```
\- 4 解压开发板厂商的BSP，其中包含源码下载的脚本、补丁包等
```
unzip Open-Q_845_Android-P_v2.1.zip 
cd Open-Q_845_Android-P_v2.1/Source_Package 
chmod +x getSource_and_build.sh 
```
\- 5 用脚本从CAF源下载代码，打补丁后编译 
```
./getSource_and_build.sh 
```

`./getSource_and_build.sh`内容如下 
```
UNDER='\e[4m' 
RED='\e[31;1m' 
GREEN='\e[32;1m' 
YELLOW='\e[33;1m' 
BLUE='\e[34;1m' 
MAGENTA='\e[35;1m' 
CYAN='\e[36;1m' 
WHITE='\e[37;1m' 
ENDCOLOR='\e[0m' 
ITCVER="P_v2.1" 
WORKDIR=`pwd` 
CAFTAG="LA.UM.7.3.r1-06700-sdm845.0" 
BUILDROOT="${WORKDIR}/SDA845_Open-Q_845_Android-${ITCVER}" 
PATCH_DIR="${WORKDIR}/patches" 
DB_PRODUCT_STRING="Open-Q 845 HDK Development Kit" 

function download_CAF_CODE() { 
# Do repo sanity test 
if [ $? -eq 0 ] 
then 
  echo "Downloading code please wait.." 
  repo init -q -u git://codeaurora.org/platform/manifest.git -b release -m ${CAFTAG}.xml 
  repo sync -q -c -j 4 --no-tags --no-clone-bundle 
  if [ $? -eq 0 ] 
  then 
    echo -e "$GREEN Downloading done..$ENDCOLOR" 
  else 
    echo -e "$RED!!!Error Downloading code!!!$ENDCOLOR" 
  fi 
else 
  echo "repo tool problem, make sure you have setup your build environment" 
  echo "1) http://source.android.com/source/initializing.html" 
  echo "2) http://source.android.com/source/downloading.html (Installing Repo Section Only)" 
  exit -1 
fi 
} 

# Function to check result for failures 
check_result() { 
if [ $? -ne 0 ] 
then 
  echo 
  echo -e "$RED FAIL: Current working dir:$(pwd) $ENDCOLOR" 
  echo 
  exit 1 
else 
  echo -e "$GREEN DONE! $ENDCOLOR" 
fi 
} 

# Function to autoapply patches to CAF code 
apply_android_patches() 
{ 
  echo "Applying patches ..." 
  if [ ! -e $PATCH_DIR ] 
  then 
    echo -e "$RED $PATCH_DIR : Not Found $ENDCOLOR" 
    return 
  fi 
  cd $PATCH_DIR 
  patch_root_dir="$PATCH_DIR" 
  android_patch_list=$(find . -type f -name "*.patch" | sort) && 
  for android_patch in $android_patch_list; do
    android_project=$(dirname $android_patch) 
    echo -e "$YELLOW  applying patches on  $android_project ... $ENDCOLOR" 
    cd $BUILDROOT/$android_project 
    if [ $? -ne 0 ]; then 
      echo -e "$RED $android_project does not exist in BUILDROOT:$BUILDROOT $ENDCOLOR" 
      exit 1 
    fi 
    git am --3way $patch_root_dir/$android_patch 
    check_result 
  done 
} 

# Function to check whether host utilities exists 
check_program() { 
for cmd in "$@" 
do 
  which ${cmd} > /dev/null 2>&1 
  if [ $? -ne 0 ] 
  then 
    echo 
    echo -e "$RED Cannot find command \"${cmd}\"  $ENDCOLOR" 
    echo 
    exit 1 
  fi 
done 
} 

#Main Script starts here 
#Note: Check necessary program for installation 
echo 
echo -e "$CYAN Product          : $DB_PRODUCT_STRING $ENDCOLOR" 
echo -e "$MAGENTA Intrinsyc Release Version : $ITCVER $ENDCOLOR" 
echo -e "$MAGENTA WorkDir          : $WORKDIR $ENDCOLOR" 
echo -e "$MAGENTA Build Root        : $BUILDROOT $ENDCOLOR" 
echo -e "$MAGENTA Patch Dir         : $PATCH_DIR $ENDCOLOR" 
echo -e "$MAGENTA CodeAurora TAG      : $CAFTAG $ENDCOLOR" 
echo -n "Checking necessary program for installation......" 
echo 
check_program tar repo git patch 
if [ -e $BUILDROOT ] 
then 
  cd $BUILDROOT 
else 
  mkdir $BUILDROOT 
  cd $BUILDROOT 
fi 

#1 Download code 
download_CAF_CODE 
cd $BUILDROOT 

#2 Apply Open-Q 845 HDK Development Kit Patches
apply_android_patches 

#3 Extract the proprietary objs 
cd $BUILDROOT 
echo -e "$YELLOW  Extracting proprietary binary package to $BUILDROOT ... $ENDCOLOR" 
tar -xzvf ../proprietary.tar.gz -C vendor/qcom/ 

#4 Build 
echo -e "$YELLOW  Building Source code from $BUILDROOT ... $ENDCOLOR" 
if [[ -z "${BUILD_NUMBER}" ]]; then export BUILD_NUMBER=$(date +%m%d%H%M); fi 
. build/envsetup.sh 
lunch sdm845-${BV:="userdebug"} 
ITC_ID=Open-Q_845_${ITCVER} make -j $(nproc) $@ 
```

编译后生成bootloader和系统等镜像：
SDA845_Open-Q_845_Android-P_v2.1/out/target/product/sdm845/xxx.img
后续重新编译只需要注释掉`./getSource_and_build.sh`的`步骤#1 #2 #3`，保留`#4 Build`

## 二、 烧写Android镜像

### 3.1 烧写、调试、打印的工具

开发板通过micro USB和type-C USB连接到主机 
type-C: 用于开发板接收adb/fastboot
micro USB： 用于HOST接收开发板的输出打印 
连接如下：

![image007](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281740889.png)

HOST端用到的工具：
fastboot: 用于烧写Android镜像到开发板 
adb(Android Debug Bridge): 用于调试Android系统 
secureCRT: 用于查看开发板串口打印 
\- 1 首先配置fastboot和adb到系统环境变量，windows环境下`win + R`输入`cmd`配置`PATH`变量

```
set PATH=%PATH%;d:\platform-tools\adb.exe
set PATH=%PATH%;d:\platform-tools\fastboot.exe
```
确认adb和fastboot加到了`PATH`环境变量
```
echo %PATH%
```
\- 2 查看开发板对应的com口，secureCRT新建会话，设置serial，设置com口和波特率115200

![image009](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281740486.png)

### 3.2 烧写镜像

\- 1 首先使开发版进入fastboot模式，连接micro USB，电源选项拨到DC电源, 上电后长按vol-, 然后连接type-C，串口打印出现`Fastboot: Processing commands`则进入fastboot。

![image011](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281740947.png)
\- 2 `win + R`打开`cmd`，用fastboot烧写编译出来的镜像 

```
fastboot flash system system.img 
fastboot flash persist persist.img 
fastboot flash boot boot.img  
fastboot flash dtbo dtbo.img 
fastboot flash vbmeta vbmeta.img 
fastboot flash vendor vendor.img 
fastboot reboot 
```
可写入flash.bat脚本,放到系统镜像同一目录下运行
```
@echo off 

@echo Reboot bootloader... 
adb reboot bootloader 

@echo Flashing device... 
fastboot flash system system.img 
fastboot flash persist persist.img 
fastboot flash boot boot.img 
fastboot flash dtbo dtbo.img 
fastboot flash vbmeta vbmeta.img 
fastboot flash vendor vendor.img 

@echo Flashing finish, rebooting system... 
fastboot reboot 
```

![image012](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281741449.png)
完成后系统重启进入Android桌面。

![image014](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303281741637.png)
