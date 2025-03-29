---
title: 开发环境准备
date: 2025/03/15 00:00:00
cover: /images/covers/01-blog.jpg
thumbnail: /images/covers/01-blog.jpg
toc: true
categories: 
 - linux内核
---

搭建一次内核开发环境不易，本文完整记录内核开发环境搭建过程，以供后续需要时使用。

<!-- more -->

## 下载内核源码

国内下载可以使用清华源

```bash
git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
```



## 编译内核

```bash
make defconfig
make
```

过程可能有一些依赖包（如flex、bison等）需安装，根据提示安装即可。



## 准备rootfs

1、下载ubuntu base镜像（网上各种busybox准备/proc、/sys等操作还不如直接使用该镜像来的方便，啥都有）

```bash
wget https://cdimage.ubuntu.com/ubuntu-base/releases/22.04.4/release/ubuntu-base-22.04.4-base-amd64.tar.gz
```

2、提供rootfs白板文件

```bash
dd if=/dev/zero of=rootfs-x86.ext4 bs=1G count=4
mkfs.ext4 rootfs-x86.ext4
```

3、将base镜像内容导入到白板文件内

```bash
mkdir mnt
mount rootfs-x86.ext4 mnt
gzip -d ubuntu-base-22.04.4-base-amd64.tar.gz
cd mnt
tar -xf ubuntu-base-22.04.4-base-amd64.tar
```

4、拷贝必要文件，为后续安装依赖包做准备

```bash
cp /etc/resolv.conf mnt/etc/resolv.conf
cp /etc/apt/sources.list mnt/etc/apt/sources.list
```

5、进入chroot环境（后面安装依赖包的操作需在该环境下完成）

借助ch-mount.sh脚本：https://github.com/psachin/bash_scripts/blob/master/ch-mount.sh

```bash
./ch-mount.sh -m mnt/
```

6、安装依赖包

```bash
apt update
apt install systemd
apt install vim
```

7、修正tty显示（后面启动内核要用到ttyS0）

```bash
ln -s /lib/systemd/system/serial-getty\@.service /etc/systemd/system/getty.target.wants/serial-getty\@ttyS0.service
```

8、删除root密码（图方便）

```bash
passwd -d root
```

9、退出chroot环境

```bash
exit
./ch-mount.sh -u mnt/
umount mnt
```



## 启动内核

```bash
qemu-system-x86_64 \
        -smp 1 \
        -m 1g \
        -kernel arch/x86/boot/bzImage \
        -append "root=/dev/sda rw console=ttyS0 init=/bin/systemd" \
        -hda rootfs-x86.ext4 \
        -nographic
```



## 配置智能提示（vscode）

settings.json内容如下：

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

"C_Cpp.default.defines"内的config可能会在开发时有调整，可通过以下python脚本刷新settings.json

file: .vscode/sync-config.py

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

刷新脚本执行方法：

```bash
cd .vscode
python3 sync-config.py
```

