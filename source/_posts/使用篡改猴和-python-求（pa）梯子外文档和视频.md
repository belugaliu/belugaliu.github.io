---
title: 使用篡改猴和 python 求（pa）梯子外文档和视频
date: 2024-10-25 18:11:02
categories:
  - code
tags:
  - code
  - tampermonkey
  - python
---

半年前，我受朋友委托请求帮忙 pa 国外网站学习资源。时间有点久，最近有空闲时间就来看看能不能搞。pa 这套资源，分析后有几个情况

1. 该网站是有 2FA 认证

2. PDF 是使用 chrome 内置程序展示，好在页面上提供了下载图标和链接

3. 视频是使用 video 标签展示并支持播放，视频是 mp4，被分片为 ts 由 m3u8 记录分片列表


其中，文档资源的下载链接需要通过网站登录验证，所以这类资源最好是通过浏览器下载。因为一般的绕不过 2FA 认证。视频资源的下载链接不需要通过网站登录验证，下载流量是直接打到网站提供的存储，这对爬虫来说是个福音。随后确定文档类资源使用篡改猴下载，视频类资源使用 python 下载。

### 文档类资源下 (pa) 载 (chong)

[篡改猴 tampermonkey](https://www.tampermonkey.net/index.php?src=a&locale=zh_CN#google_vignette)，它允许用户自定义并**增强您最喜爱的网页的功能**。用户脚本是小型 JavaScript 程序，可用于向网页添加新功能或修改现有功能。使用 篡改猴，您可以轻松在任何网站上创建、管理和运行这些用户脚本。可以使用 blob 将视频类资源下载地址保存到本地

```javascript
const videoUrls = [];
const videoBlob = new Blob([videoUrls], {type: 'text/plain'});
const videoLink = document.createElement('a');
videoLink.href = window.URL.createObjectURL(videoBlob);
videoLink.download=course +'_'+item+'_video.txt';
videoLink.click();
```

关于文档下载 javascript 直接使用 `window.open(downloadUrl, '_blank')` 完成下载。需要在浏览器设置访问 PDF 时是直接下载，不是在 chrome 中打开 PDF 文件，访问 `chrome://settings/content/pdfDocuments` 可修改，记得重启浏览器。

#### chrome 下载历史记录在哪里

我本次的 pa 虫，需要按照页面进行文档分类，所以需要按照下载地址映射下载的文件，这需要 chrome 下载历史记录。而 chrome 下载历史记录，在 `${chrome_data_home}/History` 文件中，`${chrome_data_home}` 来自于 `chrome://version/` 的**个人资料路径**。History 文件是 sqlite 数据库文件。剩下的你懂的。

### 视频类资源下 (pa) 载 (chong)

python 视频下载代码比较多，而 ts 和 m3u8 文件下载完之后，需要使用 ffmpeg 完成视频的合并。ffmpeg 需要下载，编译后的 ffmpeg [下载地址](https://www.osxexperts.net/)，自己编译[访问地址](https://ffmpeg.org/download.html)。整个的下载代码如下

```python
import os.path
import urllib.parse
import requests
import shutil
import subprocess
import concurrent.futures


def download_mp4_part_file(dir_path, video_url):
    download_part_sfile(dir_path, video_url)
    video_file_name = video_url[video_url.rindex('/')+1:]
    video_file_prefix = video_url[0: video_url.rindex('/')]
    video_file_path = dir_path + '/' + video_file_name
    if video_file_name.find('m3u8') > -1:
        with open(video_file_path, "r") as f:
            lines = f.readlines()
        ts_list = []
        for line in lines:
            if not line.startswith('#'):
                if line.find('m3u8') > -1:
                    download_mp4_part_file(dir_path, video_file_prefix + "/" + line.strip())
                elif line.find('.ts') > -1:
                    ts_list.append(video_file_prefix+'/'+line.strip())
        if len(ts_list) > 0:
            with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
                futures = [executor.submit(download_part_sfile, dir_path, ts_url) for ts_url in ts_list]
                for future in concurrent.futures.as_completed(futures):
                    result = future.result()


def download_part_sfile(dir_path, video_url):
    video_file_name = video_url[video_url.rindex('/') + 1:]
    video_file_path = dir_path + '/' + video_file_name
    if not os.path.exists(video_file_path):
        response = requests.get(video_url)
        with open(video_file_path, 'wb') as file:
            file.write(response.content)
    return True


def download_file(path, module_name, module_video, video_url):
    video_name = module_video[module_video.rindex('/')+1:]
    video_name = urllib.parse.unquote(video_name)
    video_part_name = video_name[0: video_name.rindex('.')]
    video_dir_path = path + '/' + module_name + '/' + video_part_name
    if os.path.exists(video_dir_path+'.mp4'):
        return
    if not os.path.exists(video_dir_path):
        os.mkdir(video_dir_path, 0o777)
    download_mp4_part_file(video_dir_path, video_url)
    try:
        video_file_name = video_url[video_url.rindex('/')+1:]
        subprocess.run(['ffmpeg', '-i', video_dir_path+'/' + video_file_name, '-c', 'copy', video_dir_path+'.mp4'])
    except BaseException as exc:
        print(module_video + "," + video_url + "合并视频失败" + exc)
    finally:
        shutil.rmtree(video_dir_path)

def download(path, module_map, videos, video_map):
    for module_key, module_videos in videos.items():
        module_name = module_map[module_key]
        bilibili_videos = []
        for module_video in module_videos:
            if module_video.find('bilibili') > -1:
                bilibili_videos.append(module_video)
            else:
                video_url = video_map[module_video]
                download_file(path, module_name, module_video, video_url)
        other_video_path = path + "/" + module_name + "/other_video_url.txt"
        other_video_file = open(other_video_path, "w")
        for bilibili_video in bilibili_videos:
            other_video_file.write(bilibili_video)
            other_video_file.write("\n")
        other_video_file.close()
```

### 附录

- [chrome 用户目录文件解析](https://www.foxtonforensics.com/browser-history-examiner/chrome-history-location)