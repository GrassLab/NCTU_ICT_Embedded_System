---
title: "透過MMIO控制週邊裝置"
date: 2020-10-05T03:56:32Z
weight: 1
---

CPU可以透過記憶體位址來讀取寫入記憶體。
而這些記憶體位址也可以來對應到不同的IO裝置，透過對於特定記憶體位址的讀取寫入，
使得我們有辦法可以控制多樣的週邊裝置，而這樣控制IO裝置的方法被稱為[記憶體對映輸入輸出](https://en.wikipedia.org/wiki/Memory-mapped_I/O)(MMIO)。


## 樹莓派的實體記憶體位址分佈

每個不同的嵌入式裝置會因為使用的晶片不同而有不同的IO週邊，也會有不同的記憶體位址分布。
我們可以透過[procfs](https://www.kernel.org/doc/html/latest/filesystems/proc.html)底下的``iomem``來得知當前系統的實體記憶體分佈

```bash 
sudo cat /proc/iomem
```

```none
00000000-3b3fffff : System RAM
  00008000-00bfffff : Kernel code
  00d00000-00e7aaf7 : Kernel data
3f006000-3f006fff : dwc_otg
3f007000-3f007eff : 3f007000.dma
3f00a000-3f00a023 : 3f100000.watchdog
3f00b840-3f00b87b : 3f00b840.mailbox
3f00b880-3f00b8bf : 3f00b880.mailbox
3f100000-3f100113 : 3f100000.watchdog
3f101000-3f102fff : 3f101000.cprman
3f104000-3f10400f : 3f104000.rng
3f200000-3f2000b3 : 3f200000.gpio
3f201000-3f2011ff : serial@7e201000
  3f201000-3f2011ff : 3f201000.serial
3f202000-3f2020ff : 3f202000.mmc
3f212000-3f212007 : 3f212000.thermal
3f215000-3f215007 : 3f215000.aux
3f215040-3f21507f : 3f215040.serial
3f300000-3f3000ff : 3f300000.mmcnr
3f980000-3f98ffff : dwc_otg
```

知道了記憶體的整體分佈仍然不足以我們操作週邊裝置。
以GPIO為例，他的記憶體分佈範圍是3f200000-3f2000b3，但是我們仍然不知道如何改變輸出輸入模式或讀取輸入電壓，
這時候便需要讀該晶片的[規格書](https://cs140e.sergio.bz/docs/BCM2837-ARM-Peripherals.pdf)來得知。
如果規格書是非公開或者內容不詳盡，則只能參考Linux Kernel的原始碼來得知如何操控裝置。

## 透過虛擬記憶體映射控制週邊裝置

為了使得各個[行程](https://en.wikipedia.org/wiki/Process_(computing))可以有自己的記憶體空間，使得其執行起來像是獨占整個電腦，作業系統核心透過[記憶體管理單元](https://en.wikipedia.org/wiki/Memory_management_unit)使得每個行程活在自己的虛擬記憶體空間之中。
記憶體管理單元會將虛擬記憶體位址轉換成對應實體的記憶體位址並執行讀取和寫入。

在Linux中，在行程啟動後會根據執行檔提供資訊做最基本的虛擬記憶體映射(如stack, text區段)。如果要增加新的虛擬記憶體映射則需透過``brk``或``mmap``系統呼叫來達成。
在``mmap``系統呼叫可以指定使該映射對應到一個檔案，這稱作[記憶體對映檔案](https://en.wikipedia.org/wiki/Memory-mapped_file)。

在Linux中裝置可以作為一個檔案使得程式可以用讀寫一般檔案的方式讀寫裝置，這稱作[裝置檔案](https://en.wikipedia.org/wiki/Device_file)

我們將可以結合裝置檔案和記憶體對應檔案來建立**行程的虛擬記憶體位址**映射到**週邊裝置的實體記憶體位址**，以達成透過存取虛擬記憶體位址來控制週邊裝置。

### 裝置檔案結合記憶體對映檔案

在Linux中，[虛擬檔案系統](https://en.wikipedia.org/wiki/Virtual_file_system)允許裝置檔案有自己的開啟讀寫等方法，
裝置透過註冊自己的``struct file_operations``可以使得該裝置檔案的存取方式不同於一般檔案。

在``struct file_operations``可以修改的方法之一是``int (*mmap) (struct file *, struct vm_area_struct *)``。
驅動程式可以透過修改裝置檔案中的``mmap``方法，使得``mmap``所建立的虛擬記憶體映射到該裝置所在的實體記憶體區段。

在使用者執行``mmap``系統呼叫時，倘若指定參數為對應的裝置檔案，虛擬檔案系統最終會呼叫該裝置檔案的``mmap``方法，並完成虛擬記憶體的映射。


*在樹莓派官方維護的Linux分支中有加入[GPIO的記憶體映射](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/char/broadcom/bcm2835-gpiomem.c#L104-L121)
的驅動程式，可以透過看程式碼來了解GPIO的記憶體映射是如何建立的。*

## C範例

在了解了以上原理後，便可以用透過閱讀規格書中GPIO章節的敘述來控制GPIO裝置。

```C
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

// GPIO MMIO
#define SEL (0x00 / 4)
#define SET (0x1c / 4)
#define CLR (0x28 / 4)
#define LEV (0x34 / 4)

#define INPUT 0
#define OUTPUT 1

volatile unsigned int *gpio;

void setup(int pin, int mode) {
  int set = pin / 10;
  int num = pin % 10;
  gpio[SEL + set] &= ~(7 << (3 * num));
  switch (mode) {
    case INPUT:
      break;
    case OUTPUT:
      gpio[SEL + set] |= (1 << (3 * num));
      break;
    default:
      printf("Not Supported Mode %d\n", mode);
  }
}

int input(int pin) {
  int set = pin / 32;
  int num = pin % 32;
  return (gpio[LEV + set] & (1 << num)) != 0;
}

void output(int pin, int value) {
  int set = pin / 32;
  int num = pin % 32;
  if (value == 0) {
    gpio[CLR + set] |= (1 << num);
  } else {
    gpio[SET + set] |= (1 << num);
  }
}

int main(int argc, char **argv) {
  int fd = open("/dev/gpiomem", O_RDWR | O_SYNC);
  if (fd == -1) {
    printf("open /dev/gpiomem failed\n");
    exit(-1);
  }
  gpio = mmap(NULL, 1234, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
  if (gpio == MAP_FAILED) {
    printf("mmap /dev/gpiomem failed\n");
    exit(-1);
  }
  printf("virtual address: %p\n", gpio);
  int in = 3;
  int out = 4;
  setup(in, INPUT);
  setup(out, OUTPUT);
  while (1) {
    int tmp = input(in);
    if (tmp) {
      output(out, 1);
    } else {
      output(out, 0);
    }
    usleep(100000);
  }
}
```
可以注意到上述程式碼中``gpio``變數帶有額外關鍵字``volatile``，加入的原因是當按下按鈕時，電位改變會導致讀取該記憶體位址時得到不同的數值。
但是編譯器不知道這件事情，加入該關鍵字可以讓編譯器不要對該變數做優化而是每次都實際執行讀取記憶體的動作。

## 討論

* Linux本身已經有了``/dev/mem``裝置檔案來進行實體記憶體的存取，為什麼樹莓派官方還要額外加入``/dev/gpiomem`` ?
  * [原始碼](https://github.com/raspberrypi/linux/blob/rpi-5.4.y/drivers/char/mem.c#L373-L414)

