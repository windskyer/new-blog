---
title: windows 激活
categories: Kubernetes
sage: false
date: 2019-10-08 15:52:36
tags: windows
---

## windows kms激活服务

windows 系统激活命令

```bash
set ipk=${ipk}
set kms_addr=${kms_addr}
cscript //Nologo %windir%\system32\slmgr.vbs /ipk %ipk%
cscript //Nologo %windir%\system32\slmgr.vbs -skms %kms_addr%
cscript //Nologo %windir%\system32\slmgr.vbs -ato
```

<!-- more -->

客户端激活ipk：

|操作系统|ipk|
|---|---|
|Windows Server 2012 DC R2 32/64bit|    W3GGN-FT8W3-Y4M27-J84CP-Q3VJ9|
|Windows Server 2008 DC R2 32/64bit|    74YFP-3QFB3-KQT8W-PMXWJ-7M648|
|Windows Server 2008 DC R1 32/64bit|    7M67G-PC374-GR742-YH8V4-TCBY3|
|Windows Server 2008 enterprise 32/64bit|   YQGMW-MPWTJ-34KDK-48M3W-X4Q6V|
|Windows Server 2008 enterprise R2 32/64bit|    489J6-VHDMP-X63PK-3K798-CPX3Y|
|Windows7 Professional 32/64bit|    FJ82H-XT6CR-J8D7P-XQJJ2-GPDD4|
|Windows7 Enterprise 32/64bit|  33PXH-7Y6KF-2VJC9-XBBR8-HVTHH|
|windows_10_enterprise_x64_zh|  NPPR9-FWDCX-D2C8J-H872K-2YT43|
|Windows Server 2016 DC R2 64bit|   CB7KF-BWN84-R7R2Y-793K2-8XDDG|

kms服务器地址:

xxx.xxx.xxx.xxx

## 验证结果

打开cmd 输入slmgr /dlv 查看详细 的 kms 信息
