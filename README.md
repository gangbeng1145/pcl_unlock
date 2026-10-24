立即使用：https://pcl.gber.dpdns.org/

备用网址：https://pcl-unlock.pages.dev/

备用网址：https://gangbeng1145.github.io/pcl_unlock/

# PCL 下载解锁器

> **本项目由 AI 辅助完成。**
> 样本的静态分析、IL 定位、补丁方案设计、Python 工具与网页工具的实现，均在 AI 协作下产出；
> 人工负责需求判断、方案取舍、真机验证与发布。

---

## 背景

样本是一个被改过的 PCL（Plain Craft Launcher 2），版本号被标成 `2.99.0.0`，
启动时会弹一条更新日志：

```
删除：为遵守相关法律法规，PCL 暂时不再提供 MC 下载功能，非常抱歉 qwq……
```

点“下载”会直接报错，MC 版本列表永远是空的。

目标：把下载功能恢复，并且做成一个不依赖人工分析、可重复执行的工具。

---

## 样本指纹

| 项 | 值 |
|---|---|
| 类型 | PE32 / .NET Framework 4.0 |
| 大小 | 5,980,160 字节 |
| 签名 | 无 |
| 混淆 | 无，类型名与方法名全部可读 |
| 元数据 | 665 个 TypeDef / 6,339 个 MethodDef |
| 原始 SHA256 | `6b6e1260cb5a1d6a303855f1c60cab3d0bc1aebba7e40a964180188b271e7386` |

因为没有混淆，直接按 ECMA-335 手写了一个元数据 + IL 解析器（不依赖 dnlib / Mono.Cecil），
就能拿到完整的方法名、字符串引用和调用关系。

---

## 原理

### 第一处：偷换 Loader 参数

`PCL.PageDownloadInstall.LoaderInit` 原本应该把 `DlClientListLoader`（真实的版本列表 Loader）
传给 `PageLoaderInit`，由它自动启动。样本里被换成了：

```vb
New LoaderTask(Of String, DlClientListResult)(
    "关闭下载",
    Sub() Throw New Exception("为遵守相关法律法规，PCL 暂时不再提供 MC 下载功能……"),
    Nothing, 2)
```

后果有两个：

1. 页面拿不到版本列表；
2. 这个假 Loader 一执行就抛异常，UI 直接显示“关闭下载”并失败。

日志能直接看到：

```
[ModLoader] 加载器 LoaderTask`2 关闭下载 状态改变：Loading
[ModLoader] 加载器 LoaderTask`2 关闭下载 状态改变：Failed
   在 PCL.PageDownloadInstall._Closure$__._Lambda$__3-0
```

### 第二处：在方法出口插 throw

`PCL.ModModpack.ModpackInstall` 的正常出口 `IL_02DA` 被塞进了同样的
`ldstr + newobj Exception + throw`，所以整合包安装这条路径也走不通。

### 为什么改文案没用

那句“为遵守相关法律法规”只是**字符串**，改掉它不影响逻辑。
真正的闸门是 `LoaderInit` 的参数槽位——把假 Loader 换掉，功能就回来了。

---

## 补丁

只改 `LoaderInit` 里的 48 字节：

```
原： ldstr "关闭下载" → 委托构造 → newobj LoaderTask
新： ldsfld DlClientListLoader + 43 个 nop
```

字节级特征：

```
SIG_OLD = 72 33 47 01 70 7E FB 09 00 04 2C 07 7E FB 09 00 04 2B 16
          7E F9 09 00 04 FE 06 67 14 00 06 73 8D 05 00 0A 25 80 FB
          09 00 04 14 18 73 8E 05 00 0A

SIG_NEW = 7E 0F 04 00 04 00 × 43
```

替换后文件大小不变，节表、入口点、CLR 目录全部不动。

---

## 踩过的坑

| 次数 | 现象 | 原因 | 修正 |
|---|---|---|---|
| 1 | `InvalidProgramException`（公共语言运行时检测到无效程序） | 把 `ret` 放在方法开头、末尾全是 `nop`，IL 校验器认为方法没有以 `ret` 结束 | 改成 10 个 `nop` + 末尾 `ret` |
| 2 | `NullReferenceException` | 让假 Loader 静默返回后，`LoadMinecraft_OnFinish` 去读 `DlClientListLoader.Output`，而真 Loader 从未被启动 | 放弃“静默”方案，直接把参数槽位换回真 Loader |
| 3 | 成功 | — | — |

第二条是关键教训：**只让异常消失不够，得看数据从哪来**。

---

## 工具

### 网页版 `index.html`

纯前端，文件不出浏览器：

- 拖入一个或多个 exe
- 内存里按字节特征定位，替换，生成 Blob
- 输出 `原文件名_unlocked.exe`
- 显示进度与 SHA256

### 命令行版 `pcl_download_unlock.py`

```bat
python pcl_download_unlock.py            :: 自动查找并修复
python pcl_download_unlock.py -f x.exe   :: 指定文件
python pcl_download_unlock.py -d         :: 只预演
python pcl_download_unlock.py --restore x.exe
```

自动备份为 `*.bak`，可还原。工具不硬编码文件偏移，而是按
“字符串内容 → #US token → IL 指令三连”定位，换版本号、改文件名都能咬住。

---

## 文件

| 文件 | 说明 |
|---|---|
| `index.html` | 网页版，单文件，双击即用 |
| `pcl_download_unlock.zip` | 命令行版（Python），解压后运行 |
| `PCL_已解锁版.exe` | 预置成品，可直接使用 |

MC 客户端构建器体积 50 MB，超过 Cloudflare Pages 的单文件 25 MiB 上限，
因此放在 Release 附件里：`v1.2 / MCBuilder_v1.2.zip`。

---

## 部署流程

```
  ┌─────────────────────────────┐
  │  本地  D:\Desktop\pcl_unlock │
  │  index.html / *.zip / *.exe │
  └──────────────┬──────────────┘
                 │ git add / commit
                 ▼
  ┌─────────────────────────────┐
  │  git push origin main       │
  └──────────────┬──────────────┘
                 │
        ┌────────┴─────────┐
        ▼                  ▼
┌───────────────┐   ┌──────────────────────┐
│ GitHub 仓库    │   │ Cloudflare Pages     │
│ 源码 + 成品    │   │ 监听 main，自动构建   │
└───────┬───────┘   └──────────┬───────────┘
        │                      │
        │                      ├─ 构建成功 → 部署静态产物
        │                      └─ 构建失败 → 保留上次成功版本
        │                                 │
        ▼                                 ▼
┌───────────────┐   ┌──────────────────────────────────┐
│ GitHub Pages  │   │ pcl-unlock.pages.dev              │
│ 备用入口       │   │ pcl.gber.dpdns.org  (CNAME)       │
└───────────────┘   └──────────────────────────────────┘

  大文件（> 25 MiB）
  ─────────────────
  不进仓库，走 GitHub Release 附件：
  releases/download/v1.2/MCBuilder_v1.2.zip
  页面里的 mc 链接直接指向这个地址。
```

### 步骤

1. 本地改文件，`git add -A && git commit -m "..."`
2. `git push origin main`
3. Cloudflare Pages 检测到 `main` 有新 commit，自动拉取并构建
4. 构建产物部署到 `pcl-unlock.pages.dev`，自定义域名 `pcl.gber.dpdns.org` 走 CNAME 生效
5. GitHub Pages 同时提供 `gangbeng1145.github.io/pcl_unlock/` 作为备用
6. 验证：`curl` 首页看 `const LINKS` 是否为新值

### 注意

- **Cloudflare Pages 单文件上限 25 MiB**，超过的附件必须走 Release，否则构建失败、站点回退到上一次成功版本
- 构建通常 40 秒到 2 分钟；`cf-cache-status: DYNAMIC` 说明不是缓存问题，是构建没完成或失败
- 改了 `index.html` 后以 `pcl.gber.dpdns.org` 为准，`github.io` 只是备用入口

---

## 附：MC 客户端构建器（v1.2）

Release `v1.2` 的附件 `MCBuilder_v1.2.zip` 是另一个独立作品，与 PCL 解锁无关。
体积 50 MB，超过 Cloudflare Pages 的 25 MiB 单文件上限，因此放在 Release 里分发。
**该 Release 仅用于文件下载**，不包含源码。

### 它做什么

一个不依赖启动器的 Minecraft 客户端下载 / 配置 / 打包工具。选好版本和加载器，
它自动下载原版、依赖库、资源文件，调用官方安装器装好 Forge / OptiFine / Fabric / Quilt，
最后打包成一个 zip。把 zip 解压出的文件夹作为 HMCL 的游戏目录，即可直接启动。

### 原理

**1. 版本清单**
优先从 BMCLAPI 拉 `version_manifest.json`，失败回退 `piston-meta.mojang.com`；
内置 920+ 版本的离线清单，断网也能列出目录。

**2. 原版安装**
按 version json 逐项下载 `client jar`、`libraries`、`assets`、`logging`，写入标准目录结构。

**3. 加载器**
- Forge：调用官方 installer 的 `--installClient`，新旧版本通吃
- OptiFine：调用官方 Patcher 生成补丁库
- Fabric / Quilt：读取官方 meta 生成版本 JSON

**4. 下载调度**
自动测速选源（国内优先 BMCLAPI）、线程 4/8/16/24/32、失败换源、
SHA-1 校验、按哈希缓存、断点复用。

**5. Java**
内置 Mojang 运行时下载（8 / 16 / 17 / 21 / 25），装好自动加入 Java 列表。

**6. 打包**
输出 zip 内含 `versions` / `libraries` / `assets` 与 `使用说明.txt`，解压即用。

### 技术栈

Python + PySide6，PyInstaller 打成单文件 exe。
源码结构：`app.py` / `cli.py` / `verify.py` / `core/` / `ui/`。

### 已知限制

- 仅 Windows 10 / 11 64 位
- 不含 NeoForge（网络到 `maven.neoforged.net` 被阻断）
- Forge 1.13+ 与 OptiFine 不能叠加
- 不修改现有 `.minecraft`，所有产物在构建目录下

---

## 声明

仅用于处理自己持有的样本，请勿用于其它用途。
本项目与 Mojang、PCL 官方无关。
