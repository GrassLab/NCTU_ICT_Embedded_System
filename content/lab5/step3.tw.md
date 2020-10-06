---
title: "加入新的LED裝置"
date: 2020-10-05T06:19:00Z
weight: 3
---

在前面章節已經學習到了如何透過sysfs直接控制GPIO裝置進而達到控制LED的功能。
Linux核心也有針對GPIO控制的LED燈的驅動程式。
樹梅派的LED驅動程式會透過裝置樹來得知當前有的LED裝置。同學將學習裝置樹是什麼，並嘗試加入新的GPIO LED裝置。

## Sysfs與LED

LED裝置可以藉由``/sys/class/leds``目錄存取。在該目錄下面使用``ls``會看到已經有其他LED裝置存在。

每個LED裝置有自己的預設觸發方式。透過``cat /sys/class/leds/led0/trigger``可以看到所有的觸發方式，而當前選擇的觸發方式會被``[]``所框住。

而我們也可以透過寫入字串來改變觸發方式。
```bash
sudo sh -c "echo heartbeat > /sys/class/leds/led0/trigger``
```
執行上面指令後便可以看到樹梅派的某個LED燈像心跳一樣定期的開關。

*和GPIO相同，LED雖然是簡單的裝置，由於要相容不一樣的LED裝置(非GPIO,可調整亮度,可調整顏色等)，Linux核心也是以結構化的方式撰寫驅動程式。
相關的原始碼在[leds](https://github.com/raspberrypi/linux/tree/rpi-5.4.y/drivers/leds)目錄底下。*

## 裝置樹

### 簡介

在嵌入式裝置中[裝置樹](https://elinux.org/Device_Tree_Reference)是用來描述該系統中包含CPU、記憶體、中斷晶片和其他週邊裝置的檔案。
除此之外，由於GPIO的針腳可能作為不定用途使用(如LED, UART, SPI...)，裝置樹可以使用overlays將額外的描述疊加上去而達到客製化針腳用途。

* 裝置樹的原始碼的副檔名一般為``.dts``，並可以引入其他裝置樹檔案``.dtsi``，藉此重複使用相同程式碼。
* 原始碼需要先透過``cpp``的C前置處理器，將Macro解開。再來透過``dtc``指令編譯為Linux核心可以解析的二進制檔案，副檔名一般為``.dtb``。
* ``.dtb``最後會被放在Boot Partition也就是``/boot/``底下，開機後會由Bootloader載入。
* Bootloader在載入裝置檔案前會檢查``/boot/config.txt``的內容，如果有指定``dtoverlay=...``或者``dtparam=...``則會從``/boot/overlays``底下尋找對應的裝置樹檔案，
並且將參數覆蓋上去。

*這份文件不會額外介紹裝置樹的語法，麻煩自行參閱前述文件以及
[樹梅派官方裝置樹介紹](https://www.raspberrypi.org/documentation/configuration/device-tree.md)
來了解如何撰寫裝置樹。*

### LED overlays範例

以下範例在leds節點下新增了一個裝置``myled``，該LED裝置使用GPIO的3號腳位。

```none
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2835";

    fragment@0 {
        target = <&leds>;
        __overlay__ {
            myled {
                gpios = <&gpio 3 0>;
            };
        };
    };
};
```

將其儲存為``myled.dts``後，使用``dtc``便可轉換為``.dtb``格式將其放入``/boot/overlays``目錄下後，
最後在``/boot/config.txt``加入``dtoverlay=myled``並重新開機便可看到``myled``出現在``/proc/device-tree/leds``和``/sys/class/leds``目錄之下。
```bash
dtc -I dts -O dtb -o myled.dtbo myled.dts
sudo cp myled.dtbo /boot/overlays
sudo sh -c "echo dtoverlay=myled >> /boot/config.txt" 
sudo reboot
```

重新開機之後便可以透過改變``/sys/class/leds/myled/trigger``的內容來控制LED。

---

*注:部分overlays可以在不用透過重新開機的情況下透過``dtoverlay``指令新增或移除裝置，這會在下次實驗使用*

### 原始碼

如果想要了解裝置樹和overlays如何撰寫，可以參閱以下既有的樹梅派裝置樹原始碼。

* [bcm2837-rpi-3-b-plus.dts](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/arch/arm/boot/dts/bcm2837-rpi-3-b-plus.dts)
* [overlays](https://github.com/raspberrypi/linux/tree/rpi-5.4.y/arch/arm/boot/dts/overlays)

