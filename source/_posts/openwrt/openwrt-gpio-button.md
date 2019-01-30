---
title: OpenWRT GPIO 按键绑定
date: 2019-01-30 11:34:03
categories:
	- openwrt
tags:
	- openwrt
	- gpio
---

# OpenWRT GPIO Button

## GPIO Button

OpenWRT gpio button有两种方式：
1. 普通gpio-keys/gpio-key-polled按键
	这种相当于普通键盘事件
2. 启用gpio-button-hotplug
	OpenWRT会将按键事件触发为hotplug事件，从而procd就可以接收事件从而运行/etc/rc.button下对应的脚本。 使用案件netlink转发事件到procd还是比较麻烦的，gpio-button-hotplug算是比较直接的方式。


## gpio-keys 和 gpio-key-polled

使用此方法映射按键，需要去***make menuconfig***中的Kernel Modules->Input Modules里面选中对应的选项。而此后在/dev/input下就会出现event<n\>文件


## gpio-button-hotplug 按键映射

OpenWRT的gpio-button-hotplug提供了以下按键的映射,即按键code对应的脚本文件名称。

```c

#define BH_MAP(_code, _name)		\
	{				\
		.code = (_code),	\
		.name = (_name),	\
	}

static struct bh_map button_map[] = {
	BH_MAP(BTN_0,			"BTN_0"),
	BH_MAP(BTN_1,			"BTN_1"),
	BH_MAP(BTN_2,			"BTN_2"),
	BH_MAP(BTN_3,			"BTN_3"),
	BH_MAP(BTN_4,			"BTN_4"),
	BH_MAP(BTN_5,			"BTN_5"),
	BH_MAP(BTN_6,			"BTN_6"),
	BH_MAP(BTN_7,			"BTN_7"),
	BH_MAP(BTN_8,			"BTN_8"),
	BH_MAP(BTN_9,			"BTN_9"),
	BH_MAP(KEY_BRIGHTNESS_ZERO,	"brightness_zero"),
	BH_MAP(KEY_CONFIG,		"config"),
	BH_MAP(KEY_COPY,		"copy"),
	BH_MAP(KEY_EJECTCD,		"eject"),
	BH_MAP(KEY_HELP,		"help"),
	BH_MAP(KEY_LIGHTS_TOGGLE,	"lights_toggle"),
	BH_MAP(KEY_PHONE,		"phone"),
	BH_MAP(KEY_POWER,		"power"),
	BH_MAP(KEY_RESTART,		"reset"),
	BH_MAP(KEY_RFKILL,		"rfkill"),
	BH_MAP(KEY_VIDEO,		"video"),
	BH_MAP(KEY_WIMAX,		"wwan"),
	BH_MAP(KEY_WLAN,		"wlan"),
	BH_MAP(KEY_WPS_BUTTON,		"wps"),
};

```


## 问题

在折腾OrangePi One设备按键的时候，浪费了将近两天的时间在gpio-button-hotplug不起作用上。 原因是package的dependency并不会强制使kmod模块被开启，我的config文件是从树莓派的config文件修改而来，导致即使有了

```makefile
CONFIG_DEFAULT_kmod-gpio-button-hotplug=y
```

也不代表下面这个被选中
```makefile
CONFIG_PACKAGE_kmod-gpio-button-hotplug=y
```

两天的大好时光啊~~


