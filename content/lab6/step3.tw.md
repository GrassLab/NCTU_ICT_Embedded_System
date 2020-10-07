---
title: "GPIO的中斷與sysfs"
date: 2020-10-06T07:31:19Z
weight: 3
---

在前面章節在前面章節簡單的介紹了平台裝置，以及如何透過Linux本身GPIO函式庫進行GPIO操作。
這個章節會介紹如果透過中斷來降低按鈕電位改變至LED的電位連帶改變的延遲。
並且介紹如何將``mydevice``註冊到``sysfs``

## 中斷

CPU可以接收來自周邊裝置的硬體[中斷](https://en.wikipedia.org/wiki/Interrupt)，並在接收到中斷時才存取周邊裝置的資料。
如此一來驅動程式不需要透過[輪詢](https://en.wikipedia.org/wiki/Polling_(computer_science))持續的存取周邊裝置。

透過Linux核心GPIO函式庫中的``gpiod_to_irq``可以直接拿到一個中斷號碼，中斷號碼可以再透過``request_irq``來註冊中斷處理函式。

在註冊中斷處理函式時可以指定中斷的觸發方式，藉此可以達到按下按鈕和放開時都觸發該函式達到開關LED的效果。

```C
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
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

int irq;
struct gpio_desc* led;
struct gpio_desc* btn;

irqreturn_t btn_irq_handler(int irq, void* dev) {
  int state = gpiod_get_value(btn);
  gpiod_set_value(led, state);
  return IRQ_HANDLED;
}

int mymodule_probe(struct platform_device* pdev) {
  struct device* dev = &pdev->dev;
  int retVal;
  led = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW);
  btn = gpiod_get_index(dev, "btn", 0, GPIOD_IN);
  irq = gpiod_to_irq(btn);
  retVal = request_irq(irq, btn_irq_handler,
                       IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                       "btn_irq_handler", NULL);
  printk("mymodule PROBE, requested irq %d\n", retVal);
  return 0;
}

int mymodule_remove(struct platform_device* pdev) {
  free_irq(irq, NULL);
  gpiod_put(btn);
  gpiod_put(led);
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

* 在使用中斷後可能會發現按鈕變得很敏感，這是因為按按鈕的時候訊號是會抖動的。
而由於這邊使用的中斷觸發是根據訊號邊緣變化，因此會導致重複觸發。
這時候需要透過一些硬體手法(如電容)或者軟體手法(等待一小段時間)來去除開關彈跳帶來的訊號不穩現象。

* 雖然以上程式碼看起來很簡單，然而作業系統核心在處理中斷時由於當下狀態不屬於任何行程，因此有部分操作是禁止的。
而Linux核心針對中斷處理也有引入許多額外的軟體機制來幫助驅動程式開發者可以針對不同情形選用不一樣方式處理中斷。
礙於篇幅緣故，在這份文件中不會多做介紹，同學可以從其他書籍和既有的驅動程式原始碼來學習如何寫出更好的中斷處理函式。

## Sysfs

我們在實驗五時已經知道，Linux核心提供了sysfs的框架讓裝置可以註冊自己的屬性。
同時也看了LED和GPIO的驅動程式是如何使用這些機制。

在下方程式碼中，``mymodule``會透過``device_create_file``將新載入的裝置引入兩個屬性``button_pushed``和``ledon``。

* 透過``DEVICE_ATTR``Macro可以產生``dev_attr_*``的資料結構並對應上``*_show``或``*_store``的處理函式。
* 將裝置和屬性作為參數傳入``device_create_file``後，sysfs便會在裝置和驅動程式綁定後產生對應的屬性檔案。
* ``*_show``相關的函式，可以透過``sprintf``將內容印到sysfs提供的空間，這些內容最終會被使用者以``read``系統呼叫取得。
* ``*_store``相關的函式，可以透過``kstrtoul``或者``sscanf``來將字串轉成驅動程式內部資料結構的格式，並且可以對該裝置做額外的操作。

```C
#include <linux/gpio/consumer.h>
#include <linux/module.h>
#include <linux/of.h>
#include <linux/platform_device.h>
#include <linux/sysfs.h>

MODULE_LICENSE("GPL");

struct of_device_id mymodule_dt[] = {
    {
        .compatible = "mymodule",
    },
    {},
};

struct gpio_desc *led;
struct gpio_desc *btn;
int is_blink = 0;

ssize_t button_pushed_show(struct device *dev, struct device_attribute *attr,
                           char *buf) {
  return sprintf(buf, "%d\n", gpiod_get_value(btn));
}

ssize_t ledon_show(struct device *dev, struct device_attribute *attr,
                   char *buf) {
  return sprintf(buf, "%d\n", is_blink);
}

ssize_t ledon_store(struct device *dev, struct device_attribute *attr,
                    const char *buf, size_t count) {
  int ret;
  unsigned long state;
  ret = kstrtoul(buf, 10, &state);
  if (ret) {
    return ret;
  }
  is_blink = (state == 0) ? 0 : 1;
  gpiod_set_value(led, is_blink);
  return count;
}

DEVICE_ATTR_RW(ledon);
DEVICE_ATTR_RO(button_pushed);

int mymodule_probe(struct platform_device *pdev) {
  struct device *dev = &pdev->dev;
  led = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW);
  btn = gpiod_get_index(dev, "btn", 0, GPIOD_IN);
  device_create_file(dev, &dev_attr_button_pushed);
  device_create_file(dev, &dev_attr_ledon);
  printk("mymodule PROBE\n");
  return 0;
}

int mymodule_remove(struct platform_device *pdev) {
  struct device *dev = &pdev->dev;
  device_remove_file(dev, &dev_attr_button_pushed);
  device_remove_file(dev, &dev_attr_ledon);
  gpiod_put(btn);
  gpiod_put(led);
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

在載入裝置``mydevice``後，即可看到``/sys/devices/platform/mydevice/``目錄。

在載入驅動程式``mymodule``前會發現目錄下還沒有``ledon``和``button_pushed``的屬性檔案。
載入``mymodule``後，裝置和驅動程式產生綁定，接下來``probe``函式便會進行初始化和產生屬性檔案。

接下來就可以和實驗五做過的一樣，透過簡單的腳本語言控制你實作的新GPIO裝置。