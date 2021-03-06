Systemd服务：响应变更
======

![](https://www.linux.com/sites/lcom/files/styles/rendered_file/public/webcam.png?itok=zzYUs5VK)

[我有一个这样的电脑棒][1]（图1），并将其用作通用服务器。它很小且安静，由于它是基于x86架构，因此我为我的打印机安装驱动没有任何问题，而且这就是它大多数时候干的事：与客厅的共享打印机和扫描仪通信。

![ComputeStick][3]

一个英特尔电脑棒。欧元硬币大小。

[Used with permission][4]

大多数时候，它都闲着，尤其是当我们外出时，因此我认为用它作监视系统是个好主意。该设备没有自带的摄像头，也不需要一直监视。我也不想手动启动图像捕获，因为这样就意味着在出门前必须通过SSH登录，并在shell中编写命令来启动该进程。

因此，我以为应该这么做：抓住USB摄像头，然后只需插入它即可自动启动监视系统。如果Stick重启后发现连接了摄像头也启动监视系统就更加分了。

在先前的文章中，我们看到[systemd服务可以手动启动或停止][5]或[在满足某些条件时][6]。这些条件不限于操作系统在启动或关机时序中达到某种状态，还可以在您插入新硬件或文件系统发生变化时进行。您可以通过将Udev规则与systemd服务结合起来实现。

### 有Udev(支持)的热插拔

Udev规则位于 _/etc/udev/rules_ 目录中，通常是由导致一个 _动作(action)_ 的 _条件(conditions)_ 和 _赋值(assignments)_ 的单行语句来描述。

有点神秘。让我们再试一次：

通常，在Udev规则中，您告诉systemd当连接一个设备时需要查看什么信息。例如，您可能想检查刚插入的设备的品牌和型号是否与您让Udev等待的设备的品牌和型号相对应。这些就是前面提到的条件。

然后，您可能想要更改一些内容，以便以后可以轻松使用该设备。例如，更改设备的读写权限：如果插入USB打印机，您将希望用户能够从打印机读取信息（用户的打印应用程序需要知道其模型，制造商，以及是否准备好接受打印作业）并向其写入内容，即发送要打印的内容。更改设备的读写权限是通过您之前阅读的 _赋值(assignments)_ 之一完成的。

最后，您可能希望系统在满足上述条件时执行某些动作，例如在插入某个外接硬盘时启动备份程序以复制重要文件。这就是上面提到的 _动作(action)_ 的例子。

了解这些之后, 来看看以下几点:

```
ACTION=="add", SUBSYSTEM=="video4linux", ATTRS{idVendor}=="03f0", ATTRS{idProduct}=="e207",
SYMLINK+="mywebcam", TAG+="systemd", MODE="0666", ENV{SYSTEMD_WANTS}="webcam.service"
```

规则的第一部分,

```
ACTION=="add", SUBSYSTEM=="video4linux", ATTRS{idVendor}=="03f0",
ATTRS{idProduct}=="e207" [etc... ]
```

表明了执行您想让系统执行的其他动作之前设备必须满足的条件。设备必须被添加到（`ACTION=="add"`）机器上，并且必须添加到 `video4linux` 子系统中。为了确保仅在插入正确的设备时才应用该规则，您必须确保Udev正确识别设备的制造商（`ATTRS{idVendor}=="03f0"`）和型号（`ATTRS{idProduct}=="e207"`）。

在本例中，我们讨论的是这个设备（图2）：

![webcam][8]

这个试验使用的是HP的摄像头。

[Used with permission][4]

注意怎样用 `==` 来表示这是一个逻辑操作。您应该像这样阅读上面的简要规则：

```
如果添加了一个设备并且该设备由video4linux子系统控制
而且该设备的制造商是03f0，型号是e207，那么...
```

但是，您从哪里获取的这些信息？ 您在哪里找到触发事件的动作，制造商，模型等等？您可能必须使用多个来源。您可以通过将摄像头插入机器并运行 `lsusb` 来获得 `IdVendor` 和 `idProduct` ：

```
lsusb
Bus 002 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 003: ID 03f0:e207 Hewlett-Packard
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 003: ID 04f2:b1bb Chicony Electronics Co., Ltd
Bus 001 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

我用的摄像头是HP的，您在上面的列表中只能看到一个HP设备。 `ID` 告诉了制造商和型号，它们以冒号（ `:` ）分隔。如果您有同一制造商的多个设备，不确定哪个是哪个设备，请拔下摄像头，再次运行 `lsusb` , 看看少了什么。

或者...

拔下摄像头，等待几秒钟，运行命令 `udevadmin monitor --environment` ，然后重新插入摄像头。当您使用的是HP摄像头时，您将看到：

```
udevadmin monitor --environment
UDEV [35776.495221] add /devices/pci0000:00/0000:00:1c.3/0000:04:00.0
  /usb3/3-1/3-1:1.0/input/input21/event11 (input)
.MM_USBIFNUM=00
ACTION=add
BACKSPACE=guess
DEVLINKS=/dev/input/by-path/pci-0000:04:00.0-usb-0:1:1.0-event
  /dev/input/by-id/usb-Hewlett_Packard_HP_Webcam_HD_2300-event-if00
DEVNAME=/dev/input/event11
DEVPATH=/devices/pci0000:00/0000:00:1c.3/0000:04:00.0/
  usb3/3-1/3-1:1.0/input/input21/event11
ID_BUS=usb
ID_INPUT=1
ID_INPUT_KEY=1
ID_MODEL=HP_Webcam_HD_2300
ID_MODEL_ENC=HP\x20Webcam\x20HD\x202300
ID_MODEL_ID=e207
ID_PATH=pci-0000:04:00.0-usb-0:1:1.0
ID_PATH_TAG=pci-0000_04_00_0-usb-0_1_1_0
ID_REVISION=1020
ID_SERIAL=Hewlett_Packard_HP_Webcam_HD_2300
ID_TYPE=video
ID_USB_DRIVER=uvcvideo
ID_USB_INTERFACES=:0e0100:0e0200:010100:010200:030000:
ID_USB_INTERFACE_NUM=00
ID_VENDOR=Hewlett_Packard
ID_VENDOR_ENC=Hewlett\x20Packard
ID_VENDOR_ID=03f0
LIBINPUT_DEVICE_GROUP=3/3f0/e207:usb-0000:04:00.0-1/button
MAJOR=13
MINOR=75
SEQNUM=3162
SUBSYSTEM=input
USEC_INITIALIZED=35776495065
XKBLAYOUT=es
XKBMODEL=pc105
XKBOPTIONS=
XKBVARIANT=
```

可能看起来有很多信息要处理，但是，看一下这个：列表前面的 `ACTION` 字段, 它告诉您刚刚发生了什么事件，即一个设备被添加到系统中。您还可以在几行中看到设备名称的拼写，因此可以非常确定它就是您要找的设备。输出里还显示了制造商的ID（`ID_VENDOR_ID = 03f0`）和型号（`ID_VENDOR_ID = 03f0`）。

这为您提供了规则条件部分需要的四个值中的三个。您可能也会想到它还给了您第四个，因为还有一行这样写道：

```
SUBSYSTEM=input
```

小心！尽管USB摄像头确实是提供输入的设备（键盘和鼠标也是），但它也属于 _usb_ 子系统和其他几个子系统。这意味着您的摄像头被添加到了多个子系统，并且看起来像多个设备。如果您选择了错误的子系统，那么您的规则可能无法按您期望的那样工作，或者根本无法工作。

因此，您必须检查的第三件事是摄像头添加到的所有子系统，并选择正确的那个。为此，请再次拔下摄像头，然后运行：

```
ls /dev/video*
```

这将向您显示连接到本机的所有视频设备。如果您使用的是笔记本，大多数笔记本都带有内置摄像头，它可能会显示为 `/dev/video0` 。重新插入摄像头，然后再次运行 `ls /dev/video*` 。

现在，您应该看到多一个视频设备（可能是`/dev/video1`）。

现在，您可以通过运行`udevadm info -a /dev/video1`找出它所属的所有子系统：

```
udevadm info -a /dev/video1

Udevadm info starts with the device specified by the devpath and then
walks up the chain of parent devices. It prints for every device
found, all possible attributes in the udev rules key format.
A rule to match, can be composed by the attributes of the device
and the attributes from one single parent device.

 looking at device '/devices/pci0000:00/0000:00:1c.3/0000:04:00.0
  /usb3/3-1/3-1:1.0/video4linux/video1':
 KERNEL=="video1"
 SUBSYSTEM=="video4linux"
 DRIVER==""
 ATTR{dev_debug}=="0"
 ATTR{index}=="0"
 ATTR{name}=="HP Webcam HD 2300: HP Webcam HD"

[etc...]
```

输出持续了一段时间，但是您感兴趣的只是开始：`SUBSYSTEM =="video4linux"`。您可以将这行按文本复制并粘贴到规则中。输出的其余部分（为简便起见未显示）为您提供了更多的块，例如制造商和模型ID，您也可以以同样的格式复制并粘贴到规则中。

现在，您有了识别设备的方式并明确了什么事件应该触发该动作，该对设备进行修改了。

规则的下一部分，`SYMLINK+="mywebcam", TAG+="systemd", MODE="0666"`告诉Udev做三件事：首先，您要创建设备的符号链接（例如 _/dev/video1_ 到 _/dev/mywebcam_ 。这是因为您无法预测系统默认情况下会把那个设备叫什么。当您拥有内置摄像头并热插拔一个新的时，内置摄像头通常为 _/dev/video0_ ，而外部摄像头通常为 _/dev/video1_ 。但是，如果您在插入外部USB摄像头的情况下重启计算机，则可能会相反，内部摄像头可能会变成 _/dev/video1_ ，而外部摄像头会变成 _/dev/video0_ 。这想告诉您的是，尽管您的图像捕获脚本（稍后将看到）总是需要指向外部摄像头设备，但是您不能依赖它是 _/dev/video0_ 或 _/dev/video1_ 。为了解决这个问题，您告诉Udev创建一个符号链接，该链接在设备被添加到 _video4linux_ 子系统的那一刻起就不会再变，您将使您的脚本指向该链接。

您要做的第二件事是将 `"systemd"` 添加到与此规则关联的Udev标记列表中。这告诉Udev，该规则触发的动作将由systemd管理，即它将是某种systemd服务。

注意在两种情况下该如何使用 `+=` 运算符。这会将值添加到列表中，这意味着您可以向 `SYMLINK` 和 `TAG` 添加多个值。

另一方面，`MODE` 值只能包含一个值（因此，您可以使用简单的 `=` 赋值运算符）。`MODE` 的作用是告诉Udev谁可以读或写该设备。如果您熟悉 `chmod`（您读到此文, 应该会熟悉），您就也会熟悉[如何用数字表示权限][9]。这就是它的含义： `0666` 的含义是 “ _向所有人授予对设备的读写权限_ ”。

最后， `ENV{SYSTEMD_WANTS}="webcam.service"` 告诉Udev要运行什么systemd服务。

将此规则保存到 _/etc/udev/rules.d_ 目录名为 _90-webcam.rules_ (或类似的名称)的文件中，您可以通过重启机器或运行以下命令来加载它：

```
sudo udevadm control --reload-rules && udevadm trigger
```

## 最后描述服务

Udev规则触发的服务非常简单：
```
# webcam.service

[Service]
Type=simple
ExecStart=/home/[user name]/bin/checkimage.sh
```

基本上，它只是运行存储在您个人 _bin/_ 中的 _checkimage.sh_ 脚本并将其放到后台。 [这是您在先前的部分中看过的内容][5]。 它看起来似乎很小，但那只是因为它是被Udev规则调用的，您刚刚创建了一种特殊的systemd单元，称为 _device_ 单元。 恭喜。

至于 _webcam.service_ 调用的 _checkimage.sh_ 脚本，有几种方法从摄像头抓取图像并将其与前一个图像进行比较以检查变化（这是 _checkimage.sh_ 所做的事），但这是我的方法：

```
#!/bin/bash
# This is the checkimage.sh script

mplayer -vo png -frames 1 tv:// -tv driver=v4l2:width=640:height=480:device=
  /dev/mywebcam &>/dev/null
mv 00000001.png /home/[user name]/monitor/monitor.png

while true
do
 mplayer -vo png -frames 1 tv:// -tv driver=v4l2:width=640:height=480:device=/dev/mywebcam &>/dev/null
 mv 00000001.png /home/[user name]/monitor/temp.png

 imagediff=`compare -metric mae /home/[user name]/monitor/monitor.png /home/[user name]
  /monitor/temp.png /home/[user name]/monitor/diff.png 2>&1 > /dev/null | cut -f 1 -d " "`
 if [ `echo "$imagediff > 700.0" | bc` -eq 1 ]
 then
 mv /home/[user name]/monitor/temp.png /home/[user name]/monitor/monitor.png
 fi

 sleep 0.5
done
```

首先使用[MPlayer][10]从摄像头抓取一帧（_00000001.png_）。注意，我们怎样将 `mplayer` 指向Udev规则中创建的 `mywebcam` 符号链接，而不是指向 `video0` 或 `video1` 。然后，将图像传输到主目录中的 _monitor/_ 目录。然后执行一个无限循环，一次又一次地执行相同的操作，但还使用了[Image Magick的_compare_工具][11]来查看最后捕获的图像与 _monitor/_ 目录中已有的图像之间是否存在差异。

如果图像不同，则表示摄像头的镜框里某些东西动了。该脚本将新图像覆盖原始图像，并继续比较以等待更多变动。

### 插线

所有东西准备好后，当您插入摄像头后，您的Udev规则将被触发并启动 _webcam.service_ 。 _webcam.service_ 将在后台执行 _checkimage.sh_ ，而 _checkimage.sh_ 将开始每半秒拍一次照。您会感觉到，因为摄像头的LED在每次拍照时将开始闪。

与往常一样，如果出现问题，请运行

```
systemctl status webcam.service
```

检查您的服务和脚本正在做什么。

### 接下来

您可能想知道：为什么要覆盖原始图像？ 当然，系统检测到任何动静，您都想知道发生了什么，对吗？您是对的，但是如您在下一部分中将看到的那样，将它们保持原样，并使用另一种类型的systemd单元处理图像将更好，更清晰和更简单。

请稍等。

通过Linux基金会和edX的免费["Linux简介"][12]课程了解有关Linux的更多信息。
--------------------------------------------------------------------------------

via: https://www.linux.com/blog/intro-to-linux/2018/6/systemd-services-reacting-change

作者：[Paul Brown][a]
选题：[lujun9972][b]
译者：[messon007](https://github.com/messon007)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]: https://www.linux.com/users/bro66
[b]: https://github.com/lujun9972
[1]: https://www.intel.com/content/www/us/en/products/boards-kits/compute-stick/stk1a32sc.html
[2]: https://www.linux.com/files/images/fig01png
[3]: https://www.linux.com/sites/lcom/files/styles/floated_images/public/fig01.png?itok=cfEHN5f1 (ComputeStick)
[4]: https://www.linux.com/licenses/category/used-permission
[5]: https://www.linux.com/blog/learn/intro-to-linux/2018/5/writing-systemd-services-fun-and-profit
[6]: https://www.linux.com/blog/learn/2018/5/systemd-services-beyond-starting-and-stopping
[7]: https://www.linux.com/files/images/fig02png
[8]: https://www.linux.com/sites/lcom/files/styles/floated_images/public/fig02.png?itok=esFv4BdM (webcam)
[9]: https://chmod-calculator.com/
[10]: https://mplayerhq.hu/design7/news.html
[11]: https://www.imagemagick.org/script/compare.php
[12]: https://training.linuxfoundation.org/linux-courses/system-administration-training/introduction-to-linux
