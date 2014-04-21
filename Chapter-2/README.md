## 第2章: 建立开发环境


第一步是建立一个可用且好用的开发环境。通过Vagrant和Virtualbox, 可以在其他所有操作系统（Linux，Windows或者Mac）下，对你的操作系统进行编译和测试。

### 安装Vagrant
> Vagrant是一个自由开源软件, 用于创建和配置虚拟开发环境. 可以用来“包装”VirtualBox.

Vagrant可以帮助你在所使用的任何操作系统下，创建一个干净的虚拟开发环境。第一步是从[http://www.vagrantup.com/](http://www.vagrantup.com/)下载安装Vagrant。

### 安装Virtualbox

> Oracle VM Virtualbox是用于x86和AMD64/Intel64计算机的虚拟化软件包

Vagrant需要Virtualbox来运行。从[https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)下载并安装。

### 运行并测试开发环境

安装好Vagrant和Virtualbox后，你需要为Vagrant下载ubuntu lucid32的镜像。

```
vagrant box add lucid32 http://files.vagrantup.com/lucid32.box
```

准备好lucid32镜像后，需要使用一个*Vagrantfile*来定义开发环境。[创建*Vagrantfile*文件](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/Vagrantfile)。这个文件定义了开发环境的预设需求：nasm、make、build-essential、grub和qemu。

（译注：国内用户如若网络不太好，可以考虑修改Vagrantfile修改源）

启动Virtualbox：

```
vagrant up
```

现在可以通过ssh连接到Virtualbox

```
vagrant ssh
```

代码放到*/vagrant*目录下使用

```
cd /vagrant
```

#### 编译并测试操作系统

这个[**Makefile**](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/Makefile) 定义了一些用于编译kernel, 用户libc和其他userland程序的基本规则.

编译：

```
make all
```

使用qemu来测试:

```
make run
```

qemu的文档可以看这里[QEMU Emulator Documentation](http://wiki.qemu.org/download/qemu-doc.html).

可以使用\<Ctrl-a x\>来退出模拟器