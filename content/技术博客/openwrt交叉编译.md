**一、准备工作：**

1.Linux环境，我使用的是我们实验室的服务器ubuntu22.04，也可以使用虚拟机

2.更新软件源

sudo apt update

sudo apt install -y build-essential ccache ecj fastjar file g++ gawk \

gettext git java-propose-classpath libelf-dev libncurses5-dev \

libncursesw5-dev libssl-dev python3 python3-distutils python3-setuptools \

python3-dev unzip wget rsync subversion swig time xsltproc zlib1g-dev

3.获取代码

# 选项 A: 克隆 Master 分支 (最新，但可能不稳定)

git clone https://git.openwrt.org/openwrt/openwrt.git

# 选项 B: 克隆稳定版 (例如 23.05 版本，推荐)

git clone -b v23.05.2 https://git.openwrt.org/openwrt/openwrt.git

**二、交叉编译为相应架构**

1.进入目录

cd openwrt

2.更新并安装Feeds

./scripts/feeds update -a

./scripts/feeds install -a

3.打开菜单配置

make menuconfig

4.选择Target System

Subtarget

Target Profile

举例 ：编译给 ARM (例如 Raspberry Pi 4)

Target System -> Broadcom BCM27xx

Subtarget -> BCM2711 boards (64 bit)

Target Profile -> Raspberry Pi 4

配置 LuCI (Web 管理界面):

默认编译出来的固件可能不带 Web 界面。如果需要，请前往：

LuCI -> Collections -> 勾选 `luluci

5.编译和下载

Make download

6.单线程编译

make -j1 V=s

.编译成功后文件位于bin/targets/<Target_System>/<Subtarget>/

会出现一些.img.gz的镜像

