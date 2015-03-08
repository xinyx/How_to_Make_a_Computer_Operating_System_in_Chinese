## 第二章: 建立开发环境

第一步是设置一个好的和可行的开发环境. 使用Vagrant和Virtualbox, 你可以在所有系统(Linux, Windows 或 Mac)中编译和测试你的操作系统.

### 安装 Vagrant
> Vagrant是创建和配置虚拟开发环境的免费和开源软件. 它可以被视为VirtualBox的包装器.ox.

Vagrant能帮我们在你使用的任何系统中创建一个干净的虚拟开发环境

第一步是从 http://www.vagrantup.com/ 下载和安装Vagrant

### 安装 Virtualbox

> Oracle VM VirtualBox是为x86和基于ADM64/Intel64的电脑设计的虚拟化软件包

Vagrant需要Virtualbox才能工作, 从https://www.virtualbox.org/wiki/Downloads 可以下载和安装与你操作系统对应的版本

### 开始并测试你的开发环境

Vagrant和VirtualBox安装好之后, 你需要为Vagrant下载ubuntu lucid32镜像:

```
vagrant box add lucid32 http://files.vagrantup.com/lucid32.box
```

lucid32镜像准备好之后, 我们需要用一个 *Vagrantfile* 文件定义我们的开发环境, [新建一个名为 *Vagrantfile* 的文件](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/Vagrantfile).这个文件定义了我们所需预先部署的环境: nasm, make, build-essential, grub and qemu.

这样开启你的box:

```
vagrant up
```

现在你可以通过使用ssh来连接virtual box, 进而访问你的box:

```
vagrant ssh
```

把所有文件放到Vagrantfile所在的目录, 它存在于虚拟机的 */vagrant* 目录. (在这个例子, Ubuntu Lucid32): 

```
cd /vagrant
```

#### 建立并测试你的操作系统

[**Makefile**](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/Makefile) 文件定义了一些构建内核, 用户libc和用户程序的基本的规则.

构建:

```
make all
```

用qemu测试我们的操作系统

```
make run
```

可以从[qemu模拟器文档](http://wiki.qemu.org/download/qemu-doc.html)中获得qemu的相关文档.

可以这样退出模拟器: Ctrl-a.
