+++
title = "tar.gzの解凍と圧縮に関するコマンド"
date = "2023-01-16T15:40:09+09:00"
author = ""
authorTwitter = "cohsh_" #do not include @
cover = ""
tags = ["備忘録", "Shell"]
keywords = ["", ""]
description = "調べても直ぐに忘れるので、自分のためにまとめました。"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

調べても直ぐに忘れるので、自分のためにまとめました。

#### 解凍: `hogehoge.tar.gz` → `hogehoge/`
```
tar zxvf hogehoge.tar.gz
```

#### 圧縮: `hogehoge/` → `hogehoge.tar.gz`
```
tar zcvf hogehoge.tar.gz hogehoge
```

#### 備考
- オプションのハイフンは省略できる。
