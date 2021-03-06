---
title: "基礎準備"
date: 2020-10-05T23:34:34Z
weight: 1
---

## 有線網路的替代方案

在先前實驗都是由SSH透過有線網路連入樹莓派當中。
由於這次實驗有線網路需要連到其他路由器而非開發的個人電腦，
因此需要透過其他的方式在樹莓派上開發。
以下是幾個可能的作法。

* 將樹莓派接上鍵盤滑鼠和螢幕   
* 使用UART連結樹莓派和主機
  * 將TTL線的RX接到樹莓派的TX針腳，TTL線的TX接到樹莓派的RX針腳，將接地互接。
  * 在``/boot/config.txt``加上``enable_uart=1``
  * MobaXterm的建立連線方式選擇``serial``
* 讓樹莓派和開發主機連在同一個路由器
  * 樹莓派和開發主機會在同一個子網路下，路由器會負責兩者之間的封包傳送
* 將開發主機連接網際網路的網路卡和連結樹梅派的網路卡以橋接模式相連(如果開發主機是以無線網路連至網際網路，此方法可能不會成功)
* 將開發主機作為路由器，把連到網際網路的網路介面卡以分享網路的方式，分享給與樹莓派連接之有線網路介面卡
  1. ![](https://imgur.com/w0vlMIt.jpg)
  2. 在樹莓派上加入預設閘道``sudo route add default gw 192.168.1.1``



## 將樹莓派的無線網路設定移除

在這次期末專題中，樹梅派的無線網路會做為無線存取點。
因此如果之前有設定將無線網路卡作為連線的設定要先移除。

```none
# network={
#        ssid="EOSDI"
#        psk="EOSDIEOSDI"
# }
```

```bash
# edit /etc/wpa_supplicant/wpa-supplicant.conf and comment out as above
sudo ip link set wlan0 down
sudo ip link set wlan0 up
```

## iw 
* 管理無線網路裝置的命令列工具
* 透過[netlink](https://en.wikipedia.org/wiki/Netlink)得到在Linux核心內部與網路相關資料結構
* 透過``iw list``可以看到樹莓派支援AP(無線網路存取點)的功能
![](https://i.imgur.com/2iz0Gia.png)


## rfkill

* 啟動和關閉無線裝置的工具(藍芽、無線網路)
* 使用``rfkill list``可以看到目前無線裝置是否啟用
* 使用``rfkill unblock 0``可以將屏蔽解除

## 無線存取點的靜態IP

* 由於其他裝置需要透過透過樹莓派連上網際網路，這些裝置要知道無線存取點的IP
* 使用 ``sudo ip addr add 192.168.20.1/24 dev wlan0`` 可以手動賦予無線網路介面卡靜態IP
  * 這個IP位址和其子網域不能和其他網域重疊，否則在傳遞封包時可能會無法送到正確的網路介面卡

