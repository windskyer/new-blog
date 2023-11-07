---
title: python 图片文字识别+二维码识别
categories: Python
sage: false
date: 2020-03-11 14:55:26
tags: python
---

# 文字识别

python的pytesseract为文字识别提供了很好的支持。整个实现只需要一行关键代码即可。

## 前提安装

```bash
yum install -y tesseract-langpack-chi_sim tesseract-langpack-chi_tra tesseract
pip install pytesseract
```
## 代码示例

```python
from PIL import Imageimport
import pytesseract
text=pytesseract.image_to_string(Image.open(file_path), lang='chi_sim')
print(text)
```

识别语言： 中文简体(chi_sim), 繁体(chi_tra)

<!-- more -->

# 二维码识别
在没接触 Python 之前，曾使用 Zbar 的客户端进行识别，测了大概几百张相对模糊的图片，Zbar的识别速度要快很多，识别率也比 Zxing 稍微准确那边一丢丢，但是，稍微模糊一点就无法识别。

## 前提安装

```bash
# 升级 pip 并安装第三方库
pip install -U pip
pip install Pillow
pip install pyzbar
pip install qrcode
```

## 代码示例

```python
from PIL import Imageimport
import pyzbar.pyzbar as pyzbar

bar_codes = pyzbar.decode(Image.open(file_path))
for bar_code in bar_codes:
    bar_code_info += bar_code.data.decode("utf-8")
print(bar_code_info)
```

# 整体代码

```python
# -*- coding: utf-8 -*-

import io
import requests
import pytesseract
from PIL import Image
import pyzbar.pyzbar as pyzbar


def check_image_qrcode(image):
    bar_code_info = ""
    buf_image = io.BytesIO()
    ir = requests.get(image, stream=True)
    if ir.ok:
        for chunk in ir:
            buf_image.write(chunk)
    else:
        return bar_code_info
    img = Image.open(buf_image)
    if img:
        bar_codes = pyzbar.decode(img)
        for bar_code in bar_codes:
            bar_code_info += bar_code.data.decode("utf-8")

    if len(bar_code_info) > 0:
        return bar_code_info

    if img:
        bar_code_info = pytesseract.image_to_string(img, lang='chi_tra')

    return bar_code_info



def main():
    image = "https://pic2.zhimg.com/80/v2-50eaea949ac63de5d5a84813d9efe491_720w.jpg"
    image_info = check_image_qrcode(image)
    if len(image_info) == 0 :
        print("11111")
    else:
        print(image_info)

if __name__ == "__main__":
    main()
```