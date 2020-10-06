---
title: "Linux核心模組"
date: 2020-10-05T13:41:02Z
weight: 1
---

Linux提供動態載入移除核心模組的功能，使得記憶體不會被沒有使用到的模組佔據。
大部分的驅動程式都是透過Linux核心模組在開機時或開機後動態載入核心空間中。

這個章節會教導如何寫和編譯最基礎的核心模組，使同學了解在核心空間的程式是如何運行的。

## 編譯核心模組

### 下載Linux核心標頭檔

這次實驗會直接在樹莓派上面進行編譯核心模組的工作，因此要先下載對應的標頭檔。
不同的Linux核心版本使用的標頭檔可能會不一樣，而Raspberry Pi OS (32-bit)使用的版本可以透過以下指令下載。

```bash
sudo apt install raspberrypi-kernel-headers
```

### 核心模組基礎程式碼

Linux的核心模組需要透過``module_init``和``module_exit``註冊載入時的初始函式和移除時的卸載函式。

而核心模組一般而言要透過``MODULE_LICENSE``指定使用的[軟體授權條款](https://en.wikipedia.org/wiki/Software_license)，
而Linux核心本身是[GPL](https://en.wikipedia.org/wiki/GNU_General_Public_License)條款。
在載入核心模組時會檢查該模組的條款，如果使用的條款不是GPL相容的條款，將會無法呼叫部分Linux核心提供的函式。

*以上用到的Macro可以藉由``<linux/module.h>``引入。*

```C
#include <linux/module.h>

MODULE_LICENSE("GPL");

int mymodule_init(void) {
	printk("My Module INIT\n");
	return 0;
}

void mymodule_exit(void) {
	printk("My Module EXIT\n");
}

module_init(mymodule_init);
module_exit(mymodule_exit);
```
### 核心模組Makefile

Linux核心模組在編譯過程涉及許多步驟，編譯時也要加上許多編譯選項。
可以透過``make -C``指定Linux核心所在目錄，並使用其提供的Makefile。

在核心模組的Makefile需要透過``obj-m``標明要編譯的模組，該模組所需要用到的原始碼可以用``*-objs`` 來指定。
接下來透過``M=...``指定模組原始碼的路徑後，將編譯目標設為``modules``即可編譯出``*.ko``模組。

*``clean``可以清除編譯產生的檔案*

```none
obj-m += mymodule.o
mymodule-objs := main.o

modules:
	make -C /lib/modules/${shell uname -r}/build M=${PWD} modules

clean:
	make -C /lib/modules/${shell uname -r}/build M=${PWD} clean
```

### 編譯上述範例

```bash
mkdir lab6
cd lab6
# create main.c and fill in the above content
# create Makefile and file the above cotent
make
```

如果正確執行lab6目錄下會有類似以下內容

```none
$ make
make -C /lib/modules/5.4.51-v7+/build M=/home/pi/6 modules
make[1]: Entering directory '/usr/src/linux-headers-5.4.51-v7+'
  CC [M]  /home/pi/6/main.o
  LD [M]  /home/pi/6/mymodule.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC [M]  /home/pi/6/mymodule.mod.o
  LD [M]  /home/pi/6/mymodule.ko
make[1]: Leaving directory '/usr/src/linux-headers-5.4.51-v7+'
$ ls
main.c  main.o  Makefile  modules.order  Module.symvers  mymodule.ko  mymodule.mod  mymodule.mod.c  mymodule.mod.o  mymodule.o
```

## 載入和移除模組

在編譯完成後可以透過``insmod``和``rmmod``指令來載入和移除模組，這兩個指令最後會透過系統呼叫``finit_module``和``delete_module``來執行模組的載入和移除。

```bash
sudo insmod mymodule.ko
sudo rmmod mymodule
```

``printk``的內容不會直接印到當前的終端機，這些內容可以透過``dmesg``指令來看到

```none
$ dmesg | tail -2
[ 4998.931005] My Module INIT
[ 5003.245629] My Module EXIT
```


## kthread

``module_init``所註冊的函式要在做完基本的初始化後就離開。
如果需要有一個行程持續的執行，最簡單的方法是透過在初始化階段啟動一個新的執行緒稱為kthread。

而由於在核心空間執行的執行緒並不像在使用者空間執行的行程可以簡單的藉由信號就殺掉，
並且Linux核心並不會主動幫核心模組回收用完的資源。
因此在模組被移除時要呼叫``kthread_stop``通知執行中的kthread，而執行中的kthread則要透過``kthread_should_stop``檢查是否自己應該停止。


```C
#include <linux/delay.h>
#include <linux/kthread.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

struct task_struct* kthread;

int foo(void* arg) {
  int state = 0;
  while (!kthread_should_stop()) {
    printk("state: %d\n", state++);
    msleep(200);
  }
  return 0;
}

int mymodule_init(void) {
  kthread = kthread_run(foo, NULL, "foo");
  return 0;
}

void mymodule_exit(void) { kthread_stop(kthread); }

module_init(mymodule_init);
module_exit(mymodule_exit);
```

在載入上方模組後，開啟另一終端機以``dmesg --follow``可以觀察kthread foo定期印出遞增的數字。
而在移除模組後便會停止。

{{% notice note  %}}
如果再驅動程式開發過程中因為Bug導致整個系統異常，可以透過斷電重新啟動樹莓派來嘗試解決。
{{% /notice %}}