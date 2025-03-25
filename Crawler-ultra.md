# 爬虫—Crawler-ultra

### 基础库导入

```python
import os#文件系统操作
import re#正则表达式处理
import time#延时功能
import random#随机选择IP/代理
import signal#信号处理（优雅停止）
import requests#http请求核心库
import pandas as pd数据存储为csv
from bs4 import BeautifulSoup#HTML/XML解析
from urllib.parse import urljoin,urlparse#URL处理
```

### 全局配置

```python
USER_AGENTS=[...]#伪装不同浏览器，防止反爬
PROXY_LIST=[...]#代理IP池（需替换为有效代理）
stop_flag=False#程序停止标志
```

### 信号处理模块

```python
def signal_handler(sig,frame):
    global stop_flag
    print("收到停止信号，爬虫即将停止")
    stop_flag=True
signal.signal(signal.SIGINT,signal_handler)
```

signal模块的作用是为Python程序提供信号处理的功能。信号是在Unix和类Unix系统里用于进程间通信的一种机制，可用来通过之进程发生了某些事

signal.SIGINT是一个信号常量，它代表的是由用户在终端按下Ctrl+C组合键时所产生的中断信号

signal_handler是一个自定义的信号处理函数，当接收到SIGINT信号时，该函数就会被调用

### 请求伪装模块

```python
def get_random_headers(url):
    return{
        "User-Agent":random.choice(USER_AGENTS),
        "Referer":url
    }
def get_random_proxy():
    return random.choice(PROXY_LIST) if PROXY_LIST else None
def add_delay():
    time.sleep(random.uniform(1,3))
```

random.choice，从一个非空序列（列表、元组等）中随机选取一个元素

PROXY_LIST是一个包含代理信息的列表，列表中的元素可以是代理服务器的地址、端口号等信息

- 动态切换User-Agent和Referer
- 代理IP轮换+请求间隔控制

### 文件处理模块

```python
def sanitize_filename(filename):
    return re.sub(r'[\V:*?"<>|]',"",filename)#移除非法字符
def get_filename_from_url(url,content_type):
    parsed_url=urlparse(url)
    path =parsed_url.path
    filename=os.path.basename(path) or "unknown"
    if "." not in filename:#补充扩展名
        ext=content_type.split("/")[-1]
        filename += "." +ext if ext else ""
    return sanitize_filename(filename)
```

- 自动处理URL中的特殊字符（如中文文件名）
- 根据Content-Type补充文件扩展名

parsed_url=urlparse(url)，urlparse函数会把URL分解成多个组件，像协议、域名、路径、查询参数、片段等，然后返回一个包含这些组件的ParseResult对象

parsed_url.path是ParseResult对象的一个属性，代表的是URL中域名之后、查询参数之前的部分

filename=os.path.basename(path) or "unknown"，从路径中提取出文件名部分

### 代理验证模块

```python
def validate_proxy(proxy):
    try:
        requests.get("http://www.hippopx.com",proxies=proxy,timeout=5)
        return True
    except Exception:
        return False
```

requests.get()是requests库中的一个函数，其作用：向指定的URL发送HTTP GET请求，并返回一个Response对象

proxies是requests.get函数的一个可选参数，允许通过代理服务器来发送请求

proxy是一个字典，用来指定代理服务器的信息。键是协议名（http,https），值是代理服务器的地址和端口号

timeout是可选参数，指定了请求的超时时间

### 核心下载函数

```python
def download_resourse(url,save_dir,headers,proxies):
    try:
        add_delay()
        #代理有效性判断
        if proxies and validate_proxy(proxies):
            response=requests.get(...,proxies=proxy)
        else:
            response =requests.get(...)
        #分块下载大文件
        with open(file_path,"wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                if chunk:
                    f.write(chunk)
        except:
```

### 文本提取模块

```python
def extract_text(soup):
    for script in soup(["script","style"]):
        script.decompose()#移除脚本和样式
    text=soup.get_text()
    #清理空白字符
    lines=(line.strip() for line in text.splitlines())
    chunks=(phrase.strip() for line in lines for phrase in line.split(" "))
    return "\n".join(chunk for chunk in chunks if chunk)
```

script.decompose，将script这个标签及其包含的所有子标签和文本内容从文档树中移除，并且释放相关的内存资源

### 主爬虫逻辑

```python
def crawl_resources(...):
    # 创建保存目录
    os.makedirs(os.path.join(save_dir, "images"), exist_ok=True)
    # 图片下载
    img_tags = soup.find_all("img")
    img_url = img_tag.get("data-src") or ... # 优先抓取延迟加载内容
    # 文本保存
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(text)
    # 元数据存储
    df = pd.DataFrame(downloaded_data)
    df.to_csv(...) 
```

