---
title: "設定SSH"
date: 2020-09-23T15:56:49Z
weight: 2
---

雖然樹莓派有USB和HDMI介面，這使得開發上是可以像在一般Linux系統上面開發一樣。
但是在一般的情況下，嵌入式的開發主要還是透過UART或者網路連結嵌入式開發版。
在這個部分會教導在使用有線網路連結樹莓派和開發主機後，如何連線進入樹莓派系統



## 設定靜態IP

在使用有線網路連結兩台電腦後，兩台電腦就形成了網路。
網路的溝通是透過IP位址來找到對方，因此需要設定開發主機和樹莓派的IP位址

### 樹莓派靜態IP設定

#### 在Windows上讀取Ext4檔案系統

由於樹莓派的部分設定檔是在Root Partition也就是Ext4檔案系統上，要在Windows系統調整設定檔需要透過額外工具。

Windows可以透過[Ext2Fsd]( http://www.ext2fsd.com/?page_id=16)來讀取Ext4檔案系統

右鍵點選目標分區，選擇**Assign Driver Letter** 即可從Windows系統中讀到該映像檔Root Partition內容
![](https://i.imgur.com/hPkBZvx.png)

#### dhcpcd.conf 設定

在可以讀到Root Partition後，可以看到在etc目錄底下有一個dhcpcd.conf的設定檔。
以記事本開啟後新增
```bash
interface eth0
static ip_address=192.168.1.2/24
```

完成後如下圖即可
![](https://i.imgur.com/8Tpl7ij.png)

### Windows靜態IP設定

1. 按下 Windows + R，輸入「ncpa.cpl」。
![](https://i.imgur.com/rA6s4zF.png)

2. 在新增的網卡上右鍵，然後選取【內容】> 【網際網路通訊協定第 4 版 (TCP/IPv4)】
3. 點擊【使用下列的 IP 位址】
   * IP 位址輸入 「 192.168.1.1 」
   * 子網路遮罩輸入「 255.255.255.0 」
   * 慣用 DNS 伺服器「 192.168.1.1 」

![](https://i.imgur.com/42xNkrB.png)

## 開啟樹莓派SSH服務

樹莓派預設開機後並沒有啟動SSH服務，
但是可以透過簡單在Boot Partition下(非Root Partition)建立ssh空檔案。
樹莓派開機後偵測到此檔案會自動開啟SSH服務。

新增方法:
* 在 Boot Partition 當中按下右鍵，然後選取【新增】> 【文字文件】， 將名稱改為「ssh」。

![](https://i.imgur.com/nx6HG5m.png)