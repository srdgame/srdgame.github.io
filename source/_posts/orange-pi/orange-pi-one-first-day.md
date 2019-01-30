---
title: Orange Pi One First Day
date: 2019-01-27 11:48:14
categories:
	- openwrt
	- orangepi
tags:
	- openwrt
	- orangepi
---

# Orange Pi One 上手日记

## Why Orange Pi One

刚刚入手OrangePi One，主要目标是选定一款国内设计生产的单板机，方便19年的市场宣传。

手头也弄好了树莓派3B+，OpenWRT的编译也做过了，WIFI驱动缺少的bin文件什么的（更新了一下openwrt master,这部分已经集成无需再折腾）。其实我并不满意树莓派,主要是串口太少，默认也没启用调试串口(虽然可以通过显示器来显示信息）。这对于FreeIOE来说是个天生的不足，使用扩展卡不利于做市场宣传。

国内不少单板机，BananaPi, NanoPi等等。最终选择OrangePi的原因是他的板子价格诚意更高，社区资料也算国内比较完善的，CPU是全志的，也能算得上鼓励国产？

## 亮照

![OrangePi One 包装盒](orangepi-one-package.jpg "包装盒")


![OrangePi One 和树莓派3B+](orangepi-one-and-raspberry3b-plus.jpg "合照")


## 移植OpenWRT 18.06

### OrangePi PC固件

OrangePi PC固件是能让OrangePi One跑起来的，直接<code>`make menuconfig`</code>时选择Orange Pi PC即可编译生成镜像文件。(直接解压ext4的img文件，然后dd到tf卡上即可）


### OrangePi One专属固件

One和Pc还是有一些差别的，比如电源芯片部分PC用的是SY8106A，同时从sunxi的uboot configs里面看到CONFIG_DRAM_CLK也不一致。
> http://linux-sunxi.org/Table_of_Allwinner_based_boards


#### OpenWRT的改动:

1. u-boot

```patch
--- a/package/boot/uboot-sunxi/Makefile
+++ b/package/boot/uboot-sunxi/Makefile
@@ -156,6 +156,12 @@ define U-Boot/orangepi_pc
   BUILD_DEVICES:=sun8i-h3-orangepi-pc
 endef
 
+define U-Boot/orangepi_one
+  BUILD_SUBTARGET:=cortexa7
+  NAME:=Orange Pi One (H3)
+  BUILD_DEVICES:=sun8i-h3-orangepi-one
+endef
+
 define U-Boot/orangepi_plus
   BUILD_SUBTARGET:=cortexa7
   NAME:=Orange Pi Plus (H3)
@@ -231,6 +237,7 @@ UBOOT_TARGETS := \
        nanopi_neo_plus2 \
        orangepi_r1 \
        orangepi_pc \
+       orangepi_one \
        orangepi_plus \
        orangepi_2 \
        pangolin \

```

2. target

```patch
--- a/target/linux/sunxi/image/cortex-a7.mk
+++ b/target/linux/sunxi/image/cortex-a7.mk
@@ -149,6 +149,16 @@ endef
 TARGET_DEVICES += sun8i-h3-orangepi-pc
 
 
+define Device/sun8i-h3-orangepi-one
+  DEVICE_TITLE:=Xunlong Orange Pi One
+  DEVICE_PACKAGES:=kmod-rtc-sunxi kmod-gpio-button-hotplug
+  SUPPORTED_DEVICES:=xunlong,orangepi-one
+  SUNXI_DTS:=sun8i-h3-orangepi-one
+endef
+
+TARGET_DEVICES += sun8i-h3-orangepi-one
+
+
 define Device/sun8i-h3-orangepi-plus
   DEVICE_TITLE:=Xunlong Orange Pi Plus
   DEVICE_PACKAGES:=kmod-rtc-sunxi

```

3. 串口启用(非必要)

以下是 patches-4.14/900-ARM-dts-sun8i-kooiot-Orange-Pi-One.patch 文件内容
```patch
--- a/arch/arm/boot/dts/sun8i-h3-orangepi-one.dts
--- b/arch/arm/boot/dts/sun8i-h3-orangepi-one.dts
@@ -84,7 +84,7 @@

 		sw4 {
 			label = "sw4";
-			linux,code = <BTN_0>;
+			linux,code = <KEY_RESTART>;
 			gpios = <&r_pio 0 3 GPIO_ACTIVE_LOW>;
 		};
 	};
@@ -156,19 +156,19 @@
 &uart1 {
 	pinctrl-names = "default";
 	pinctrl-0 = <&uart1_pins>;
-	status = "disabled";
+	status = "okay";
 };
 
 &uart2 {
 	pinctrl-names = "default";
 	pinctrl-0 = <&uart2_pins>;
-	status = "disabled";
+	status = "okay";
 };
 
 &uart3 {
 	pinctrl-names = "default";
 	pinctrl-0 = <&uart3_pins>;
-	status = "disabled";
+	status = "okay";
 };
 
 &usb_otg {
```

至此<code>`make menuconfig`</code>并选中ext4镜像文件，之后就是等电脑干活了 -.-


## 镜像下载地址

[百度网盘下载地址](https://pan.baidu.com/s/1s1sehr_goxokSe5H04VBsg)


## TODO

1. 重启键

重启按键绑定到KEY_RESTART也并没有启作用，这几天再研究一下。

[2019-01-30 补充]
由于config文件是从其他配置修改而来，导致kmod-gpio-button-hotplug并没有选中(CONFIG_PACKAGE_kmod-gpio-button-hotplug=y)。而选中此选项后，才能保证按键会正确触发/etc/rc.button中的脚本。

我已经将sw4案件改名为power，按下会让OrangePi One关机。(网盘镜像已经更新，如需指向其他案件脚本: 1. dts中改名 2. 更改/etc/rc.button/power脚本

```path
--- a/arch/arm/boot/dts/sun8i-h3-orangepi-one.dts
--- b/arch/arm/boot/dts/sun8i-h3-orangepi-one.dts
@@ -82,9 +82,9 @@
 		pinctrl-names = "default";
 		pinctrl-0 = <&sw_r_opc>;
 
-		sw4 {
-			label = "sw4";
-			linux,code = <BTN_0>;
+		power {
+			label = "power";
+			linux,code = <KEY_POWER>;
 			gpios = <&r_pio 0 3 GPIO_ACTIVE_LOW>;
 		};
 	};
```


