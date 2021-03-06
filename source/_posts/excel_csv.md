---
title: Excel打开csv文件乱码的解决办法
date: 2016-04-21 11:40:01
tags: [excel,csv]
categories: [其他]
---

## 产生背景

 > 今天在用excel打开一个脚本生成的csv文件（存储编码格式为utf-8）时出现了乱码情况，但是用WPS直接打开却正常显示。因为文件是要交给客户的，office又是普遍使用的一枚办公软件，总不能让客户去安装一个WPS吧。网上一番查阅后，众说纷纭，因此特此记录下自己所踩过的坑：

## 解决方案

* 用记事本打开csv文件，另存为Unicode格式
* 之后用excel打开CSV文件，注意此时该文件的编码已是Unicode
* 若出现每一行所有字段在一个单元格的情况，解决步骤接着看下面
* 重新打开excel，执行"数据"->"自文本"->选择csv文件->"导入"->出现文本导入导向对话框->"下一步"->取消Tab键，选中逗号作为分隔符号->"确定"
* 待转换成功，则会在excel中正常显示

-------------------

## 原因分析

	Excel默认打开文件的编码格式是Unicode，所以当文件里面同时含有中文、韩文、西欧字符等等的时候，此时若文件为非Unicode格式，由于编码格式不一致，将会出现乱码问题。
