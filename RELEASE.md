# 微信聊天记录导出工具 — 项目文档

> 版本: v1.0
> 最后更新: 2026-06-11
> 微信版本: 4.1.10.29

---

## 项目概述

Windows 微信聊天记录导出工具。双击即用，提取密钥 → 解密数据库 → 导出文字聊天记录。

**核心链路：**
1. `wx_key.dll` 从微信进程内存提取 SQLCipher 密钥
2. `WCDB.dll` 解密 SQLite 数据库
3. Python tkinter GUI 展示会话和消息，支持导出 TXT/JSON

---

## 项目结构

```
wechat_v4_export_research/
├── gui/
│   ├── app_v3.py              ← 主 GUI 程序 (tkinter)
│   └── icon.ico               ← 窗口图标
├── scripts/
│   ├── get_key.js              ← 密钥提取 (Node.js + koffi + wx_key.dll)
│   ├── wcdb_server.js          ← WCDB HTTP 服务 (Node.js + Electron + WCDB.dll)
│   ├── wcdb_server.py          ← WCDB 客户端 (Python)
│   └── node_modules/           ← Node.js 依赖 (koffi, fzstd)
├── build_dist.py               ← 打包脚本 (PyInstaller + 资源复制)
├── dist/WeChatExport/          ← 发布包 (双击 WeChatExport.exe)
├── research/                   ← 研究资料 (参考源码、实验数据、日志)
│   ├── tools/                   ← 参考的开源项目
│   ├── experiments/             ← 实验数据和脚本
│   ├── dailylog/                ← 研究日志
│   └── ...
└── gui/icon.ico                ← 窗口图标
```

---

## 技术方案

### 密钥提取

通过 `wx_key.dll` (MIT 开源项目) 注入 shellcode 到微信进程，Hook `SetDBKey` 函数，拦截 SQLCipher 加密密钥。

- 调用方式: Node.js + koffi (FFI 库) 加载 DLL
- 时机: 微信进程启动时 Hook，捕获启动过程中的 SetDBKey 调用
- 输出: `%USERPROFILE%/Desktop/wx_export/key.txt` (64 位 hex 字符串)

### 数据库解密

利用微信官方的 WCDB 框架 (BSD 开源) 解密 SQLCipher v4 数据库。

- `WCDB.dll` + `wcdb_api.dll` 通过 Electron 进程加载
- 提供 HTTP API: `/sessions`, `/messages/{wxid}`, `/displaynames`
- 消息中 ZSTD 压缩内容自动解压

### 加密参数

```
算法: AES-256-CBC
KDF: PBKDF2-HMAC-SHA512 × 256000 次迭代
页面大小: 4096 bytes
保留字节: 80 bytes/page (IV 16 + HMAC 64)
Key: 64 字符 hex 字符串
```

---

## 构建和运行

```bash
# 构建发布包
python build_dist.py
# 输出: dist/WeChatExport/ (双击 WeChatExport.exe)

# 开发模式
python gui/app_v3.py
```

---

## v1.0 功能

- ✅ 一键提取微信数据库密钥
- ✅ 浏览会话列表（支持搜索过滤）
- ✅ 查看文字聊天记录（按微信时间顺序）
- ✅ 会话昵称和发送者昵称显示
- ✅ 导出 TXT / JSON
- ✅ ZSTD 压缩消息自动解压
- ✅ 可调加载条数 (50~2000)

### 已知限制

- ❌ 仅文字消息，不支持图片/语音/视频/文件
- ❌ 需要管理员权限 (wx_key.dll 注入需求)
- ❌ 需要捆绑 Electron (WCDB.dll 只能在 Electron 环境初始化)
- ❌ 发布包 417MB (Electron 占大头)

---

## 依赖的开源组件

| 组件 | 许可证 | 用途 |
|------|--------|------|
| wx_key.dll | MIT | 微信内存密钥提取 |
| WCDB.dll | BSD 3-Clause | 数据库解密 |
| SDL2.dll | zlib | WCDB 依赖 |
| Electron | MIT | WCDB 运行时 |
| koffi | MIT | Node.js FFI 库 |
| fzstd | MIT | ZSTD 解压 |
| PyInstaller | GPL 2.0 | Python 打包 |

---

## 版本历史

- **v1.0** (2026-06-11): 首次发布。支持密钥提取、文字消息查看和导出。
