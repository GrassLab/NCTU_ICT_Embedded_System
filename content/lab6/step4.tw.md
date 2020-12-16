---
title: "Demo"
date: 2020-10-06T07:31:19Z
weight: 4
---

## Demo

實做平台裝置驅動程式，使其可以透過按鈕來開關LED燈

該驅動程式必須完成以下功能

* 在sysfs下新增裝置屬性``method``和``time``，使用者可以透過寫入裝置屬性來控制按鈕開關LED的方式，並可以透過讀取裝置屬性來得知當前開關之設定
  * ``method``的可能值有``press_time``和``press_count``兩種，預設為``press_time``。 
  * ``time``的可能值為大於等於1的整數。
  * 如果使用者輸入非法數值，則維持先前數值
* 在``method``為``press_time``的情況下，請實做以下功能
  * 以中斷的方式得知按鈕持續按壓的時間
  * 若按壓時間超過``time``設置的時間，則將LED切換為打開/關閉
  * ``time``的單位可以為秒或者毫秒 
* 在``method``為``press_count``的情況下，請實做以下功能
  * 以kthread的方式監控按鈕連擊的次數
  * 若連擊次數剛好等於``time``，則將LED切換為打開/關閉


同學可以使用``jiffies_to_msecs(jiffies)``來計算經過的時間

```c
#include <linux/jiffies.h>
#include <linux/delay.h>

void example(void) {
  int start_ms, end_ms;
  start_ms = jiffies_to_msecs(jiffies);
  msleep(200);
  end_ms = jiffies_to_msecs(jiffies);
  printk("%dms\n", end_ms - start_ms); // should be around 200ms
}
```
