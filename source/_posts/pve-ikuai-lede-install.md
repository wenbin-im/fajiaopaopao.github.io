title: n3540软路由安装pve+ikuai+openwrt(lede)双软路由系统记录
date: 2020-12-13
categories:
- 软路由
tags:
- OpenWrt
- Lede
- iKuai
language: zh-CN
toc: true
cover: /gallery/covers/openwrt-logo.webp
thumbnail: /gallery/covers/openwrt-logo-thumbnail.png
---

爱快+OpenWrt是目前比较流行的双软路由方案。我具体方案是：1.爱快作为拨号和控流、Nas外网映射；2.Lede插件市场丰富作为旁路由，来解决404网站访问，广告屏蔽；3.网件路由器作为中继AP接管有线设备、WiFi设备的网络接入。
<!-- more -->

## 硬件设备
- 某宝N3540软路由+4G内存条+64G SSD
- NETGEAR 6800
- 群辉 DS220+

## 软件工具
- PVE镜像：https://pve.proxmox.com/wiki/Main_Page
- iKuai固件：https://www.ikuai8.com/component/download
- HomeLede三方固件：https://github.com/xiaoqingfengATGH/HomeLede/wiki

> 选择使用第三方OpenWrt固件原因是：HomeLede内置内置了科学工具、AdGuard、DNS等刚需解决方案，就懒得自己去折腾了。

## 功能需求
- 科学访问：电脑、手机等设备能够无限访问外网
- 广告拦截：多设备上对去除广告，禁止用户追踪，防止DNS污染
- NAS外网映射：利用DDNS将NAS的应用端口映射到外网，可以远程访问和管控
- 网络共享：内网设备之间能够网络共享，备份Mac Time Machine到NAS中

## 网络拓扑
具体如何在软路由上安装PVE以及iKuai、Lede这里就不详细给出了，网上相关的文章已经很多了，自己根据自己需求搜索相关教程即可，下面是根据我设备和需求情况画出的网络拓补图：
![](/assets/2020-12-13/openwrt-network.png)
