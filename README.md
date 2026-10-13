# 夸克网盘 MCP Server（fork）

本仓库是 [MichealJl/quark-nd-mcp](https://github.com/MichealJl/quark-nd-mcp) 的 fork。

上游项目本身的说明、安装包，以及全部工具（列出 / 搜索 / 上传 / 下载 / 分享保存等）请看源仓库，这里不重复；本文只讲本 fork 相对上游改了什么。

## 相对上游的改动

### 修复：上传显示成功，网盘里却没有文件

分片上传的最后一次 `CompleteMultipartUpload` 会被 OSS 以 `203 CallbackFailed` 拒绝，而上层的 `upload_file` 仍然返回 success，于是出现"传完了但文件不存在"。两个原因叠加：

- **SHA1 算成了 MD5**：`quark/client.go` 里写的是 `sha1Hash := md5.New()`，而服务端要的是真正的 SHA1。
- **callback 被重新序列化**：`/file/upload/pre` 返回的 `callback` 对象先被解码成固定字段的结构体、再序列化回去，服务端多下发的字段（如 `callbackVar`）被丢掉。现在该字段保持原始 JSON，逐字节透传。

### 修复：Windows 上默认配置路径退化成相对路径

`config.DefaultConfigPath` 原来用 `os.Getenv("HOME")` 拼路径；Windows 上由非 shell 进程（计划任务等）启动时 `HOME` 常为空，配置就被写到进程的当前目录。改用 `os.UserHomeDir()`。

### 新增：`login` 子命令，用本机浏览器取 Cookie

```bash
./quark-nd-mcp login
```

它打开一个**专属 profile** 的浏览器窗口让你登录，再通过 DevTools 协议把 Cookie（含 HttpOnly 的 `__pus`）取回，**并先真调一次接口校验、通过才写入配置**。

之所以要开浏览器：夸克接口只认 `__pus`，而它是 HttpOnly（页面脚本读不到）；浏览器自身的 Cookie 库在运行期间被独占，新版 Chromium 又启用了 App-Bound 加密，外部进程解不开 —— 只能让浏览器自己把已解密的值交出来。

- 驱动本机**已安装**的 Edge / Chrome（`-browser` 也可直接给可执行文件路径），新增依赖只有 `gorilla/websocket`：不内置浏览器，也不依赖 playwright 之类的自动化运行时
- 用独立 profile：**不打扰你日常使用的浏览器**，也不需要先关闭任何浏览器
- 结束时通过 `Browser.close` 让浏览器正常退出（会刷盘，也不会留下"意外崩溃"的恢复提示），不退才强杀进程树兜底
- 记住这次用的浏览器与 profile（配置旁的 `login.json`），以后不带参数直接沿用；没有记录时会列出本机浏览器让你确认一个，`-yes` 或 stdin 不是控制台时自动取第一个（计划任务不会被卡住）
- 登录用的 profile 内关掉同步与扩展、只保留一个标签页

参数：

| 参数 | 说明 |
| --- | --- |
| `-browser <名字或路径>` | `edge` / `chrome`，或浏览器可执行文件路径；不填则列出本机浏览器让你确认 |
| `-yes` | 跳过确认，直接用探测到的第一个（Edge）；计划任务等无人场景 |
| `-profile-dir <目录>` | 专属浏览器配置目录，默认 `~/.quark-nd-disk/browser-profile-<浏览器>` |
| `-config <路径>` | Cookie 写入位置，默认 `~/.quark-nd-disk/config.json` |
| `-timeout <时长>` | 等待登录的上限，默认 `10m` |
| `-dump-json <路径>` | 额外导出一份 Cookie-Editor 兼容的 JSON |

手工方式：在浏览器里登录夸克网盘 → F12 → Network → 找任意发往 `drive.quark.cn` 的请求 → 复制请求头里的 `Cookie` 值，再 `./quark-nd-mcp config set cookie "<粘贴>"`。

### 新增：配置热更新

客户端改为**每次请求按配置文件修改时间取 Cookie**（`config.Watcher` + `Client.SetCookieSource`），`login` 刷新配置后**正在运行的实例无需重启**就会用上新值；多个实例共用同一份配置，登录一次即可。

## 编译

```bash
go build -o quark-nd-mcp .
```

需要 Go 1.25 及以上。

## 关于本 fork

- 上游：[MichealJl/quark-nd-mcp](https://github.com/MichealJl/quark-nd-mcp)
- 上述改动位于分支 [`fix/upload-and-cookie-login`](https://github.com/rea1s/quark-nd-mcp/tree/fix/upload-and-cookie-login)，并已向上游提交 [PR #2](https://github.com/MichealJl/quark-nd-mcp/pull/2)
- 许可证：MIT（与上游一致）
