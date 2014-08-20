---
layout: default
title: Ubuntu驱动USB视频采集卡（一）
---
 - 需求描述

在学习处理视频的过程中，用自己的Sony T9卡片相机录像，存储卡只有1GB，仅仅能录制10～15分钟左右的视频，不够用，发现相机的视频线还在，就产生了将视频通过采集卡传到OpenCV程序中的念头。后来又搞到了几个监控摄像头，这个想法更加强烈了。

 - 查阅资料

视频采集卡，我倾向于USB接口，便捷，通用。关于Ubuntu系统能驱动的采集卡，[有这个权威网页][1]，看完的结论是，按照页面列出的顺序选购。[在淘宝上买到了这个][2]，属于页面中的“2.3 Somagic"类型。还找到了[驱动程序][3]。

 - 资源限制

需要指出的是，该款采集卡虽然有四路视频输入和一路音频输入，但Ubuntu下貌似音频采集不通。四路视频也不能同时工作，只能单路选通。最后就是USB总线限制，该采集卡是USB2.0接口，在电脑的USB2.0总线上，只能有一个设备工作，通俗来说，如果笔记本有自带的摄像头，很有可能摄像头和采集卡不能同时工作。这点已经实践过。


 - 安装驱动程序

1. 在Windows平台上获取驱动程序。买回来的采集卡附带一张光盘，里面有DriverSetup.exe程序，安装后，在目录3F/xp中，提取出sys文件。
2. 安装somagic-easycap_1.1_amd64.deb和somagic-easycap-tools_1.1_amd64.deb。
3. 用somagic-extract-firmware从sys文件中提取驱动固件到默认目录。
4. 卸载模块：sudo modprobe -r usbhid
5. 用somagic-init初始化设备，得到lsusb输出：1c88:003f

 - 采集视频

1. 用somagic-capture捕获通道1的图像，并保存到文件：

        sudo somagic-capture -i 1 --vo a.raw
    
2. 捕获并实时观看：

        sudo somagic-capture -i 1 | mplayer -vf yadif,screenshot -demuxer rawvideo -rawvideo "pal:format=uyvy:fps=25" -aspect 4:3 -

  [1]: http://www.linuxtv.org/wiki/index.php/Easycap "EasyCAP系列USB采集卡介绍"
  [2]: http://item.taobao.com/item.htm?spm=a1z09.2.9.38.ZI60Ik&id=16080303910&_u=k54ac4f8542 "easycap 4路USB采集卡 4路USB视频采集卡 4路监控采集卡 支持win7"
  [3]: https://code.google.com/p/easycap-somagic-linux "Somagic的Ubuntu驱动程序"

