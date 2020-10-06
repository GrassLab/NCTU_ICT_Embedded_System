---
title: "透過sysfs操作GPIO"
date: 2020-10-05T06:17:02Z
weight: 2
---

在實驗二時曾經透過載入溫度感測器的驅動程式，然後讀取``/sys/``目錄底下的檔案來得到感測器的數值。
這是透過Linux中特殊的檔案系統[sysfs](https://www.kernel.org/doc/html/latest/filesystems/sysfs.html)來完成。

同樣地，在Linux中的GPIO驅動程式也有結合sysfs，使得一般使用者不用寫程式，透過腳本語言就可以簡單控制GPIO。

## Sysfs 與 GPIO

在sysfs中，驅動程式可以跟sysfs註冊一系列的屬性，這些屬性會以檔案的形式出現在sysfs的目錄下面。
這些屬性通常和裝置和驅動程式的行為緊密結合，使得使用者可以透過改變檔案的內容進而改變驅動程式的參數，最後改變裝置的行為。
而部份簡單的輸入如GPIO也可以透過這些檔案來顯示當前參數或者輸入。

在Linux核心中，sysfs屬性的註冊已經有既定的框架使得驅動程式的撰寫者可以更容易開發。
所有關於寫入的屬性以``*_store``的方式命名，而讀取的屬性以``*_show``的方式命名。
透過``ATTR``系列的Macro，會將相對應的資料結構一併建立完成。

以[gpio-sysfs](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/gpio/gpiolib-sysfs.c#L101-L152)為例，讀寫當前GPIO針腳電位的屬性名為``value``，
可以看到驅動程式只須撰寫與GPIO相關的程式邏輯，而不必寫與sysfs相關的程式邏輯。

## 結構化的驅動程式設計

在前一個章節有介紹了週邊裝置的存取要透過MMIO，然而在上面的程式碼中並沒有見到任何與MMIO相關的程式碼。
這個原因是因為不同的嵌入式裝置都有自己的GPIO控制晶片，如果在sysfs相關的程式碼中加入邏輯判斷不同的裝置採用不一樣的記憶體映射會導致程式碼混雜。
在Linux核心中，驅動程式的設計都是有階層結構化的，簡單如GPIO也不例外。

以GPIO輸入的數值為例，從gpio-sysfs中可以看到數值是透過``gpiod_get_value_cansleep``得到，這個函式的是在另一份原始碼
[gpiolib](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/gpio/gpiolib.c#L4061-L4076)定義，
繼續往下追蹤可以看到[chip-get](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/gpio/gpiolib.c#L3274)的函式呼叫。

這個函式呼叫會找到對應的GPIO晶片並呼叫對應的方法來得到輸入的值，
樹莓派使用的針腳驅動程式
[pinctrl-bcm2835](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/pinctrl/bcm/pinctrl-bcm2835.c#L343-L357)會在載入時註冊一系列方法。
因此在一系列的函式呼叫後，最後會呼叫到[bcm2835_gpio_rd](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/pinctrl/bcm/pinctrl-bcm2835.c#L236-L239)並讀取GPIO的數值。

由上面例子可以學到如何以結構化的方式將抽象的行為和實際的硬體操作解藕，在進行實際的驅動程式開發時也能事半功倍。

## Shell Script範例

在了解了以上GPIO和sysfs的基本原理後，可以嘗試用腳本語言來達到控制GPIO的效果

```bash
#!/bin/bash

output=3
input=4

echo $output > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio${output}/direction
echo $input > /sys/class/gpio/export
echo in > /sys/class/gpio/gpio${input}/direction

trap "echo ${output} > /sys/class/gpio/unexport;\
	echo ${input} > /sys/class/gpio/unexport; exit" \
	SIGHUP SIGINT SIGTERM


while true
do
	a=$(cat /sys/class/gpio/gpio${input}/value)
	echo $a | tee /sys/class/gpio/gpio${output}/value
	sleep 0.3
done
```

在儲存上述檔案為``gpio.sh``後可以將其以``chmod +x``轉為可執行，便可以直接執行

```bash
chmod +x gpio.sh
./gpio.sh
```

### 上方腳本解釋

如果對於腳本語言不熟悉的同學可以參考下方解釋，並自行嘗試其他變化以熟悉。
對於沒見過得指令可以用``man``或者直接google來查詢。

---

```bash
#!/bin/bash
```

* 上面看起來像是註解的內容被稱作[Shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))，
  * 其原理為Linux核心執行一個執行檔時，會先檢查該檔案開頭的部份。如果發現了``#!``則會認為這是一個給直譯器讀的腳本，Linux核心會執行該直譯器，並將該腳本檔案路徑作為參數傳遞給該直譯器。
  * 該直譯器在被執行後可以根據參數打開腳本並且執行對應的內容。

---

```bash
output=3
input=4
```
* 在bash中也有變數，可以直接賦予變數數值。

---


```bash
echo $output > /sys/class/gpio/export
# echo 3 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio${output}/direction
# echo out > /sys/class/gpio/gpio3/direction
```

* ``echo``會將後方的字串印出到stdout。
* 為了區分變數和一般字串，使用``$``和``{}``來標示是變數，變數會被替代為之前賦予的值。
* ``>``可以重新導向stdout的值到後方指定的檔案。

---

```bash
trap "echo ${output} > /sys/class/gpio/unexport;\
	echo ${input} > /sys/class/gpio/unexport; exit" \
	SIGHUP SIGINT SIGTERM
```

* ``trap``可以指定在接收到對應[訊號](https://en.wikipedia.org/wiki/Signal_(IPC))時要執行的動作，
  * 由於後面會用無窮迴圈來讀取按鈕輸入，使用``trap``可以讓腳本結束時清理中間產生的額外檔案

---

```bash
while true
do
	a=$(cat /sys/class/gpio/gpio${input}/value)
	echo $a | tee /sys/class/gpio/gpio${output}/value
	sleep 0.3
done
```

* bash也有提供if, for, while的邏輯控制，因此也可以完成簡單的程式邏輯
* ``X=$(指令)``可以將指令執行的結果存到變數裡面