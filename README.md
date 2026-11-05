# BOSS Anti Debug

针对 **BOSS 直聘（www.zhipin.com）网页端**的定向反调试兼容脚本。

当打开浏览器开发者工具（DevTools / F12）后，BOSS 页面可能触发特定的反调试检测，表现为页面刷新、跳转或聊天页面无法正常停留。本脚本针对目前分析确认的检测入口做兼容处理，尽量减少这类行为对正常调试和使用的影响。

> 本项目不是通用反调试工具，也不保证兼容 BOSS 未来所有版本。脚本只处理代码中能够识别的特定检测逻辑。

## 功能

当前版本为 **v0.0.1**，主要包含以下三类处理：

### 1. Webpack 模块级反调试处理

在页面初始化阶段监听以下 Webpack chunk 队列：

- `webpackJsonp`
- `webpackChunkfe_web_sign`
- `webpackChunkgeek`

脚本会检查新加载和已存在的模块工厂函数，只对符合已确认特征的反调试模块进行替换，不会无差别修改所有模块。

替代实现会保留目标模块的初始化回调，并提供调用方需要的兼容返回值。

### 2. 独立页面 Sign 检测入口处理

针对符合特定源码特征的 `Sign.encryptPwd` 入口进行识别和替换。

处理时会继续调用 `Detail.initDg()`，以保留页面原有的初始化流程，避免只移除检测逻辑却遗漏页面初始化。

### 3. 聊天页面专用兼容处理

针对 BOSS 聊天页面（路径以 `/web/geek/chat` 开头）增加专用处理：

- 在页面初始化前监听 `window.iGeekRoot`。
- 定位 `iGeekRoot.aBService.getABData`。
- 仅针对配置项 `nd_result_13912_number_1` 返回指定的兼容值 `QM&Lb`。
- 其他 AB 配置仍由原方法处理，不做统一覆盖。

这部分逻辑用于处理聊天页中与反调试检测启动相关的配置分支。

## 安装

### 环境要求

- 桌面版 Chrome 或 Edge 等 Chromium 浏览器。
- 用户脚本管理器，推荐 [Tampermonkey](https://www.tampermonkey.net/)。
- 访问 BOSS 直聘网页端：<https://www.zhipin.com/>。

### 安装步骤

1. 在浏览器中安装并启用 Tampermonkey。
2. 打开 Tampermonkey 管理面板。
3. 点击「添加新脚本」。
4. 打开本仓库的 [script 源文件](https://raw.githubusercontent.com/songsinfo/boss-anti-debug/main/script)，复制全部内容。
5. 将新建脚本编辑器中的默认内容替换为复制的代码，然后保存。
6. 确认脚本处于启用状态，重新打开或完整刷新 BOSS 页面。

**注意：** 当前仓库中的脚本文件名为 `script`，没有使用常见的 `.user.js` 后缀，因此建议按照上面的方式手动导入代码。

## 生效范围

脚本元数据和运行时均做了范围限制：

| 项目 | 配置 |
| --- | --- |
| 匹配域名 | `https://www.zhipin.com/*` |
| 运行时机 | `document-start` |
| 执行环境 | `raw` |
| 权限 | `@grant none` |
| 页面范围 | 仅顶层页面，不处理 iframe |
| 聊天页专用逻辑 | `/web/geek/chat` 及其子路径 |

脚本只在 HTTPS 的 `www.zhipin.com` 顶层页面运行，不会主动在其他网站或子框架中执行。

## 查看运行状态

打开 BOSS 页面后，按 F12 打开开发者工具，在 Console 中查看以以下前缀开头的日志：

```text
[BOSS 定向防跳转] v0.1.6
```

常见日志含义：

| 日志 | 含义 |
| --- | --- |
| `webpackJsonp 已就绪` | 已安装对应的 Webpack 队列监听 |
| `独立页面 Sign 入口已就绪` | 已安装 Sign 入口监听 |
| `Chat entry guard ready before page initialization.` | 聊天页面专用入口监听已就绪 |
| `已识别并替换反调试模块` | 已识别并替换符合特征的 Webpack 模块 |
| `Chat AB guard applied` | 聊天页指定 AB 配置兼容处理已触发 |
| 包含「未生效」或 `failed` 的日志 | 对应处理未能成功安装或执行，需要进一步排查 |

**注意：** 出现「入口已就绪」只代表监听器安装成功，不代表当前页面一定存在匹配的检测模块。只有出现对应的「已识别并替换」或「guard applied」日志，才表示相应处理实际触发。

## 常见问题

### 打开 F12 后页面还是刷新或跳转

可以按以下顺序检查：

1. 确认 Tampermonkey 中脚本已启用。
2. 确认当前访问的是 `www.zhipin.com`，而不是其他域名。
3. 对 BOSS 页面执行一次完整刷新（Ctrl + F5），确保脚本在页面初始化前运行。
4. 在 Console 中检查是否出现脚本启动日志，以及是否出现「未生效」或异常信息。
5. 如果只有监听器就绪日志，没有模块识别日志，可能是当前页面使用的前端代码与脚本中的特征不一致。

### 聊天页面没有出现 Chat AB guard 日志

聊天页专用逻辑只在路径匹配 `/web/geek/chat` 时安装，并且只有页面实际读取目标 AB 配置项时，才会出现 `Chat AB guard applied` 日志。

### BOSS 页面更新后脚本失效

BOSS 可能调整前端打包方式、模块特征或 AB 配置。当前脚本采用特征匹配，而不是依赖固定的模块 ID；但如果目标逻辑发生变化，仍需要重新分析并更新兼容实现。

如果遇到问题，建议提供以下信息：

- 当前页面 URL（注意隐藏个人信息）。
- Console 中以 `[BOSS 定向防跳转]` 开头的日志。
- 页面发生刷新或跳转前后的现象。
- BOSS 前端资源版本（如果能够获取）。

## 安全与使用说明

- 当前脚本针对页面运行时的特定函数和配置进行处理，不会关闭浏览器全部调试保护。
- 脚本不负责登录、账号认证或权限绕过。
- 当前源码没有主动向第三方服务发送请求或上传用户数据的逻辑。
- 页面运行时逻辑可能随 BOSS 更新而变化，使用前请自行检查源码。
- 请遵守相关网站的服务条款及适用法律法规。

## 版本记录

### v0.1.6

- 增加聊天页面专用兼容逻辑。
- 针对 `iGeekRoot.aBService.getABData` 的指定 AB 配置项进行定向处理。
- 保留原有 Webpack 模块识别和独立页面 Sign 入口处理。

## License

本项目采用 [MIT License](LICENSE) 开源。

Copyright (c) 2026 creamtea47
