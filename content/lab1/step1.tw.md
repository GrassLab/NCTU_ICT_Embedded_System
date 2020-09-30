---
title: "安裝樹莓派作業系統"
date: 2020-09-23T15:53:09Z
weight: 1
---

## 作業系統映像檔安裝

請至[樹莓派官網映像檔下載頁面](https://www.raspberrypi.org/downloads/raspberry-pi-os/)下載 
**Raspberry Pi OS (32-bit) with desktop and recommended software**。

下載完畢後將壓縮檔解壓縮後，將SD卡置入讀卡機並插入個人電腦即可進行燒錄。
燒錄可以使用[Raspberry Pi Imager](https://www.raspberrypi.org/downloads/)

開啟Imager後先選好你要燒錄的SD卡後，在選擇燒錄作業系統時請使用**Use custom**選項，
![](https://i.imgur.com/U3AkiOf.png)


## 映像檔結構

Raspberry Pi OS 映像檔分為兩個部分，一個是Boot Partition，另一個是Linux的Root Partition

### Boot Partition

顧名思義為放置Linux開機所需相關檔案，包含Linux核心(kernel)，裝置描述檔案(Device Tree)，樹莓派自身所需韌體及設定檔。
該分區的檔案系統為FAT，因此Windows系統也可以直接讀取

如果同學對於樹莓派開機流程及內部檔案有興趣可以參考以下兩個樹莓派官網提供之資訊

[Boot Partition檔案](https://www.raspberrypi.org/documentation/configuration/boot_folder.md)

[開機流程](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/README.md)

### Root Partition

該分割區放置的剩餘Linux作業系統所需的所有檔案。
該分區的檔案系統為Ext4，是Linux的檔案系統。
在Windows上讀取會需要額外的工具，這部分會在下一部份說明。