# 提交到 folium-compound 的材料

这份文件不进模组包，只是给你提交 issue 时照抄用的。

仓库：<https://github.com/chthollyphile/folium-compound>
表单：<https://github.com/chthollyphile/folium-compound/issues/new?template=mod-submission.yml>

---

## 一、先把这个仓库推上去

表单要求**公开源码仓库 + tag**。这个目录已经初始化好 git、打完 `v1.9.0` 的 tag，只差一个 remote：

```powershell
cd bili-mv-window-repo

# 在 GitHub 上新建一个空仓库（不要勾 README / .gitignore / license），然后：
git remote add origin https://github.com/<你的用户名>/bili-mv-window.git
git push -u origin main
git push origin v1.9.0
```

推完确认这两件事能打开：

- `https://github.com/<你的用户名>/bili-mv-window`
- `https://github.com/<你的用户名>/bili-mv-window/tree/v1.9.0/bili-mv-window`

> 仓库里以 `.` 开头的目录和 `docs/` 不会被收进模组包 —— CI 只取 `path` 指向的那个目录。

---

## 二、开 issue

**标题**（「[模组提交]」后面写模组名）：

```
[模组提交] B站 MV 小窗
```

**表单逐项**：

| 字段 | 填什么 |
| --- | --- |
| 模组 id | `bili-mv-window` |
| 源码仓库 | `https://github.com/<你的用户名>/bili-mv-window` |
| 版本 | `v1.9.0` |
| 模组目录 | `bili-mv-window` |
| 发布渠道 | 正式发布 / Release |
| 许可证 | `MIT` |
| 运行截图 | 把 `docs/window.jpg` **拖进截图框**（外部链接不算） |

**模组说明**（可直接粘贴）：

```
换歌时自动去哔哩哔哩搜索当前歌曲的 MV，用一个可拖动的小窗在播放页上播放，也可以铺满整个播放页当背景。

画面里没有声音：只取 B 站 DASH 分离出来的纯视频轨（dash.video[]），播放器手里没有音频流可放，
不是靠 muted 静音做出来的。适合「本地在放这首歌，想看它的 MV 但不想两路声音打架」的场景。

主要功能：
- 自动搜 MV，按歌名 / UP 主名 / 时长 / 播放量打分挑一个；错了可以从匹配度前 5 个里自己挑，也能自由搜关键词
- 小窗可拖动、三挡大小；或铺成播放页背景
- 跟随播放器的播放 / 暂停 / 切歌，换歌时把视频对齐到当前进度（不是从头播）
- 音画偏移可调 ±5000ms，每个视频各记各的
- 画面高斯模糊 0–40px、歌词一键隐藏
- 扫码登录 B 站后可取 1080P / 1080P60 / 4K
- 可选把视频存到本地（按「歌名 - 歌手」命名），下次秒开
```

**权限说明**（逐条）：

```
stageLayers —— 在播放页上画小窗图层（player.stage.front）、背景图层（player.stage.back）、
候选选择面板和扫码面板（app.overlay）。这是模组的主要输出方式。

net.embed —— 只在用户手动打开「嵌入播放器兜底」设置时使用，用于把 player.bilibili.com 的官方
iframe 播放器嵌进播放页，作为所有直链都失败时的最后手段。默认关闭。

filesystem.data —— 用 folium.storage 记住用户拖出来的小窗位置、背景模式、每个视频各自的音画偏移、
手动挑选过的视频；本地视频缓存目录也从 api.storage.paths.cache 取。

visualizer.mode —— 歌词开关需要切换歌词动画模式：模组注册了一个什么都不画的模式
mod:bili-mv-window:lyrics-off，关闭歌词时切过去，打开时切回用户原来的模式（切换前用 getMode() 记录）。

embedOrigins：https://player.bilibili.com —— 只有上面那个兜底 iframe 用到。

experimental：无。
```

**三个确认框**都要勾：

- [x] 我是这个模组的作者，或有权代表作者提交
- [x] 模组不收集或上传用户数据，也不下载并执行远程代码（除非这是它明确声明的功能）
- [x] 我了解每个新版本都要经过审查才会签名

> 说明一下第 2 条：模组会根据用户输入的歌名去 B 站公开搜索接口取列表、按 B 站登录 cookie 取播放地址，
> 这些都是它声明的功能本身，不上传任何用户数据；没有下载并执行远程代码的行为
> （`vendor/qrcode.mjs` 是本仓库内的静态文件，不联网获取）。

---

## 三、CI 会做什么

CI 取回 `https://github.com/<用户名>/bili-mv-window` 在 `v1.9.0` 的那个 commit，检查 `bili-mv-window/`
目录的格式（清单字段、声明的入口文件是否存在、权限名是否合法等），然后在 issue 里回复一份**将要签名的文件清单**：

```
mod.json
index.cjs
client.mjs
preview.jpg
vendor/qrcode.mjs
```

看到清单和你的预期一致，就等维护者审查。审查通过后由维护者签名、复制进 folium-compound、发布到模组市场。

格式检查不通过、或 14 天内没有回应，issue 会被自动关闭。以后发新版开「模组更新」issue 即可，不要重开提交 issue。

---

## 四、本目录结构

```
bili-mv-window-repo/
  README.md                 仓库首页（功能说明 + 截图）
  LICENSE                   MIT
  SUBMIT.md                 本文件
  docs/                     README 用的截图，不会进模组包
    settings.jpg
    commands.jpg
    background.jpg
    window.jpg
  bili-mv-window/           ← 提交时 path 填这个
    mod.json
    index.cjs
    client.mjs
    preview.jpg             mod.json 里 preview 指向它，16:9 1280x720
    vendor/qrcode.mjs
```
