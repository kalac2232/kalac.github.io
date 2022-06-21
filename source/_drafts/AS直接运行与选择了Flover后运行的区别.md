---
title: AS直接运行与选择了Flover后运行的区别
date: 2021-12-28 15:15:24
tags: Android
categories: Android
---



通过分析Gradle 执行Task可知

1. AS直接运行其实是执行的assemble task，

   任务完成后，由Android studio自己拉起App的laucher，猜测执行的命令为`adb -s<目标的serial number> device shell install xxx.apk`

2. 而flover是installTask

   只负责打包，完成后不自动拉起app，并且在执行安装过程中不受Android Studio中选中的目标设备影响（AS设备列表为灰色且不可切换），目前连接几个设备就会依次遍历设备进行安装。

   *执行的方式为Android Gradle Plugin中的InstallVariantTask.install方法*

