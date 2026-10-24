# PCL 下载解锁器

把 PCL 的 exe 拖进网页，浏览器本地完成修复，输出 `*_unlocked.exe`。
整个过程不联网、不上传，文件只在你的机器内存里过一遍。

## 文件

| 文件 | 说明 |
|---|---|
| `index.html` | 单文件网页工具，双击即用，支持批量拖入 |
| `pcl_download_unlock.zip` | Python 命令行版，解压后 `python pcl_download_unlock.py` |
| `Plain Craft Launcher 2_unlocked.exe` | 已处理好的成品，可直接使用 |

## 网页版

1. 双击 `index.html`
2. 把一个或多个 PCL exe 拖进虚线框
3. 进度条走完，点“下载”
4. 得到 `原文件名_unlocked.exe`

页面里的 JS 与 Python 工具的字节补丁完全一致，输出文件哈希相同。

## 命令行版

```bat
python pcl_download_unlock.py            :: 自动查找并修复
python pcl_download_unlock.py -f x.exe   :: 指定文件
python pcl_download_unlock.py -d         :: 只预演
python pcl_download_unlock.py --restore x.exe
```

会自动备份为 `*.bak`，可用 `--restore` 还原。

## 说明

仅用于处理自己持有的样本，请勿用于其它用途。
