# B站 MV 小窗 · `bili-mv-window`

Folia（[chthollyphile/folia-major](https://github.com/chthollyphile/folia-major)）的 **Folium 1** 模组。

换歌的时候自动拿歌名去哔哩哔哩搜这首歌的 MV，在播放页角落开一个小窗放着。**小窗里没有声音** ——
不是靠 `muted` 糊的，是这条流本身就没有音轨。

粘一段 B 站 Cookie 就能解锁 1080P / 1080P60 / 4K。

---

## 为什么真的是「纯视频流」

B 站的 DASH 把画面和声音存成**两个不同的文件**：

```
dash.video[]  →  upos-*.bilivideo.com/....m4s   只有画面
dash.audio[]  →  upos-*.bilivideo.com/....m4s   只有声音
```

这个模组只取 `dash.video[]` 那一条，播放器手里从头到尾就没有音频轨可以放。

实测（`夜曲` / 周杰伦，`BV1Ek4y1r7Rg`，走本地代理取前 512 KB 拆 box）：

```
moov: 有，893 字节
trak 数量: 1
hdlr handler_type: [ 'vide' ]
含 soun 轨: false
含 vide 轨: true
Content-Type: video/mp4
```

一个 `trak`，handler 是 `vide`，没有 `soun`。这条 mp4 里根本没有音频轨。

`<video muted>` + `volume = 0` 只是第二道保险 —— 只在最后的 `durl` 兜底源（那是有音轨的整段 mp4）上才真正起作用。

---

## 安装

两种都行：

**A. 直接放目录**

```
%APPDATA%\Folia\mods\bili-mv-window\
    mod.json
    index.cjs
    client.mjs
```

（模组面板右上角「打开模组目录」就是这个 `userData/mods`。）

**B. 拖 zip**

把 `bili-mv-window-1.0.0.zip` 直接拖进模组面板。

---

## 启用

Folium 的模组是可信代码，默认全关，要开两道：

1. **设置 → 模组 → 模组系统** 总开关打开。关着的时候加载器根本不扫目录。
2. 模组面板里找到「B站 MV 小窗」，点启用，在原生确认窗口里确认一次。

确认窗口会列出它要的权限：

| 权限 | 用来干什么 |
| --- | --- |
| `stageLayers` | 在播放页上画那个小窗 / 背景层 / 扫码面板 |
| `net.embed` | 只在「嵌入播放器兜底」打开时才用（见下） |
| `filesystem.data` | 记住位置、背景模式、每个视频的偏移、手选的视频 |
| `visualizer.mode` | 歌词开关：切到「不显示歌词」模式再切回来 |

`embedOrigins` 只声明了 `https://player.bilibili.com`。

> 注意：启用确认绑的是模组目录的 sha256。之后**改任何一个文件都会让授权失效并自动禁用**，需要重新确认。
> 唯一的例外是放在仓库 `mods/` 目录下的未打包开发版 —— 改完直接重载即可。

---

## 接 B 站登录（解锁 1080P / 4K）

两种方式，扫码最省事。

### 方式 A：扫码登录（推荐）

1. 命令面板里跑 **「B站账号：扫码登录」**。
2. 会自动切到播放页，弹出一张二维码（在本地画的，不走任何第三方图床）。
3. 手机 B 站 App 扫码 → 手机上点确认。
4. 面板显示「登录成功」后自动关掉，并按新账号重新取一次流。

二维码 3 分钟内有效，过期了点「换一张」。拿到的是 `SESSDATA` + `bili_jct` + `DedeUserID` +
`DedeUserID__ckMd5` + `sid`，加上当前设备的 `buvid3`，一起写进凭证库。

> 二维码在 `app.overlay` 图层上渲染，而 `stageLayers` 只在真实播放页生效，所以命令会先
> `folium.ui.navigate('player')` 把你带到播放页。在首页直接点命令也会被带过去。

### 方式 B：手动粘贴 Cookie

1. 浏览器登录 [bilibili.com](https://www.bilibili.com)。
2. F12 → **Network** → 随便点一个发往 `api.bilibili.com` 的请求 → **Request Headers** → 复制 `Cookie:` 后面**整段**。
3. 模组面板 → 本模组 → **B 站 Cookie** 那一栏，整段粘进去。

只要里面有 `SESSDATA` 就行。`bili_jct`、`DedeUserID`、`buvid3` 有就一起带上，写回请求时是原样整段发的。

> 也可以从 **Application → Cookies → https://www.bilibili.com** 手动凑，但整段复制省事且不会漏字段。

### 粘完先验一下

命令面板里跑 **「B站账号：检查登录状态」**，会回你：

- 已登录 + 用户名 +「大会员：1080P60 / 4K 可用」或「普通账号：最高 1080P」
- 或者「Cookie 读到了，但 B 站说没登录 —— SESSDATA 大概过期了」

然后是 **「B站账号：刷新会话」**，它会丢掉旧会话缓存并按新 cookie 重新取一次流，不用重启。

### 这套东西存在哪

走的是 Folium 的 `secret` 类型设置字段（Folium 1.7+）：

- 值由**主进程**加密保管，Windows 用 DPAPI、macOS 用 Keychain、Linux 用 libsecret/kwallet；
- **不进**设置值表、**不进**视觉配置导入导出、**不进**导出窗口；
- **不写**任何日志（`resolveMv` 的返回值里只有 `loggedIn` / `uname` / `vip` 三个布尔与字符串，没有 cookie）；
- Linux 上如果没有可用的凭证库，宿主只在本次运行的内存里留着，设置页也会写明。

模组本身读的是 `api.secrets.get('cookie')`（main 入口）—— 它只能读自己声明过的 key，
10 秒缓存一次，值一变就作废旧会话、重新算 wbi 密钥和登录态。

### 画质到底怎么定

`quality` 是**上限**，不是承诺：

| 设置 | qn | 未登录实际 | 普通账号 | 大会员 |
| --- | --- | --- | --- | --- |
| 480P | 32 | 480P | 480P | 480P |
| 720P | 64 | 360P | 720P | 720P |
| 1080P | 80 | 360P | 1080P | 1080P |
| 1080P60 | 116 | 360P | 1080P | 1080P60 |
| 4K | 120 | 360P | 1080P | 4K |

实际拿到的那一档会写在小窗的标题条上，比如 `【4K修复】… · 作者 1080P AVC`。

**编码器优先级是 AVC > AV1 > HEVC。** 这是 4K 能不能放出来的关键：

- B 站 4K **没有 AVC**，只有 HEVC 和 AV1；
- Chromium 自带 AV1 解码器，HEVC 要看系统有没有装解码扩展；
- 所以选 4K 时会自动落到 AV1，而不是拿一条放不出来的 HEVC 让你盯黑屏。

同一编码器内取不超过上限的最高档；上限太高但账号不够时，自然落到账号允许的最高档。

---

## 用起来

不用点任何东西。换歌 → 小窗自己出现。

**位置靠拖。** 按住小窗往哪拖就到哪，松手就记住（`folium.storage` 里存的是相对舞台的比例，
换个窗口尺寸也不会跑偏）。窗口不会被拖出播放页边界。**单击**小窗（不拖）会打开候选列表。

模组面板 → 该模组展开 → **设置区**：

| 设置 | 默认 | 说明 |
| --- | --- | --- |
| 启用小窗 | 开 | 关掉就完全不出现 |
| **B 站 Cookie** | 空 | 见上。只在 Folium ≥ 1.7 的宿主上出现 |
| 小窗大小 | 中 | 三挡：视口宽的 22% / 32% / 44% |
| 不透明度 | 92 | 20–100 |
| 跟随播放 / 暂停 | 开 | 歌曲停，视频就停；歌曲走，视频接着走 |
| 音画偏移 | 0 | ±5000 ms 的**基准值**。每个视频各自记的实际偏移优先，见下 |
| 清晰度上限 | 1080P | 见上面的表 |
| 歌手匹配忽略大小写 | 开 | 见下 |
| 把视频存到本地 | 不保存 | 关 / 只存我手选的 / 自动存匹配度最高的 |
| 优先用本地文件 | 开 | 关掉就每次都联网取；手动指定的 BV 仍然走本地 |
| 本地占用上限（MB） | 2048 | 超了按最久没看的先删 |
| 存放位置 | 模组缓存目录 | 模组缓存 / 系统视频 / 系统下载 / 系统音乐 / 自定义 |
| 自定义路径 | 空 | 绝对路径，目录不存在会自动建 |
| 画面填充 | 裁切铺满 | 或「完整显示（黑边）」 |
| 强制静音 | 开 | 每 250 ms 把 `muted`/`volume` 拉回静音 |
| 隐藏标题条 | 关 | 平时不显示歌名条，悬停才出 |
| 允许嵌入播放器兜底 | **关** | 见下 |

命令面板里九条：

- **BV 小窗：显示 / 隐藏**
- **BV 小窗：换一个视频** —— 按匹配度排 5 个候选，点一个就换过去
- **BV 小窗：重置位置和大小**
- **BV 小窗：忘掉这首歌的手选** —— 清掉手选记录，下次回到自动挑
- **BV 小窗：重新搜索** —— 带一个搜索词参数，留空就用当前歌
- **BV 小窗：换一个视频源**
- **B站账号：扫码登录**
- **B站账号：检查登录状态**
- **B站账号：刷新会话**

---

## 三个跟播放器对齐的细节

### 换歌时视频从当前进度开始，不从 0 开始

以前的毛病：搜 MV 要 1.5 秒、加载流要 2 秒，这 3.5 秒里歌曲往前走了，视频却从 0 秒开始 ——
越看越不对。现在每次换源、每次 `loadeddata` / `canplay` 都对齐一次：

```js
const target = clamp(playback.position - offsetSec, 0, video.duration - 0.5);
if (Math.abs(video.currentTime - target) > 1.5) video.currentTime = target;
```

位置来自 `folium.playback.getState().position`（读不需要权限）。偏差 1.5 秒内就不折腾，
免得反复 seek 把播放打断。用户拖歌曲进度条时收到 `playback.seeked` 也会重新对齐。

#### 画面上的偏移微调条

小窗底部浮着一条胶囊控件，**平时完全透明，鼠标移到那块才浮出来**，不占画面：

```
      ┌──────────────────┐
      │  −   +0.3s   ＋  │
      └──────────────────┘
```

- **− / ＋**：每次 100 ms。**按住不放连续调**（140 ms 一步），不用点十几次。
- **中间的数字**：点一下归零。
- **偏移按视频分别记**：`avOffsets` 存成 `{ [bvid]: ms }`，A 视频调成 −1.3s，换到 B 视频就是 B 自己的值，
  再回到 A 还是 −1.3s。设置面板的「音画偏移」只是**没有单独记录时**的基准值。
- 在设置面板里改「音画偏移」，只清掉**当前这个视频**的单独记录让它回落到新基准，别的视频的记录不动。
- 控件上的 `pointerdown` 会 `stopPropagation()`，所以点它不会把小窗拖走。

### 歌曲暂停，视频就停

```js
folium.events.on('playback.stateChanged', ({ state }) => syncPlaybackState(state));
```

`playing` → 放；`paused` / `stopped` → 停。原来 `<video loop>` 自己一路播，歌停了画面还在动。

用事件而不是轮询，也不调 `folium.playback.pause()` —— 从没声明过 `playback.control`，
模组不碰用户的播放器。

### 选错了可以自己换，而且换过就记住

同一个歌名可能撞上翻唱、扩曲版、越南版。单击小窗，或跑「BV 小窗：换一个视频」，列出
**按匹配度排的 5 个**候选：

```
1. 【4K修复】周杰伦 - 夜曲 MV 2160P修复版
   zyl2012_音乐无限 · 3:52 · 歌名 ✓ +62 · 标题 ✓ +18 · 匹配 136
2. 【4K Hi-Res】夜曲-周杰伦 失去你 还有什么事好关心
   你去巴黎我在北京 · 3:47 · 歌名 ✓ +62 · 标题 ✓ +18 · 匹配 136
3. 《夜曲》周杰伦丨百万级录音棚试听丨【Hi-Res无损】
   十月Free-music · 3:49 · 歌名 ✓ +62 · 歌手 ✗ +0 · 匹配 112
```

**歌手那一项看的是 UP 主名字。** `UP主 ✓ +34` 表示上传者名字里带歌手名（官方号 / 本人号），
是最强的信号；退一档是 `标题 ✓ +18`，视频标题里带歌手名；都没有就是 `歌手 ✗ +0`。

> 一开始试过「只按 UP 主名字判断」，结果《夜曲》搜出来前 5 个全是搬运号
> （`zyl2012_音乐无限`、`你去巴黎我在北京`……），这一维全是 0 分，反而没有区分度。
> 所以做成两档：UP 主优先，标题兜底。

**选了 A 就是 A。** 手选之后会记进 `pinnedMv`（键是「歌名⋈歌手」），下次再放到这首歌，
直接按那个 BV 取流，不再重新搜。想回到自动挑，跑「BV 小窗：忘掉这首歌的手选」。

候选列表只在**搜索**时刷新 —— 手动挑完再点开还是原来五个，随时能换回去。

### 候选面板最下面那格：自己搜

固定五个候选之后还有一行输入框：

```
┌────────────────────────────────────────┐
│ 5. [厦门圣地巡礼]一道厦门，一抹憨色      │
│    狼王小姐姐zrx · 2:19 · 歌名 ✓ +62 … │
├────────────────────────────────────────┤
│ [ 搜点别的…                    ] [搜索] │
│ 当前：一道厦门 · Hanser                 │
└────────────────────────────────────────┘
```

- 输入任意关键词（换语言、换拼写、找翻唱、找 Live 版都行），回车或点「搜索」。
- 走的是 `searchCandidates` —— **只出候选、不取流**，所以点搜索不会打断正在放的东西。
- 搜完这一步列表就换成新结果，副标题显示「一道厦门」· 1 个。
- 输入框下面那行固定显示**当前歌曲的歌名和歌手**，提醒你在给哪首找 MV。

> 回车做了 `stopPropagation()`，不会漏给宿主的全局快捷键。

---

## 把 MV 当成播放页背景

播放器右下角那个控制面板（封面 / 歌词 / 控制 / 播放列表 / 音质）多了一个 **「视频」** 标签：

```
┌─ 视频 ────────────────────────┐
│ 作为播放页背景            [ ] │
│ 显示歌词                  [✓] │
│                              │
│ 【4K修复】周杰伦 - 夜曲 MV     │
│ BV1Ek4y1r7Rg                 │
│                              │
│ [ 换一个视频 ]  [ 重新搜索 ]   │
│ ───────────────────────────  │
│ 画面模糊  [=====|=======] 12px │
│ ───────────────────────────  │
│ 音画偏移        −  −0.1s  ＋  │
└──────────────────────────────┘
```

**画面模糊**：0–40px 高斯模糊，拖动即时生效。模糊会让画面边缘透出底色，所以会自动
`scale(1.04)` 把毛边推出画外。背景模式下压一下模糊，歌词会好读很多。

**显示歌词**：关掉后整个播放页不显示歌词。

> 这个开关的做法有点绕。宿主没给模组「隐藏歌词」的接口，`styles` 注册表又只认
> `[data-folium-part="..."]` 的白名单（歌词没有公开 part）。所以模组自己注册了一个
> **什么都不画的歌词模式** `mod:bili-mv-window:lyrics-off`，开关就是 `folium.visualizer.setMode()`
> 切过去；打开时切回你原来的模式（切之前用 `getMode()` 记下来）。
> 需要 `visualizer.mode` 权限，Folium 1.8+ 才有。

打开开关之后：

- MV 从角落的小窗搬出来，铺满整个播放页（`player.stage.back`，背景之上、歌词之下），
  `object-fit` 跟着设置里的「画面填充」走。
- **小窗消失**，位置和大小都不再占地方。歌词照常在上面滚。
- 偏移和视频选择都在这个面板里，不用再回小窗上点。
- 关掉开关，同一个 `<video>` 元素搬回小窗原位置，位置和大小还是你上次拖的。

> 面板里的开关存的是 `folium.storage` 的 `bgMode`，不是宿主设置。
> 原因很实际：`settingsSections` 只给了 `get()` / `onChange()`，**没有 `set()`**，
> client 端写不了设置值。所以这个开关做成了运行时状态，重启后还在。

---

每次换歌，client 把 `{ 标题, 歌手, 时长 }` 丢给 main 入口，main 返回一串候选源，
client 按顺序试，`<video>` 的 `error` 一响就换下一个：

```
1. dash-proxy    本地 127.0.0.1 代理转发 DASH 视频轨，补 Referer    ← 纯视频，主线路
2. dash-direct   <video> 直接吃 B 站 CDN 的 m4s 直链               ← 纯视频
3. durl-proxy    重新要一份 durl 并代理转发                         ← 含音轨，强制静音
4. durl-direct   durl 直链                                          ← 含音轨，强制静音
5. embed         B 站官方 iframe 播放器（默认关闭，跨域没法静音）
```

第 3、4 步不是白写的：B 站 CDN 有一堆带端口的 PCDN 节点（`*.mcdn.bilivideo.cn:8082`），
防盗链和 HTTP 协议都可能让 1、2 失败，那时候宁可要一份带音轨但被死死静音的 mp4，也不要黑屏。
触发方式是 client 发现所有 dash 源都失败后，回头调一次 `resolveMv({ durlOnly: true })`。

**为什么要本地代理**：B 站 CDN 校验 `Referer`，而渲染进程发请求改不了 `Referer`。
main 入口是 Node，起一个只读的 `127.0.0.1` 服务器，把 `Referer: https://www.bilibili.com/`
和 UA 补上，`Range` 原样透传（`206 Partial Content` 已验证）。

**代理的两道锁：**

- **一切都要 token。** 每个上游地址和每个本地文件都发一次性随机 token（16 字节，2 小时过期），
  URI 形如 `/s/<token>`、`/f/<token>`，**里面不含任何文件名**。别的网页就算扫到端口，
  也只能拿到 404 —— 探不出你缓存了哪些歌。
- **校验 `Host`。** 所有请求的 `Host` 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`，防 DNS rebinding。
- 模组停用时代理、上游 token、本地 token 一起销毁。

---

## 把视频存到本地

设置里「把视频存到本地」三档：

| 值 | 什么时候存 |
| --- | --- |
| `不保存`（默认） | 什么都不留，每次现取 |
| `只存我手选的` | 你在候选列表里点了哪个，就把哪个落到本地 |
| `自动存匹配度最高的` | 每次搜索出来的第一名，后台默默存一份 |

存的地方由「存放位置」决定：

| 选项 | 实际路径 |
| --- | --- |
| 模组缓存目录（默认） | `mods-cache/bili-mv-window/mv/` |
| 系统「视频」文件夹 | `<视频>/Folia MV/` |
| 系统「下载」文件夹 | `<下载>/Folia MV/` |
| 系统「音乐」文件夹 | `<音乐>/Folia MV/` |
| 自定义路径 | 你填的那个绝对路径（不存在会自动建） |

一个视频两个文件，**文件名就是「歌名 - 歌手」**：

```
夜曲 - 周杰伦.m4s    视频本体（DASH 视频轨，本来就没音轨）
夜曲 - 周杰伦.json   标题 / UP 主 / 时长 / 清晰度，回放时不用再问 B 站
```

文件名里的 `\ / : * ? " < > |` 会换成 `_`，过长截到 90 字。

**开了保存之后会优先用本地。** 换歌时 `resolveMv` 先做两件事，都不联网：

1. 指定了某个 BV，本地正好有它 → 直接用；
2. 有歌名歌手，本地有同名文件 → 直接用。

命中时返回的 `sources` 只有 `['local']`，`fromCache: true`。只有 `cache` 设成 `off` 时才跳过第 2 条。

所以同一首歌第二次播放是秒开的，而且 B 站那边的视频被删了、CDN 抽风了都不影响。

存过之后再放到这首歌（或手选到同一个 BV），`resolveMv` **一个网络请求都不发**：

```js
if (bvidAsked && hasCachedVideo(bvidAsked)) {
  await ensureProxyServer();
  return buildLocalResult(bvidAsked);   // sources: ['local']
}
```

`local` 源指向 `http://127.0.0.1:端口/f/<bvid>`，由 main 从磁盘读、按 `Range` 切片吐出来。
不走 `file://`——那玩意在 Electron 里经常被 `webSecurity` 拦。

**播放器面板「视频」标签最下面**有当前占用和「清空缓存」按钮：

```
本地 3 个 · 412 MB              [ 清空缓存 ]
```

**上限之外自动清理**：每次存完按文件的 mtime 从旧到新删，直到低于「本地占用上限」。
默认 2 GB。

> 只存视频轨这件事有额外好处：落盘的文件拆开看也没有音频流，和「纯视频流」是同一件事。

---

## 歌手匹配的大小写

「歌手匹配忽略大小写」开关（默认开）：

- **开**：歌手名和 UP 主名都走 `normalize()` —— 小写化、去标点空白，再比子串。
  「周杰伦」能匹配「ZHOUJIELUN」「周 杰 伦」「Jay Chou 周杰伦」。
- **关**：拿原文逐字比 `uploaderRaw.includes(artist)`，只认一模一样的写法。

关掉之后，`ZhouJielun` 这类大小写不同的官方号就匹配不上了 —— 除非你输入的歌手名和它完全一致。
两种模式都能搜，区别只在 `artistScore`（UP 主命中 +34 / 标题命中 +18）。

---

## 已知限制

- **B 站 CDN 偶发抽风。** 实测中遇到过一次：同一个视频的 PCDN 节点返回的片段里没有 `moov`（init segment），
  这种片段 `<video>` 放不出来。这就是降级链存在的理由 —— 换源、换节点、最后落到 durl。自测脚本遇到这种情况会
  明确报「判断不了，重跑一次」，而不是假装通过。
- **4K 只能拿到 AV1。** B 站 4K 没有 AVC 轨。理论上如果系统 Chromium 没编 AV1 解码器，4K 会放不出来，
  这时会自动落到 durl（1080P，带音轨但静音）。主流 Electron 构建都带 AV1。
- **嵌入兜底可能有声音。** `player.bilibili.com` 是跨域 iframe，宿主页面动不了它里面的 `audio`。
  所以这个选项**默认关闭**，打开时也会明确 toast 提示。
- **能不能嵌进你那一版 Folia 的媒体策略不确定。** 小窗是 `<video>` 直连 `http://127.0.0.1:端口`，
  如果宿主给渲染进程设了严格的 `CSP media-src`，这条会被拦。真遇到就走 2、4、5。
- **只在真实播放页出现。** `stageLayers` 在设置预览、OBS、导出窗口里不渲染，这是宿主的规定。
- **拖动会把小窗上的点击吃掉。** 小窗定位是「只看不操作」，整块都用来拖拽和打开候选列表，
  里面没有进度条、没有音量按钮 —— 反正它本来就没有声音。要操作 MV 本身，去 `bilibili.com` 搜那个 BV 号。
- **Cookie 会过期。** B 站的 `SESSDATA` 有有效期，失效后命令里跑一次「检查登录状态」就能看出来，重新粘一段即可。

---

## 自测

`tools/bili-mv-window/` 下有三个脚本，不需要启动 Folia，直接跑 main 入口：

```powershell
$node = "C:\Users\32165\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe"
$py   = "C:\Users\32165\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\python\python.exe"

& $node tools\bili-mv-window\smoke.cjs          # 搜一首歌，打印选中的 MV、候选列表，探活本地代理
& $node tools\bili-mv-window\test-client.mjs    # DOM stub 里真跑一遍 activate()：条目、拖拽、音画同步
& $node tools\bili-mv-window\test-bvid.cjs      # 手选 BV 的路径 + 非法 BV 拦截 + 候选字段完整性
& $node tools\bili-mv-window\verify-tracks.cjs  # 拆 m4s 的 box，证明流里没有音轨
& $node tools\bili-mv-window\test-login.cjs     # 用 mock 凭证库跑登录态的四条分支
& $node tools\bili-mv-window\test-qr-login.cjs  # 真的去 B 站要一张二维码，再轮询一次

# 二维码能不能被真扫码器读出来：生成 → 画图 → zbar 解码 → 比对原文
& $node tools\bili-mv-window\qr-selftest.mjs    # 用三个固定字符串，验证编码器本身
& $py   tools\bili-mv-window\qr-verify.py       # 解码上面生成的矩阵
& $node tools\bili-mv-window\qr-live.mjs        # 用 B 站真实返回的 URL 生成矩阵
& $py   tools\bili-mv-window\qr-verify.py

# 掩码选择、逐版本对照（这三个用 OpenCV，是为了复现下面那个结论）
& $py   tools\bili-mv-window\qr-compare.py
& $py   tools\bili-mv-window\qr-mask.py
& $node tools\bili-mv-window\qr-versions.mjs ; & $py tools\bili-mv-window\qr-versions.py
```

`qr-verify.py` 需要 `pyzbar`、`Pillow`、`numpy`；对照脚本另外要 `opencv-python-headless` 和 `qrcode`。

**为什么用 zbar 而不是 OpenCV。** 一开始用的就是 `cv2.QRCodeDetector`，真实的 49×49 码它
`detectAndDecode` 直接返回空串 —— 但同一个字符串交给 Python `qrcode` 库生成，它就能解。

所以要区分「我生成错了」和「解码器弱」，做了三步：

1. 把 JS 的矩阵和 Python `qrcode` 库在 **8 个掩码下逐个逐点比对** → **每一个版本都能完全对上**
   （v2 对 mask 3、v4 对 mask 2、v8 对 mask 2…），说明 JS 产出的是标准合法 QR，
   只是掩码选法和 Python 不同（Penalty 评分实现的细节差异，两者都合法）。
2. 换 zbar（工业级，和手机扫码器一个量级）解码 → **全部通过**。
3. 把 v2–v10 九个版本全跑一遍 zbar → 全部解回原文。

结论：生成没问题，OpenCV 5.0 的 QR 实现对部分掩码确实不稳。扫描端以 zbar 为准。

实测输出：

```
$ node qr-selftest.mjs ; python qr-verify.py
[OK ] 33x33  len=59
[OK ] 41x41  len=97
[OK ] 33x33  len=44
all cases decoded back to the original text

$ node qr-live.mjs ; python qr-verify.py
真实二维码 URL (131 字符) → 49x49 矩阵
[OK ] 49x49  len=131
```

```
$ python qr-mask.py
v2  与 Python 的 mask=3 逐点相同 —— JS 生成正确
v4  与 Python 的 mask=2 逐点相同 —— JS 生成正确
v8  与 Python 的 mask=2 逐点相同 —— JS 生成正确
…（v2..v10 全部命中）
```

`test-qr-login.cjs` 覆盖：

```
1. 申请二维码        → https URL + 32 位 qrcode_key
2. 轮询（没扫码）    → state:waiting，且不带 cookie 字段
3. 轮询一个瞎编的 key → state:expired，不崩
```

`test-login.cjs` 覆盖：

```
1. 没有 cookie            → ok, loggedIn:false, hasCookie:false
2. cookie 里带 buvid3     → 不被 spi 覆盖
3. 未登录取流 480P        → quality { id:32, height:480, codec:'AVC' }
4. 过期 SESSDATA          → hasCookie:true 但 loggedIn:false
```

第 3 条同时证明了「上限」语义：要 480P 就是 480P，未登录要不来更高。

`smoke.cjs` 输出示例：

```
耗时 2382 ms
{ "ok": true, "bvid": "BV1Ek4y1r7Rg",
  "title": "【4K修复】周杰伦 - 夜曲 MV 2160P修复版 …",
  "sources": [ { "kind": "dash-proxy", "videoOnly": true }, { "kind": "dash-direct", "videoOnly": true } ] }
代理探测: 206 video/mp4 2048 bytes, head = 000000246674797069736f35
```

`ftypiso5` = 合法的 fragmented MP4，`<video>` 吃得下。

---

## 挑 MV 的规则

`index.cjs` 里的 `scoreCandidate()`，权重是可读的常量：

- 归一化后标题包含完整歌名 **+62**，只匹配开头 6 个字 **+16**，完全不沾边 **−26**
- **UP 主名字**含歌手名 **+34**；否则标题含歌手名 **+18**
- 时长落在 45s–15min **+12**；超过 30min 或短于 25s **−30**
- 播放量 `log10 × 6`，最多 **+30**
- 标题含 `MV` / `官方` / `Official` / `1080` / `4K` / `中字` **+14**
- 标题含 `翻唱` / `cover` / `伴奏` / `钢琴` / `演奏` / `教学` / `直播` / `混剪` / `鬼畜` **−55**
- 和这首歌时长差 <10s **+22**，<35s **+9**，>240s **−14**

`夜曲` 的实测结果（174 分胜出）：

| 分 | BVID | 标题 |
| --- | --- | --- |
| 174 | BV1Ek4y1r7Rg | 【4K修复】周杰伦 - 夜曲 MV 2160P修复版 |
| 174 | BV1eN41147pj | 【4K Hi-Res】夜曲-周杰伦 失去你 还有什么事好关心 |
| 160 | BV1mdtB64EVv | 《夜曲》周杰伦丨百万级录音棚试听丨【Hi-Res无损】 |
| 153 | BV1fW411e7Qw | 周杰伦 夜曲 越南版 |
| 132 | BV16r4y1U7YP | 夜曲一响，上台领奖！周杰伦《夜曲》音高测量 |

同分时取搜索结果里靠前的那个（`totalrank` 排序）。

`晴天` 的实测：`BV1d4411N7zD`（【4K修复】周杰伦 - 晴天MV 2160P修复版），未登录拿到 480P AVC。

---

## 文件

```
mods/bili-mv-window/
  mod.json             清单：id / 权限 / embedOrigins / folia 版本范围
  index.cjs            main 入口（Node）：wbi 签名、cookie/secret、扫码登录、搜索、打分、取流、编码器挑选、Range 代理
  client.mjs           client 入口（渲染）：设置区、stageLayer 小窗、扫码模态、命令、静音守护
  vendor/qrcode.mjs    kazuhikoarase/qrcode-generator 1.4.4（MIT），只把末尾 UMD 换成 export default
```

`vendor/qrcode.mjs` 是唯一的第三方代码，许可证在文件头的注释里。之所以内置而不是引 CDN：
`folium.ui.embed` 只能嵌 https iframe，`<img>` 又要过 CSP，只有本地画 canvas 是稳的。
而且这个文件只有 57 KB，不联网也能扫。

只用了 `node:http` / `node:https` / `node:crypto`，没有第三方依赖 —— Folium 的 client 入口
本来也不能 import 裸模块名。
