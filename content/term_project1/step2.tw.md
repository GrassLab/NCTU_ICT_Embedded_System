---
title: "無線網路和路由相關工具"
date: 2020-10-06T00:11:06Z
weight: 2
---

## hostapd

* ``sudo apt install hostapd``
* 處理無線存取點的加密和認證身份的背景程式
* 填寫``ssid``以改變名稱，填寫``wpa_passphrase``以改變密碼

```none
# wireless interface name
interface=wlan0

ssid=<AP's name you want>

# IEEE 802.11g (2.4 GHz) with 7 channels
hw_mode=g
channel=7

# cipher protocol
wpa=2
auth_algs=1
wpa_pairwise=CCMP
wpa_key_mgmt=WPA-PSK
wpa_passphrase=<password you want>
```

```bash
# create hostapd.conf and fill in the above content
sudo hostapd hostapd.conf
```
在執行完上面步驟後，打開手機便可找到與你設定的SSID的存取點。

![](https://i.imgur.com/vYnDTum.png)
![](https://i.imgur.com/FnjxHCM.png)

在輸入正確密碼後會發現和上圖一樣卡在取得IP位址的地方

## 行動裝置選擇靜態IP

* 上面無法取得IP的原因是因為在手機上預設是用DHCP的方式取得IP
* 可以選擇使用靜態IP，挑選一個在無線存取點當中尚未被使用的子網域IP(例如 192.168.20.2/24)
![](https://i.imgur.com/pN72HkG.png)

獲得了IP後會發現裝置仍然沒有連上網際網路，這是因為樹莓派預設沒有執行[封包轉送](https://en.wikipedia.org/wiki/Packet_forwarding)。

## 封包轉送

### ip_forward

* 裝置的封包藉由會送入樹莓派的無線網路介面卡
* 為了連上網際網路，該封包要能夠透過樹莓派的有線網路介面卡出去
* 啟動``ip_forward``來達到這像功能
* 可以藉由``sysctl net.ipv4.ip_forward``確認當前``ip_forward``是否啟動
* ``sudo sysctl net.ipv4.ip_forward=1``可以啟動``ip_forward``
* ``sysctl``藉由讀寫``/proc/sys/net/ipv4/ip_forward``內容來改變Linux核心設定


### 網路位址轉換

封包路由轉送的規則是透過在封包中的IP來決定，
目前我們使用的無線存取點的子網域是自己定義的[私有IP](https://en.wikipedia.org/wiki/Private_network)，
這個IP是連上網際網路的路由器所不認得的。
因此當外界要回傳封包時，根據IP並無法將封包回傳。

因此除了設定``ip_forward``之外，還要設定[網路位址轉換](https://en.wikipedia.org/wiki/Network_address_translation)
網路位址轉換可以透過``iptables``工具來進行。

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**在完成以上步驟之後可以藉由在Google搜尋``my ip``，確認IP是和樹莓派的相符則設定成功。**

## DHCPD

* 可以透過``dhcpd``支援[動態分配IP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)的服務
* ``sudo apt install isc-dhcp-server``

```none
option domain-name-servers 8.8.8.8, 8.8.4.4;
option subnet-mask 255.255.255.0;
option routers 192.168.20.1;
subnet 192.168.20.0 netmask 255.255.255.0 {
  range 192.168.20.10 192.168.20.20;
}
```

```bash
# create dhcpd.conf and fill in the above content
sudo dhcpd -cf dhcpd.conf
```