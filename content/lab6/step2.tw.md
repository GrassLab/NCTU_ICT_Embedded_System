---
title: "平台裝置/驅動程式"
date: 2020-10-05T14:58:35Z
weight: 2
---

所有的周邊裝置都是藉由[匯流排](https://en.wikipedia.org/wiki/Bus_(computing))來連接，
某些裝置在在載入時可以主動通知Linux核心，此時Linux核心可以載入對應的驅動程式。
然而在嵌入式系統中，大部分的裝置都無法透過像這樣的方式讓Linux核心枚舉出現在載入的裝置。

這種無法主動被Linux核心發現的裝置稱為[平台裝置](https://www.kernel.org/doc/html/latest/driver-api/driver-model/platform.html)，
平台裝置通常是透過實驗五提過的裝置樹在開機時載入，而Linux核心提供了平台裝置驅動程式的框架來統一並加速相關驅動程式的開發。

## 平台驅動程式範例

```
#include <linux/module.h>
#include <linux/of.h>
#include <linux/platform_device.h>

MODULE_LICENSE("GPL");

struct of_device_id mymodule_dt[] = {
    {
        .compatible = "mymodule",
    },
    {},
};

int mymodule_probe(struct platform_device *pdev) {
  printk("mymodule PROBE\n");
  return 0;
}

int mymodule_remove(struct platform_device *pdev) {
  printk("mymodule REMOVE\n");
  return 0;
}

struct platform_driver mymodule = {
    .probe = mymodule_probe,
    .remove = mymodule_remove,
    .driver =
        {
            .name = "mymodule",
            .of_match_table = of_match_ptr(mymodule_dt),
            .owner = THIS_MODULE,
        },
};
module_platform_driver(mymodule);
```

和前面介紹過的核心模組不同，這這份程式碼中並沒有看到``module_init``和``module_exit``，取而代之的是``probe``和``remove``函式。
但是如果往下追蹤``module_platform_driver``這個註冊平台驅動程式的Macro會發現``module_init``和``module_exit``是被包裝在裡面的。

**參考原始碼**
* [platform_device.h](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/include/linux/platform_device.h#L232-L239)
* [device.h](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/include/linux/device.h#L1884-L1908)

``probe``函式會在裝置和驅動程式[綁定](https://www.kernel.org/doc/html/latest/driver-api/driver-model/binding.html)時呼叫。
在載入和移除模組時由於還沒有將對應的裝置載入的緣故，``probe``和``remove``函式都不會被執行到。

另外，可以注意到平台的驅動程式會透過指定``.compatible``來比對驅動程式和某個被枚舉出來的裝置是否相容。如果相容，才會進行裝置和驅動程式綁定並執行``probe``函式。


## 新增裝置overlay

透過裝置overlay使得上述範例的驅動程式可以綁定到裝置上。
在下面範例overlay中，新增了一個新的裝置叫做``mydevice``，他使用了針腳3、4作為LED和按鈕。

而由於該裝置的``target-path``是在根節點，在樹莓派上可以透過``dtoverlay``動態載入和移除``mydevice``裝置。

```none
/dts-v1/ ;
/plugin/ ;

/ {
  compatible = "brcm,bcm2835";
  fragment@0 {
    target-path = "/";
    __overlay__ {
      mydevice {
        compatible = "mymodule";
        led-gpios = <&gpio 3 0>;
        btn-gpios = <&gpio 4 0>;
      };
    };
  };
};
```

```bash
# crate mydevice.dts and fill in the above content
mkdir overlays
dtc -I dts -O dtb -o overlays/mydevice.dtbo mydevice.dts
sudo dtoverlay -d overlays mydevice
sudo dtoverlay -r mydevice
```

* 透過``dtoverlay -d <overlay所在目錄> <overlay名稱>``即可載入overlay
* 透過``dtoverlay -r  <overlay名稱>``即可移除overlay

在已經載入``mymodule``模組的情況下，可以發現當``mydevice``也被載入時，``probe``函式會被執行，在``mydevice``被移除時，``remove``函式會被執行。

```C
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/kthread.h>
#include <linux/delay.h>
#include <linux/gpio/consumer.h>

MODULE_LICENSE("GPL");

struct of_device_id mymodule_dt[] = {
    { .compatible = "mymodule", },
    {  }
};


struct task_struct* kthread;
struct gpio_desc* led;
struct gpio_desc* btn;

int blink(void* arg) {
    int state;
	while(!kthread_should_stop()) {
        state = gpiod_get_value(btn);
		gpiod_set_value(led, state);
		msleep(200);
	}
	return 0;
}
static int mymodule_probe (struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    led = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW);
    btn = gpiod_get_index(dev, "btn", 0, GPIOD_IN);
    kthread =  kthread_run(blink, NULL, "LED_BLINK");
    printk("mymodule PROBE\n");
    return 0;
}

static int mymodule_remove(struct platform_device *pdev)
{
    kthread_stop(kthread);
    gpiod_put(btn);
    gpiod_put(led);
    printk("mymodule REMOVE\n");
    return 0;
}

static struct platform_driver mymodule = {
    .probe      = mymodule_probe,
    .remove     = mymodule_remove,
    .driver     = {
        .name     = "mymodule",
        .of_match_table = of_match_ptr(mymodule_dt),
        .owner    = THIS_MODULE,
    },
};
module_platform_driver(mymodule);
```

{{% notice warning %}}
如果該針腳已經被其他驅動程式所使用的，``gpiod_get_index``將會回傳錯誤。
在文件中為求簡潔沒有做額外的檢查，在實際驅動開發中檢查回傳值是不可省略的。
{{% /notice %}}