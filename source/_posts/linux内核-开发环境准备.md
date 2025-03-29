---
title: 开发环境准备
date: 2025/03/15 00:00:00
cover: /images/covers/01-blog.jpg
thumbnail: /images/covers/01-blog.jpg
toc: true
categories: 
 - linux内核
---



良好的开发环境可以让内核的学习事半功倍。

由于内核的特殊性，它不像普通的可执行程序那样可以直接`./xxx`运行，而是需要借助外围工具qemu启动。此外，在启动过程中，内核还需要另一个很重要的文件——rootfs，来为其挂载各类虚拟文件系统和提供shell会话。故在搭建开发环境时，我们需要分别准备：编译好的内核镜像、rootfs文件以及qemu启动脚本。

<!-- more -->

## 编译内核

先下载内核源码并编译，以获得内核镜像文件。

当前内核源码分别在web.git.kernel.org、github等平台上有托管。其源码被fork有多份，如[torvalds/linux](https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)、[stable/linux](https://web.git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)等，一般我们常说的mainline主线指的是前者，也就是linus本人维护的torvalds/linux仓。

我们可以通过git clone的方式来下载源码：

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

> 当然，你也可以选择直接下载内核的[tar.gz](https://www.kernel.org/)压缩包。好处是包体积较小，下载的快些；缺点是下载下来的包内没有git commit日志。内核的很大一部分文档其实就在commit日志内，因此对于学习内核而言这种下载压缩包的方式不推荐。

由于源码体积较大（1G以上），且国内的一些网络限制，下载可能不会那么顺利，必要时可以切国内的镜像源：

```bash
git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
```



内核源码下载下来后，就可以开始编译了。编译命令如下所示：

```bash
# 一些必要的依赖
apt install make flex bison libncurses-dev python3 libelf-dev libssl-dev
# 进入内核源码目录
cd linux/
# 生成默认的.config文件
make defconfig
# （可选）如有想要修改某些config选项，可打开menu窗口修改（打开后通过"/"快捷键可帮助快速搜索定位config所在的菜单位置）
make menuconfig
make -j8
```

如编译成功，将获得几个产物：

1. 内核镜像：位于`arch/<架构名称>/boot/`路径下，镜像文件名称在不同架构下可能会有些许不同，以x86_64为例，其名称为bzImage；
2. 调试文件：vmlinux，位于源码根路径下。后续调试内核会用到。



## 准备rootfs

rootfs的制作方式有很多，有通过busybox的，也有借助buildroot的，笔者这里直接采用ubuntu base镜像来做。

首先下载ubuntu base镜像并解压，获得`ubuntu-base-22.04.4-base-amd64.tar`：

```bash
wget https://cdimage.ubuntu.com/ubuntu-base/releases/22.04.4/release/ubuntu-base-22.04.4-base-amd64.tar.gz
gzip -d ubuntu-base-22.04.4-base-amd64.tar.gz
```

准备一个用于承载rootfs的白板文件，格式化为ext4，这样我们下一步才能挂载它：

```bash
dd if=/dev/zero of=rootfs-x86.ext4 bs=1G count=4
mkfs.ext4 rootfs-x86.ext4
```

将ubuntu base镜像的内容导入到白板文件内：

```bash
mkdir rootfsmnt
mount rootfs-x86.ext4 rootfsmnt
cd rootfsmnt/
tar -xf ubuntu-base-22.04.4-base-amd64.tar
```

导入成功后，在mnt/下将会看到和当前OS根目录/几乎一致的文件夹（usr、proc、etc等），此时已经有了rootfs的雏形。

接下来就是丰富rootfs内的工具链了，包括启动用的systemd、基本编辑工具vim等。此步骤需要用到chroot的能力，基本思路是通过chroot将`rootfsmnt/`转为`/`根路径，然后在其内部通过apt安装相关工具。

因为涉及到联网，在切之前需把本地的dns配置文件、apt源配置文件复制到rootfs内：

```bash
cp /etc/resolv.conf rootfsmnt/etc/resolv.conf
cp /etc/apt/sources.list rootfsmnt/etc/apt/sources.list
```

复制好后，借助[ch-mount.sh](https://github.com/psachin/bash_scripts/blob/master/ch-mount.sh)脚本来进入chroot环境：

```bash
$ ./ch-mount.sh -m rootfsmnt/
MOUNTING
$ pwd
/  # 这里看到的/其实就是rootfsmnt/
```

现在可以安装任意工具命令了（所有的依赖包安装都只会发生在rootfs内，不会影响到外部环境）：

```bash
apt update
apt install systemd vim
```

后续的qemu启动我们会用到串口ttyS0，ubuntu base镜像默认没有将串口关联到ttyS0，故需要修正：

```bash
ln -s /lib/systemd/system/serial-getty\@.service /etc/systemd/system/getty.target.wants/serial-getty\@ttyS0.service
```

（可选）删除root密码，此步骤是为了避免每次qemu启动内核时频繁输入密码：

```bash
passwd -d root
```

完成上述步骤后，就可以退出chroot环境了：

```bash
$ exit
exit
$ ./ch-mount.sh -u rootfsmnt/
UNMOUNTING
$ umount rootfsmnt
```

此时`rootfs-x86.ext4`文件即可用于配合内核镜像启动。



## 启动内核

启动内核需用到qemu。

ubuntu提供了各种架构的qemu命令，如qemu-system-x86_64、qemu-system-aarch64等，可通过apt install安装：`apt install qemu-system-x86_64`。这是不是意味着我们可以在一台机器上实现各类架构的内核镜像调试？—— 答案是肯定的。内核本身提供了交叉编译的参数`ARCH`和`CROSS_COMPILE`，我们只需将qemu、gcc、rootfs三者的架构对齐，即可调试各类镜像。

以x86机器为例，如果想调试aarch64的内核，一般会这么做：

1. 制作好指定架构的rootfs：步骤同前面章节，唯一要变更的是ubuntu base镜像，需找指定架构的镜像来做；
2. 准备aarch64版本的gcc编译器：`apt install gcc-aarch64-linux-gnu`；
3. 编译出指定架构的镜像：`make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`；
4. 最后配合同样架构的qemu，从而实现内核启动：`qemu-system-aarch64 ...`

x86内核的qemu启动命令如下：

```bash
qemu-system-x86_64 \
        -smp 1 \
        -m 1g \
        -kernel arch/x86/boot/bzImage \
        -append "root=/dev/sda rw console=ttyS0 init=/bin/systemd" \
        -hda rootfs-x86.ext4 \
        -nographic
```



## 调试内核

内核是支持gdb调试的，具体为：

1、qemu命令追加`-s -S`参数。其中：

- `-s`用于开启调试端口1234；
- `-S`用于让内核陷入等待直至调试端口1234连上后才能启动，这通常对于想要调试内核启动过程的一些问题很有用

2、gdb连接调试端口（可以看到，前文编译内核时第二个产物vmlinux在这里发挥了作用，有了它才能断点追踪到内核源码）：

```bash
$ gdb vmlinux
(gdb) target remote :1234
```



## 配置智能提示

配置智能提示的目的为了让函数跳转、结构体定义查询得更准确，以支持更好的阅读内核源码。以下是笔者vscode的配置，可参考：

.vscode/settings.json

```json
{
    "editor.rulers": [
        80
    ],
    "C_Cpp.default.includePath": [
        "${workspaceFolder}/**",
        "${workspaceFolder}/include",
        "${workspaceFolder}/arch/x86/include",
        "${workspaceFolder}/arch/x86/include/generated",
        "/usr/include",
        "/usr/local/include"
    ],
    "C_Cpp.default.defines": [
        "CONFIG_CC_IS_GCC=1",
        "CONFIG_AS_IS_GNU=1",
        "CONFIG_LD_IS_BFD=1",
        "__KERNEL__",
        "__GNUC__"
    ],
    "editor.tabSize": 8,
    "C_Cpp.default.cStandard": "c11",
    "C_Cpp.default.intelliSenseMode": "linux-gcc-x64",
    "files.exclude": {
        "**/*.o": true,
        "**/.*.*.cmd": true
    }
}
```

其中，"C_Cpp.default.defines"的`CONFIG_XXX`对应到内核根路径下的`.config`文件内选项，这些选项往往在内核开发调试过程中会频繁变更，为避免手动对齐的麻烦，可通过以下脚本来刷新"C_Cpp.default.defines"：

.vscode/sync-config.py

```python
# extract kernel configs for .config
configs = []
with open('../.config', 'r') as f:
        for line in f.readlines():
                line = line.strip()
                if not line.startswith('CONFIG'):
                        continue
                if not line.endswith('=y') or line.endswith('=m'):
                        continue
                conf = line.replace('=y', '=1').replace('=m', '=1')
                configs.append(conf)


# insert configs into settings.json
settings_json_content = []
with open('settings.json', 'r') as f:
        offset = 1
        find_defines = False
        inserted = False
        for line in f.readlines():
                # find the 'defines' line
                if not find_defines and 'C_Cpp.default.defines' not in line:
                        settings_json_content.append(line)
                        continue
                find_defines = True
                # shift the offset of 'defines' line
                if offset > 0:
                        offset -= 1
                        settings_json_content.append(line)
                        continue
                # skip the previous configs
                if 'CONFIG_' in line:
                        continue
                # insert configs
                if not inserted:
                        inserted = True
                        for conf in configs:
                                conf_str = f'        "{conf}",\n'
                                settings_json_content.append(conf_str)
                # append the rest content normally
                settings_json_content.append(line)


# save settings.json
with open('settings.json', 'w') as f:
        for line in settings_json_content:
                f.write(line)
```

