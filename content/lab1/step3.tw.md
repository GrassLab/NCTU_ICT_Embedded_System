---
title: "連線進入樹莓派及樹莓派無線網路設定"
date: 2020-09-30T03:14:59Z
weight: 3
---

在上一部份中，透過開發主機完成了樹莓派的基礎設定。
將SD卡插入樹梅派後，連結電源和插入有線網路後，開發主機即可透過SSH工具連入樹莓派。

這個部分會介紹如何透過MobaXterm連入樹莓派並教導如何使樹梅派透過無線網路連上網際網路。

## MobaXterm

[MobaXterm](https://mobaxterm.mobatek.net/download-home-edition.html) 是在Windows上一個十分便利終端機程式。
他可以建立SSH, Serial等方式與遠端建立連線，並且支援遠端檔案編輯。


### 建立SSH連線

1. 點選【Session】> 【SSH】
2. 再Remote Host 輸入「192.168.1.2」
3. 點選【Specify username】
4. 再username欄位輸入``pi``
5. 點擊 OK
6. 輸入密碼``raspberry`` 即可看到登入進樹莓派中

![](https://i.imgur.com/Y7XgYCY.png)

## 無線網路設定

由於開發主機和樹莓派之間的網路連線只是區域網路，
在沒有設定開發主機的封包轉發和NAT的狀況下，樹莓派是無法連至外部網路的。

之後的實驗需要連線至網際網路的能力，這邊可以透過無線網路來達成。

### 步驟
1. 在終端機中輸入``sudo raspi-config``
2. 選擇【Network Options】>【Wireless LAN】
![](https://i.imgur.com/ZMHuWzG.png)
3. 輸入無線網路的 SSID
4. 輸入無線網路的密碼
5. 選擇【Finish】
6. 使用 ``ping`` 指令確認是否連上網際網路
   * ```ping -c4 google.com```
![](https://i.imgur.com/KKt9LfF.png)




