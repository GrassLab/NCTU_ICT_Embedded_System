---
title: "無線路由器基本功能需求"
date: 2020-10-06T00:42:31Z
weight: 3
---

## 需要實作的無線路由器基本功能

基本功能的無線路由器需要用到1個按鈕和2個LED燈。
* 按鈕作為開關樹莓派的無線存取點功能。
* 其中一個LED燈指示當前無線存取點功能是否開啟。
* 另外一個LED燈在接收和傳送封包時會閃爍。
* 重新啟動樹莓派後，所有服務需要自動重啟。

## Python範例程式

這個期末專題不限定用C，Python，或者Shell Script。
但是如果同學想要使用Python，下方的範例程式可能會有幫助

```python
import os
import psutil

# 可以從sysfs讀取wlan0累積接收的封包大小
with open('/sys/class/net/wlan0/statistics/rx_bytes', 'r') as rx:
    rbytes = int(rx.readline())
# 可以藉由os.system()來執行命令列程式
os.system('service hostapd start')
os.system('service hostapd stop')
# 可以藉由psutil來確認hostapd是否執行中
hostapd_running = 'hostapd' in (p.name() for p in psutil.process_iter())
```

## 開機後自動啟動服務和設定

對於一般使用者而言，在嵌入式裝置在每次重新開機後手動啟動對應服務和設定是不可能的。
因此需要啟動的背景程式和設定檔必須要紀錄在檔案系統中。
在重新開機之後，初始化的程式透過預設的腳本來將服務一一啟動。

在不同的Linux發行板可能有不一樣的自動啟動服務方式，以下是在樹莓派上可行的自動啟動服務方式。


**Linux核心模組**

* 若是外部核心模組需要先進行以下步驟
  1. 將``<模組>.ko``複製到``/lib/modules/$(uname -r)/``目錄底下
  2. ``depmod``指令建立模組的關係圖
  3. 可以用``modprobe <模組>``來確定是否成功
* 在``/etc/modules``加入模組名稱

**overlays**
* 將指定的``<overlay名稱>.dtbo``放到``/boot/overlays``
* 在``/boot/config.txt``加入``dtoverlay=<overlay名稱>``

**靜態IP**

* 在``/etc/network/interfaces``加入以下內容

```none
auto wlan0
  
iface wlan0 inet static 
    address 192.168.20.1
    network 192.168.20.0
    netmask 255.255.255.0
```

**hostapd**

* 將``hostapd.conf``複製到``/etc/hostapd/hostapd.conf``
* ``sudo systemctl unmask hostapd``將對應服務屏蔽解除


**ip_forward**
* 在``/etc/sysctl.conf``加入``net.ipv4.ip_forward=1``

**iptables**
* ``sudo apt install iptables-persistent`` 下載套件
* ``sudo iptables-save > /etc/iptables/rules.v4`` 將現有的IP轉換規則儲存

**dhcpd**          

* 將``dhcpd.conf``複製到``/etc/dhcp/dhcpd.conf``

**一般腳本**

* 使用``crontab -e``可以將要執行的內容放到[Cron](https://en.wikipedia.org/wiki/Cron)中自動執行
```none
@reboot  sudo /home/pi/your_script.sh
```