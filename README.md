# 闲鱼自动回复 for Synology DSM 7

把 [Heartestrella/xianyu-auto-reply-golang](https://github.com/Heartestrella/xianyu-auto-reply-golang) 的最新版本封装成群晖 DSM 7 套件（SPK），由 GitHub Actions 定时构建并发布。

上游是「闲鱼自动回复系统」的 Golang 重写版：常驻内存小、只依赖内置 SQLite，自带 Web 管理界面，适合长期跑在 NAS 上。

## 特性

- 不依赖 Docker，直接运行上游原生二进制
- 跟随上游 Release 自动构建，通常在上游发版后数小时内产出新包
- 同时提供 x86_64 与 ARM 两种架构
- 构建过程固定时间戳，同版本产物可重现

## 安装

### 手动安装

在 [Releases](../../releases) 下载对应架构的安装包，然后到套件中心 →「手动安装」：

| 架构 | 文件 |
| --- | --- |
| x86_64 机型 | `xianyu-auto-reply_<version>_x86_64.spk` |
| ARM 机型 | `xianyu-auto-reply_<version>_armv8.spk` |

若不确定机型对应的架构，可在 DSM「控制面板 → 信息中心」查看，或参考 [群晖机型列表](https://kb.synology.com/DSM/tutorial/What_kind_of_CPU_does_my_NAS_have)。

### 通过套件源安装

把本仓库的 Release 汇总到自建套件源后，即可在套件中心直接搜索安装。仓库本身不提供在线套件源。

## 使用

安装并启动套件后，浏览器访问：

```
http://<NAS 地址>:18089
```

首次登录账号 `admin`，密码 `admin123`，**登录后请立即修改密码**。

### 端口

默认使用 **18089**，未占用常见 NAS 服务端口。如需修改，编辑：

```
/var/packages/xianyu-auto-reply/var/listen.conf
```

写入 `PORT=自定义端口`，然后重启套件。注意套件中心里的「打开」链接仍指向 18089，改端口后请直接访问新地址。

### 数据

所有数据保存在 `/var/packages/xianyu-auto-reply/var/data`，包括 SQLite 数据库、上传文件、OAuth 令牌与 JWT 密钥。

- 升级套件不会覆盖该目录
- 卸载时选择保留数据即可重新安装后继续使用
- 建议纳入 Hyper Backup 备份范围

## 安全须知

上游项目定位为「个人自用」，默认配置偏宽松。本套件已做能力范围内的收敛，但仍需你自行把关：

**套件层面已处理**

- 以独立低权限账号 `xianyu-auto-reply` 运行，不使用 root
- 数据目录 `/var/packages/xianyu-auto-reply/var/data` 每次启动强制 `0700`，内部文件 `0600`
- JWT 密钥由程序生成并以 `0600` 保存在数据目录
- 默认端口 18089，不占用 DSM 与常见 NAS 服务的端口

**必须自己做的**

1. **首次登录后立刻改密码**。上游默认口令 `admin/admin123` 写在公开代码里，且没有强制改密机制，服务端也无登录限流——暴露到网络即可被接管。
2. **不要直接暴露公网**。确需外网访问时，用 DSM 反向代理 + HTTPS，并叠加访问控制（IP 白名单或额外认证）。
3. 用 DSM 防火墙把 18089 限制在局域网，或只放通你实际使用的网段。
4. 限制套件账号的出网范围（如条件允许），仅放通闲鱼、AI 网关与通知渠道。

**上游已知问题（无法在打包层修复）**

- `/uploads/*` 静态目录未鉴权且可列目录，卡券图片等文件可被未认证访问
- 提货/发货接口 `/api/v1/agree-pickup/*` 挂在鉴权之外，仅凭订单号+订单 ID 即可发货并回显卡密
- 数据库中的闲鱼 Cookie、登录密码、SMTP/支付宝密钥为明文存储
- 部分功能存在 SSRF 与任意本地文件读取面（发布素材、远程接口测试、通知渠道）
- 登出接口为空实现，改密后旧令牌仍有效至过期

以上问题需要上游修复。在修复前，请把本服务当作**仅限内网、单管理员**使用。
## 构建

```bash
python build.py
```

产物输出到 `dist/`。构建流程：

1. 查询上游最新 Release
2. 按架构下载 `xianyu-go-linux-*.tar.gz`
3. 取出 `xianyu-go` 二进制与 `frontend/dist` 前端，组装 `package.tgz` 与 `INFO`
4. 生成 `dist/manifest.json` 与发布说明

仓库结构：

```
build.py               构建脚本
spk_builder.py         SPK 封装实现
app.json               套件元数据与上游资产匹配规则
template/              INFO、conf、scripts、UI 等套件框架文件
.github/workflows/     自动构建与发布
```

最低 DSM 版本：7.0-40851

## 许可证

本仓库的打包脚本采用 MIT 许可证。

套件中分发的程序来自上游项目，采用 **AGPL-3.0**，其许可证原文随套件一起分发（`.../target/LICENSE-UPSTREAM`），源码见上游仓库。两者版权归各自作者所有。
