---
title: mac安装dex2jar并更新lib
date: 2021-05-12 11:02:56
tags: 反编译
---

## 通过brew安装

```
brew install dex2jar
```

## 替换lib

因为d2j-2.0会在解析Android8.0的dex时，报`Dexexception: not support version`错误，故使用[DexPatcher-dex2jar](https://github.com/DexPatcher/dex2jar/releases/tag/v2.1-20190905-lanchon) 替换2.0版本

1. 进入本地brew安装的dex2jar目录：`/usr/local/Cellar/dex2jar/2.0/libexec`
2. 删除lib目录下所有jar
3. 解压下载的`dex-tools-2.1-20190905-lanchon`，将lib中的全部文件移动到`/usr/local/Cellar/dex2jar/2.0/libexec/lib`下即可

