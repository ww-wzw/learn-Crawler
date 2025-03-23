# 爬虫-Crawler_better

### 库

```python
import os
import time
import random
import signal
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin

os:操作文件系统
time:添加随机延迟，防止请求过于频繁
random:生成随机数（随机延迟和随机User-Agent）
signal:处理中断信号（Ctrl+C终止程序）
requests:发送HTTP请求获取内容和图片数据
BeautifulSoup:解析HTML网页，提取图片标签
urljoin:用于将相对URL转换为绝对URL
```

### 代码功能概述

- 随机User-Agent伪装请求，减少被封风险
- 信号处理，允许用户Ctrl+C终止爬虫
- 随机延迟，避免请求过于频繁
- 支持srcset高分辨率图片优先下载高清版本
- 错误处理，确保爬虫不会因异常崩溃

### 代码解析

#### 定义User-Agent池(反爬虫策略)

```python
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.0 Safari/605.1.15",
]
```

- 通过User-Agent伪装请求，避免被网站反爬机制屏蔽
- 在请求时随机选择一个User-Agent，模拟不同浏览器访问

#### 定义全局变量stop_flag及信号处理

```python
#全局变量，标记爬虫是否停止
stop_flag = False

#signal_handler()监听Ctrl+C(SIGINT信号)，当用户手动终止程序时将stop_flag设为True，从而停止爬虫
def signal_handler(sig, frame):
    global stop_flag
    print("收到停止信号，爬虫即将停止...")
    stop_flag = True

# 注册信号处理函数
signal.signal(signal.SIGINT, signal_handler)
```

##### 信号处理机制：

- SIGINT是终端中断信号
- 注册信号处理函数实现优雅停止
- 生产环境可能需要处理更多信号(如SIGTERM)

#### 生成请求头

```python
def get_random_headers(url):
    headers = {
        "User-Agent": random.choice(USER_AGENTS),#随机选择一个值，伪装请求
        "Referer": url,#避免某些网站拦截直接访问的请求
    }
    return headers
```

##### 请求头优化：

- Referer字段模拟正常访问流程
- 可添加Accept-Language、Accept等字段更真实
- 注意不要添加非法头字段（可能违反HTTP规范）

#### 添加随机延迟

```python
def add_delay():
    time.sleep(random.uniform(1, 3))
#让爬虫在1到3秒之间随机等待，降低被封IP的风险
```

##### 反封禁策略：

- 随机延时模拟人类操作
- 根据目标网站调整延时范围
- 分布式爬虫需要协调不同节点的请求频率

#### 图片下载函数

```python
def download_images(url, save_dir="images"):
    global stop_flag
    try:
        add_delay()#延迟
        response = requests.get(url, headers=get_random_headers(url))#发送HTTP请求
        response.raise_for_status()#自动检查HTT状态码
        soup = BeautifulSoup(response.content, "html.parser")#解析HTML，提取所有img标签；使用response.content而非response.text保持原始字节数据；选择html.parser解析器。无需安装，但lxml更快
        img_tags = soup.find_all("img")

        if not os.path.exists(save_dir):
            os.makedirs(save_dir)
		for img_tag in img_tags:
            if stop_flag:
                break
            img_url = img_tag.get("data-src") or img_tag.get("data-original") or img_tag.get("src")
#优先检查data-src/data-original(延迟加载技术)
            if img_url:#确保URL是绝对路径
                img_url = urljoin(url, img_url)

```

```python
                try:
        #再次添加延迟，避免请求过快
                    add_delay()
                    # 优先使用srcset中的高分辨率图片
                    srcset = img_tag.get('srcset')
                    if srcset:
                        urls = srcset.split(',')
                        high_res_url = urls[-1].split()[0]
                        img_url = urljoin(url, high_res_url)
                    #获取图片数据
                    img_data = requests.get(img_url, headers=get_random_headers(img_url)).content
                    img_name = os.path.basename(img_url)
                    img_path = os.path.join(save_dir, img_name)
                    with open(img_path, "wb") as f:
                        f.write(img_data)
                    print(f"Downloaded: {img_name}")
                except Exception as e:
                    print(f"Failed to download {img_url}: {e}")
				#提前退出函数，防止继续爬取   
                if stop_flag:
           				 print("爬虫已经停止")
           				 return
```

#### 主函数

```python
if __name__ == "__main__":
    target_url = input("请输入要爬取的网页 URL: ")
    save_directory = input("请输入保存图片的目录 (默认为 'images'): ") or "images"
    download_images(target_url, save_directory)

```

### 优化修改

#### 日志记录

```python
import logging
logging.basicConfig(filename='crawler.log',level=logging.INFO,format='%(asctime)s-%(levelname)s-%(message)s')
try:
    #爬取代码
except Exception as e:
    logging.error(f"Error processing {url}:{str(e)}")
```

