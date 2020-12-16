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

**完成TODO的部分(bonus不必要)，可以再之後進行補Demo並獲得3分**
然而由於TODO是用填空的方式，如果還是有不知道要填什麼的部分，請於之後課後詢問。

```c
#include <linux/delay.h>
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/kthread.h>
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

struct task_struct *count_thread;
struct task_struct *count_bonus_thread;
int irq;

// shared variables between each methods
struct gpio_desc *led;
struct gpio_desc *btn;
int led_state = 0;
unsigned int time = 1;

// methods
enum {
  PRESS_TIME,
  PRESS_COUNT,
  PRESS_COUNT_BONUS,
};
size_t current_method = PRESS_TIME;
const char *methods[] = {
    [PRESS_TIME] = "press_time",
    [PRESS_COUNT] = "press_count",
    [PRESS_COUNT_BONUS] = "press_count_bonus",
};

// kthread: press_count_bonus
int press_count_bonusd(void *arg) {
  // NOTE: You don't have to do the bonus.
  // If you'd like to try, finish the TODO part.
  const int RESET_TIME = 400;
  static int last_state = 0;
  static int last_click_time = 0;
  static int count = 0;
  int now;
  int current_state;
  while (!kthread_should_stop()) {
    msleep(20);
    if (current_method != PRESS_COUNT_BONUS) {
      continue;
    }
    current_state = gpiod_get_value(btn);
    now = jiffies_to_msecs(jiffies);
    if (last_state == 1 && current_state == 0) { // button released
      // TODO You should do something after click.
    }
    if (now - last_click_time > RESET_TIME) {
      // TODO if the time has passed the RESET_TIME,
      // you should reset something and switch the LED if needed.
    }
    last_state = current_state;
  }
  return 0;
}

// kthread: press_count
int press_countd(void *arg) {
  static int count = 0;
  static int last_state = 0;
  int current_state;
  while (!kthread_should_stop()) {
    msleep(20);
    if (current_method != PRESS_COUNT) {
      continue;
    }
    current_state = gpiod_get_value(btn);
    if (/* TODO finish this condition, this should be true if button is just released ) {
           HINT: you can use the value of last_state and current_state. */) {
      count = (count + 1) % time;
      if (count == 0) {
        // TODO you should switch the LED here.
      }
    }
    last_state = current_state;
  }
  return 0;
}

// irq: press_time
irqreturn_t btn_irq_handler(int irq, void *dev) {
  static int last_rise = 0;
  static int last_state = 0;
  int current_state;
  int now;
  if (current_method != PRESS_TIME) {
    return IRQ_HANDLED;
  }
  now = jiffies_to_msecs(jiffies);
  current_state = gpiod_get_value(btn);
  if (/* TODO finish this condition, this should be true if button is just released ) { 
         HINT: you can use the value of last_state and current_state.*/) {
    if (/* TODO check if push time is long enough
           HINT: you can use last_rise, now, and time.*/) {
      led_state = !led_state;
      gpiod_set_value(led, led_state);
    }
  } else {
    if (last_state == 0) {
      last_rise = now;
    }
  }
  last_state = current_state;
  return IRQ_HANDLED;
}

// sysfs attribute
ssize_t time_show(struct device *dev, struct device_attribute *attr,
                  char *buf) {
  // TODO write time as string into buf by sprintf as in the documentation.
}

ssize_t time_store(struct device *dev, struct device_attribute *attr,
                   const char *buf, size_t count) {
  int ret;
  unsigned long tmp;
  ret = kstrtoul(buf, 10, &tmp);
  if (ret) {
    return ret;
  }
  time = tmp;
  return count;
}

ssize_t method_show(struct device *dev, struct device_attribute *attr,
                    char *buf) {
  return sprintf(buf, "%s\n", methods[current_method]);
}

ssize_t method_store(struct device *dev, struct device_attribute *attr,
                     const char *buf, size_t count) {
  int i;
  char str[20];
  // consume '\n'
  sscanf(buf, "%19s", str);
  for (i = 0; i < ARRAY_SIZE(methods); ++i) {
    if (/* TODO compare str and methods[i] */) {
      current_method = i;
      return count;
    }
  }
  return -EINVAL;
}

// TODO use DEVICE_ATTR_RW(...) to declare method and time device attribute

/* initialization and exit code */
int mymodule_probe(struct platform_device *pdev) {
  int ret;
  struct device *dev = &pdev->dev;
  led = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW);
  btn = gpiod_get_index(dev, "btn", 0, GPIOD_IN);
  gpiod_set_value(led, led_state);
  count_thread = kthread_run(press_countd, NULL, "press_countd");
  count_bonus_thread =
      kthread_run(press_count_bonusd, NULL, "press_count_bonusd");
  irq = gpiod_to_irq(btn);
  ret = request_irq(irq, btn_irq_handler,
                    IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                    "btn_irq_handler", NULL);

  device_create_file(dev, &dev_attr_method);
  device_create_file(dev, &dev_attr_time);
  printk("mymodule PROBE\n");
  return 0;
}

int mymodule_remove(struct platform_device *pdev) {
  struct device *dev = &pdev->dev;
  device_remove_file(dev, &dev_attr_time);
  device_remove_file(dev, &dev_attr_method);
  free_irq(irq, NULL);
  kthread_stop(count_thread);
  kthread_stop(count_bonus_thread);
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
