**一、准备工作：**

1.Linux环境，我使用的是我服务器ubuntu22.04，也可以使用虚拟机

2.下载Busybox源码，推荐1.36.1

对应下载命令为：wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2

3.安装对应架构交叉编译器：

我们以ARM64，小端MIPS32为例，其安装命令分别为：

sudo apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu

sudo apt-get install gcc-mipsel-linux-gnu

**二、交叉编译为相应架构**

1.命令行输入make menuconfig会显示如下界面：

![](file:///C:\Users\admin\AppData\Local\Temp\ksohtml12456\wps1.jpg) 

2.打开Settings -> Build static binary (no shared libs) 勾选Y

3.交叉编译时Settings -> Cross compiler prefix 设为相应架构的工具链前缀，如交叉编译模板为ARM64，则工具链前缀为aarch64-linux-gnu-

下表给出了各类架构对应的工具链前缀及安装命令：

|   |   |   |
|---|---|---|
|**架构类型**|**工具链前缀**|**编译工具安装命令(Ubuntu)**|
|**ARM架构系列**|||
|ARMv5 (软浮点)|arm-linux-gnueabi-|sudo apt-get install gcc-arm-linux-gnueabi|
|ARMv7 (硬浮点)|arm-linux-gnueabihf-|sudo apt-get install gcc-arm-linux-gnueabihf|
|ARM64 (AArch64)|aarch64-linux-gnu-|sudo apt-get install gcc-aarch64-linux-gnu|
|**MIPS架构系列**|||
|MIPS32 大端|mips-linux-gnu-|sudo apt-get install gcc-mips-linux-gnu|
|MIPS32 小端|mipsel-linux-gnu-|sudo apt-get install gcc-mipsel-linux-gnu|
|MIPS64 大端|mips64-linux-gnuabi64-|sudo apt-get install gcc-mips64-linux-gnuabi64|
|MIPS64 小端|mips64el-linux-gnuabi64-|sudo apt-get install gcc-mips64el-linux-gnuabi64|
|**PowerPC架构系列**|||
|PowerPC 32位大端|powerpc-linux-gnu-|sudo apt-get install gcc-powerpc-linux-gnu|
|PowerPC 64位大端|powerpc64-linux-gnu-|sudo apt-get install gcc-powerpc64-linux-gnu|
|PowerPC 64位小端|powerpc64le-linux-gnu-|sudo apt-get install gcc-powerpc64le-linux-gnu|

打开Networking Utilities -> tc (8.3kb) 取消勾选tc，然后保存重新编译

4.然后保存并退出，执行命令make && make install

5.得到生成文件夹_install，其中bin目录包含busybox可执行文件

6.输入命令 readelf -h busybox | grep Machine即可验证所生成elf文件是否为对应架构

7. 使用qemu进行验证：

 qemu-aarch64-static ./busybox-arm64

![](file:///C:\Users\admin\AppData\Local\Temp\ksohtml12456\wps2.jpg) 

结果显示正常可用

**三、集成式shell工具**

我们将以上交叉编译过程集成到一个shell脚本中，如附件所示auto_busybox_compile.sh

直接输入不带参数后缀或者-h会提示用法：

![](file:///C:\Users\admin\AppData\Local\Temp\ksohtml12456\wps3.jpg) 

按照要求可以直接输出对应架构的ELF文件，如果报错或缺少依赖需按照对应错误安装相应的依赖。