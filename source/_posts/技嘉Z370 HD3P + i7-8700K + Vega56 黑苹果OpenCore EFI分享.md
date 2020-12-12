title: 技嘉Z370 HD3P + i7-8700K + Vega56 黑苹果OpenCore EFI分享
date: 2020-12-12
categories:
- 黑苹果
tags:
- 黑苹果
language: zh-CN
toc: true
cover: /gallery/covers/macbook-pro-bigsur.png
thumbnail: /gallery/covers/macbook-pro-bigsur.png
---

之前折腾的Catalina正常运行了大概快一年左右时间了，作为日常生产力机器使用没什么问题，但随着最近BigSur正式版的推出，又尝试跟进升级下我的黑苹果，使用了一个礼拜，暂时没发现问题，借此把我用的EFI做个分享吧。
<!-- more -->

## 使用版本
- macOS 11.0.1

## 硬件配置
|  规格   | 详情  |
|  ----  | ----  |
| 主板  | 技嘉 Z370 HD3P |
| CPU  | Intel Core i7-8700K |
| 核显  | Intel UHD Graphics 630 |
| 独显  | RX VEGA56 8G HBM2 超白金 OC |
| 内存卡  | 海盗船复仇者 DDR4 3200 8G*2 |
| SSD1  | 英睿达 BX500 480G => Windows + EFI |
| SSD2  | 西部数据 Black系统 250GB => MacOS |
| HDD  | 希捷 酷鱼系列 2TB => Storage + Time Machine |
| 板载网卡 | Realtek ALC1220 |
| 板载网卡 | Intel I219V2 PCI Express Gigabit Ethernet |
| 无线网卡&蓝牙 | 淘宝免驱BCM943602CS |

## 必备软件
- 镜像
  - [macOS Big Sur 11.0.1 20B50 Installer](https://blog.daliansky.net/macOS-BigSur-11.0.1-20B29-Release-version-with-Clover-5126-original-image-Double-EFI-Version-UEFI-and-MBR.html)

- 工具
  - 优盘镜像写入工具：[Etcher](https://www.balena.io/etcher/)
  - OpenCore配置编辑器：[OpenCore Configurator Changelog version 2.18.0.2](https://mackie100projects.altervista.org/occ-changelog-version-2-18-0-2/)
  - 实用工具：[Hackintool Releases List](https://github.com/headkaze/Hackintool)
  - 调解显示器HIDPI：[resxtreme](https://macdownload.informer.com/resxtreme/)
- 自用OpenCore配置
  > **注意：请自行通过OpenCore Configurator生成序列号**
  - 链接: https://pan.baidu.com/s/1Qrq_GnG3mN2QRecT0ZZTEg  密码: 7u3j


## BIOS设置
> BIOS版本升级到了F14，需要解锁CFG LOCK，详见[🔗](https://khronokernel-2.gitbook.io/opencore-vanilla-desktop-guide/intel-config.plist/coffee-lake)
1. Save & Exit → Load Optimized Defaults
2. M.I.T. → Advanced Memory Settings Extreme Memory Profile(X.M.P.) : Profile 1
3. BIOS → Fast Boot : Disabled
4. BIOS → CMS Support: Disabled
5. BIOS → LAN PXE Boot Option ROM : Disabled
6. Peripherals → Super IO Configuration → Serial Port : Disabled
7. Peripherals → USB Configuration → XHCI Hand-off : Enabled
8. Chipset → Vt-d : Disabled

## 截图
![](/assets/2020-12-12/hackintosh-bigsur.png)

## 遇到的问题
1. [x] **访达硬盘图标显示异常**：发现是访达拓展兼容问题，偏好设置-> 拓展 -> "访达"拓展 取消勾选可疑的拓展，重启访达就好
2. [ ] **唤醒后隔空投送自动关闭**：平时不常用隔空投送，暂时先这样，不影响正常使用

## 参考链接
- [Gigabyte Z370 HD3P - i7 8700K - RX580 8GB - Catalina 10.15.5 - OpenCore 0.5.9](https://www.tonymacx86.com/threads/gigabyte-z370-hd3p-i7-8700k-rx580-8gb-catalina-10-15-5-opencore-0-5-9.256221/)
- [技嘉Z370-HD3P和i7-8700的OC0.5.6配置，媲美白苹果](http://bbs.pcbeta.com/forum.php?mod=viewthread&tid=1847176)