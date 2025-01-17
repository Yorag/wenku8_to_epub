# wenku8-to-epub

[轻小说文库](https://www.wenku8.net)在线小说转epub



## 项目简介

### 1. 功能

- **按卷下载**、**指定卷下载**、**整本下载**小说各章节（含插图页）并封装至epub，并且通过了calibre检错
- 从Web端/APP端获取章节内容
- 填写小说元数据（标题、作者、介绍、出版商、标签、封面）
  - 支持[calibre](https://github.com/kovidgoyal/calibre)自有元数据（丛书series、丛书编号series_index）
- 生成小说目录

> 所有数据源自[轻小说文库](https://www.wenku8.net/)。

### 2. 功能演示

![分卷下载](./screenshot/image-20240510170318408.png)

![整本下载](./screenshot/image-20240509114425897.png)



## 如何使用

> 首先需要python3配置环境。

在运行前需要安装依赖包：

```python
pip install -r requirements.txt
```

请确保工作目录是项目根目录，然后在终端输入以下指令：

```python
python main.py
```

输入要下载的book_id，根据提示输入指令，等待下载完成。

> 封装好的epub小说默认保存在项目根目录`epub`文件夹下。



**可调参数**（在`main.py`文件内）

| 参数名                   | 默认值           | 描述                                                         |
| :----------------------- | ---------------- | ------------------------------------------------------------ |
| `save_epub_dir`          | `epub`           | 打包的epub存储目录（相对路径/绝对路径）                      |
| `sleep_time`             | `2`              | 每次网络请求后停顿时间（秒）                                 |
| `use_divimage_set_cover` | `True`           | 是否将插图第一张长图设为封面。若否就使用小说详情页封面       |
| `wenku_host`             | `www.wenku8.com` | 访问wenku8的主机名                                           |
| `wenkupic_proxy_host`    | `None`           | 反代`pic.wenku8.com`的host：`xxxx.xxxx.workers.dev` 或 自定义域名 |
| `wenkuapp_proxy_host`    | `None`           | 反代`app.wenku8.com`的host：`xxxx.xxxx.workers.dev` 或 自定义域名 |



## 常见问题

### 1. SSLError

出现`requests.exceptions.SSLError: HTTPSConnectionPool(host='xxx.wenku8.com', port=443)`

**原因：** 可能是本地网络环境问题。

**解决：**

方法1：换个网络环境再试试。

方法2：使用[cloudflare workers](https://www.cloudflare-cn.com/developer-platform/workers/)反代报错域名`pic.wenku8.com`。

> 可临时使用作者搭建的反代域名`wk8-test.jsone.gq`，不保证长期有效。

- 新建一个cloudflare workers；
- 部署。选择编辑代码，将下面代码粘贴进编辑器后部署；

  > 下面代码可以设置代理多个域名，通过路径区分。

```js
const host_mapping = {
    // 'app.wenku8.com': ['/android.php'],
    'pic.wenku8.com': ['/pictures/']
};

addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
    var u = new URL(request.url);
    var path = u.pathname;
    
    var proxy_host = null;
    for (const host in host_mapping) {
        const paths = host_mapping[host];
        for (const p of paths) {
            if (path.startsWith(p)) {
                proxy_host = host;
                break;
            }
        }
        if (proxy_host !== null) { break; }
    }

    if (proxy_host !== null) {
        u.host = proxy_host;
        var req = new Request(u, {
            method: request.method,
            headers: request.headers,
            body: request.body
        });
        const result = await fetch(req);
        return result;
    } else {
        return new Response('Not Found', {status: 404});
    }
}
```

- 将反代的网址粘贴到main.py的 wenkupic_proxy_host / wenkuapp_proxy_host变量处，如`wenkupic_proxy_host = xxxx.xxxxx.workers.dev`





### 2. Access denied

出现`Access denied | www.wenku8.net used Cloudflare to restrict access`。

**原因：** 访问过于频繁，请求受限。

**解决：** 增加请求延迟，自定义参数增加`sleep_time`的值。





## 其他说明

- 同一系列的不同卷用的还是同一组元数据，所以目前有书名/封面区分各卷。
  > 不同卷一般有两个元数据需要变动：封面和介绍。 <br>
  >
  > - 封面：默认使用插图的第一张长图（如果有，没有就使用详情页的缩略图），可以通过修改`main.py`的`use_divimage_set_cover`关闭。<br>
  >
  > - 介绍：目测wenku8只有详情页有，其他分卷没有，所以目前同一系列的不同卷用的是一个介绍。
- 目测轻小说文库的小说目录只有两级`卷 -> 章`，所以分卷下载时不考虑分级（卷内全都是一级目录），整本下载时分两级。
