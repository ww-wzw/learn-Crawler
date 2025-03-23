# 爬虫-Crawler_simple

### 库

```python
import os
import re
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin

os提供与系统交互的功能，如文件/目录操作
re是python的正则表达式库，用于字符串匹配、查找、替换等
requests用于发送HTTP请求(如GET、POST)，爬取网页内容或与API交互
bs4(BeautifulSoup)用于解析HTML/XML数据，方便提取网页内容
urllib.parse(urljoin)用于处理URL，例如拼接相对路径和基础URL
```

### 代码功能概述

- 从指定的URL页面提取所有img标签的src属性（即图片链接）
- 处理相对URL，确保所有图片链接都是完整的
- 下载这些图片并保存到images目录下
- 处理异常，如无法访问网站、下载失败、非图像文件等

### 代码解析

#### 尝试请求网页

```python
try:
    response = requests.get(url)#发送HTTP GET请求获取网页内容
    response.raise_for_status()#确保请求成功，如果服务器返回4xx/5xx错误，会抛出异常
#检查response.status_code来判断具体的错误
if response.status_code==404:
    print("网页不存在!")
elif resonse.status_code == 403:
    print("访问被禁止!")
```

#### 解析HTML，获取所有图片标签

```python
soup = BeautifulSoup(response.content, "html.parser")#用bs4解析网页内容
img_tags = soup.find_all("img")#获取所有<img>标签
#获取所有超链接
links=soup.find_all("a")
for link in links:
    print(link.get("href"))#获取链接地址
```

#### 创建存储图片的目录

```python
if not os.path.exists(save_dir):#检查目录是否存在
    os.makedirs(save_dir)#如果不存在就创建目录
#创建多级目录
os.makedirs("downloads/images/thumbnails",exist_ok=True)
```

#### 处理每个图片的URL

```python
for img in img_tags:
    img_url = img.get("src")#获取图片的URL
    if img_url:
        absolute_url = urljoin(url, img_url)#处理相对路径，确保img_url是完整的URL
#如果 img_url是相对路径（如 /images/logo.png），urljoin() 会自动拼接成 https://example.com/images/logo.png;绝对路径（如 https://example.com/logo.png），urljoin() 不会改变。
```

#### 下载图片

```python
try:
    img_data = requests.get(absolute_url, stream=True)#以流模式逐步下载图片
    img_data.raise_for_status()#确保下载成功
```

#### 检查是否为图片

```python
if 'image' in img_data.headers.get('Content-Type', ''):
#获取文件类型，确保是图片
```

- image/jpeg(JPG)
- image/png(PNG)
- text/html(网页)

#### 规范化文件名

```python
filename = os.path.basename(absolute_url)#获取URL的文件名
filename = re.sub(r'[^\w\-. ]', '_', filename)#替换非法字符，避免文件名错误
#添加时间戳避免重名
import time
fiilename=f"{int(time.time())}_{filename}"
```

#### 保存图片

```python
filepath = os.path.join(save_dir, filename)#确保文件存储路径正确

with open(filepath, "wb") as f:#以二进制写入模式打开文件
    for chunk in img_data.iter_content(chunk_size=8192):#分块写入，防止一次性加载大文件导致内存占用过高
        f.write(chunk)
        
#打印成功消息
print(f"已下载：{filename}")
```

#### 处理异常

```python
except requests.exceptions.RequestException as e:
    print(f"下载 {absolute_url} 失败：{e}")
#如果下载失败，就会打印错误信息
```

### 优化修改

#### 设置User-Agent

爬取网页时，服务器会检查请求头，其中user-agent是最重要的字段之一，标识访问者的设备、浏览器、操作系统

若user-agent不符合浏览器格式，造成

- 返回403禁止访问
- 直接封锁IP
- 返回假数据
- 要求输入验证码