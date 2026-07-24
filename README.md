# r-nacos APT 仓库

[![Build](https://github.com/qyzhg/r-nacos-apt/actions/workflows/apt-repo.yml/badge.svg)](https://github.com/qyzhg/r-nacos-apt/actions/workflows/apt-repo.yml)
[![Architecture](https://img.shields.io/badge/arch-amd64%20%7C%20arm64-blue)](https://github.com/qyzhg/r-nacos-apt)
[![Suite](https://img.shields.io/badge/suite-stable-green)](https://github.com/qyzhg/r-nacos-apt)

通过 **GitHub Actions** 自动拉取 [nacos-group/r-nacos](https://github.com/nacos-group/r-nacos) 的最新 Release、编译并打包成 `.deb`，再以 **GitHub Pages** 作为 APT 源对外发布。一句话：**在 Debian / Ubuntu 上用 `apt` 安装 rnacos。**

> r-nacos 是用 Rust 实现的 Nacos 服务，更轻量、更快。本仓库只负责 **打包分发**，r-nacos 本体归上游所有。

---

## ✨ 特性

- 🔄 **每日自动更新**：UTC 02:00 定时检查上游 Release，有新版本即自动编译发版。
- 🔏 **GPG 签名**：所有 `Release` / `InRelease` 索引均经签名，`apt` 可校验完整性。
- 📦 **零依赖二进制**：由 `cargo build --release` 产出静态精简二进制，安装即用。
- 🖥️ **双架构**：同时构建 `amd64` 与 `arm64`(aarch64) 两套 `.deb`，x86 服务器与 ARM（树莓派、AWS Graviton 等）都能直接装。
- 🔌 **内置 systemd 服务**：`.deb` 自带 unit 与安装脚本，`apt install` 即自动注册、开机自启并运行，无需手动配置。
- 🚀 **手动触发**：在 Actions 页面可随时 `workflow_dispatch` 手动重建。
- 🧩 **标准 APT 布局**：`pool/` + `dists/stable/main/binary-amd64/`，兼容所有 `apt` 客户端。

---

## 🚀 快速安装

适用于 Debian 11+ / Ubuntu 20.04+（使用现代 `signed-by` 方式）。

```bash
# 1. 下载并导入你的 GPG 公钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://qyzhg.github.io/r-nacos-apt/KEY.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/rnacos.gpg

# 2. 添加 APT 源到系统
echo "deb [signed-by=/etc/apt/keyrings/rnacos.gpg] https://qyzhg.github.io/r-nacos-apt stable main" | sudo tee /etc/apt/sources.list.d/rnacos.list

# 3. 更新并安装 rnacos
sudo apt update
sudo apt install rnacos
```

### 验证安装

```bash
rnacos --version       # 查看版本
which rnacos           # /usr/bin/rnacos
```

服务由 systemd 托管（安装时已自动启动），验证：

```bash
sudo systemctl status rnacos # 应为 active (running)
ss -lntp | grep 8848          # 确认 HTTP 端口在监听
```

控制台：浏览器访问 `http://<服务器IP>:10848/rnacos/`，默认账号 `admin / admin`（**登录后请立即修改密码**）。

服务管理、改端口 / 数据目录 / 环境变量等见下方「🖥️ 服务管理（systemd）」一节。更多运行参数请参考 [r-nacos 官方文档](https://github.com/nacos-group/r-nacos#readme)。

---

## 🇨🇳 国内用户（大陆镜像）

`qyzhg.github.io` 在国内大陆经常被重置（RST），`curl` / `apt` 会失败。已用 Cloudflare 搭了一个国内可达的镜像 **`https://rnacos-img.qyzhg.cc`**——内容和官方源完全一致、同样支持自动更新。

国内服务器把安装命令里的域名换成镜像即可（**公钥不变，和上面一样**）：

```bash
# 1. 下载并导入 GPG 公钥（走镜像，国内可达）
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://rnacos-img.qyzhg.cc/KEY.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/rnacos.gpg

# 2. 添加 APT 源（域名换成镜像）
echo "deb [signed-by=/etc/apt/keyrings/rnacos.gpg] https://rnacos-img.qyzhg.cc stable main" | sudo tee /etc/apt/sources.list.d/rnacos.list

# 3. 更新并安装 rnacos
sudo apt update
sudo apt install rnacos
```

后续升级同样走镜像：`sudo apt update && sudo apt upgrade rnacos`。和官方源完全等价，`apt update` 能正常跟新版。

<details>
<summary><b>镜像偶尔也不稳时的备选（jsDelivr）</b></summary>

若 Cloudflare 镜像在某台机器上也不通，可用 jsDelivr（国内有节点）顶一下，把上面命令里的域名换成：

```
https://cdn.jsdelivr.net/gh/qyzhg/r-nacos-apt@gh-pages
```

注意：jsDelivr 偶尔会出现索引缓存不一致（`InRelease` 和 `Packages` 哈希对不上，导致 `apt install` 报找不到包）。遇到时把 `@gh-pages` 换成具体的 commit（到 [gh-pages 分支提交记录](https://github.com/qyzhg/r-nacos-apt/commits/gh-pages) 复制最新 commit SHA）即可恢复，代价是该源不会自动跟新版。**日常优先用上面的 Cloudflare 镜像。**

</details>

---

## 🔄 更新 rnacos

本仓库每天 UTC 02:00 自动拉取上游最新版本并重新打包，所以「更新」就是刷新 apt 索引后升级。

### 常规更新

```bash
sudo apt update
sudo apt upgrade rnacos          # 等价于 sudo apt install rnacos
```

### 查看版本

```bash
rnacos --version                  # 当前安装的版本

apt list -a rnacos               # 仓库中可用的版本（apt update 之后）
```

### 更新须知

- **新版本还没出现？** 上游刚发布的 release，本仓库最快次日 UTC 02:00 之后才入库。急着用可去 [Actions 页面](https://github.com/qyzhg/r-nacos-apt/actions/workflows/apt-repo.yml) 手动点一次 `Run workflow` 立即触发。
- **数据与配置不会丢。** 你的数据目录（`/var/lib/r-nacos`）和 `systemctl edit` 的 drop-in 配置都不在包里，`apt upgrade` 只更新二进制与 unit，不会动你的数据与自定义配置。
- **升级前建议备份。** 跨大版本升级时，推荐先备份 r-nacos 的数据目录再执行升级，方便回滚。

<details>
<summary><b>需要回滚到旧版本？</b></summary>

APT 索引只登记最新版本，`apt install rnacos=<旧版本号>` 通常会报 "not found"。但历史 `.deb` 仍保留在 pool 里，可直接用 `dpkg -i` 安装指定版本：

```bash
# 将 <version> 与 <arch> 替换为你需要的值，例如 0.8.4 / amd64
wget https://qyzhg.github.io/r-nacos-apt/pool/main/r/rnacos/rnacos_<version>_<arch>.deb
sudo dpkg -i rnacos_<version>_<arch>.deb
```

</details>

---

## 🖥️ 服务管理（systemd）

`.deb` 内置 systemd unit 与安装脚本：`apt install` / `apt upgrade` 时会**自动创建运行用户、注册、启用并启动**服务（开机自启），无需手动配置。

### 常用命令

```bash
sudo systemctl status rnacos        # 查看状态（应为 active (running)）
sudo systemctl restart rnacos       # 重启（改完配置后）
sudo systemctl stop rnacos          # 停止
sudo systemctl start rnacos         # 启动
sudo systemctl disable rnacos       # 取消开机自启（安装时已默认 enable）
sudo journalctl -u rnacos -f        # 实时查看日志
```

### 默认运行配置

| 项 | 值 |
| --- | --- |
| 运行用户 | `rnacos`（安装时自动创建） |
| HTTP / gRPC / 控制台端口 | `8848` / `9848` / `10848` |
| 数据目录 | `/var/lib/r-nacos/nacos_db` |
| 日志等级 | `info`（`RUST_LOG`） |
| 时区 | 东 8 区（`RNACOS_GMT_OFFSET_HOURS=8`） |
| 崩溃自重启 | `Restart=on-failure` |

### 自定义配置（端口 / 数据目录 / 环境变量）

**不要直接改 unit 文件**（升级时会被覆盖），用 drop-in 覆盖：

```bash
sudo systemctl edit rnacos
```

在打开的编辑器里追加（示例：改 HTTP 端口等**非机密**配置）：

```ini
[Service]
Environment=RNACOS_HTTP_PORT=18848
```

保存后重启生效：

```bash
sudo systemctl restart rnacos
```

> ⚠️ **不要把密码、token 等机密写进 `Environment=`**：它以明文存在 unit/drop-in 里，本机其他用户可通过 `systemctl show rnacos` 看到。
>
> **管理员密码推荐直接在控制台修改**（最简单，且不落盘到任何配置文件）。若必须在首次初始化时用配置设定，请放进**权限受限**的环境变量文件，再用 `EnvironmentFile=` 引用（systemd 以 root 读取，`rnacos` 进程继承但读不到文件本身）：
>
> ```bash
> sudo install -m 600 -o root -g root /dev/null /etc/rnacos/env
> echo "RNACOS_INIT_ADMIN_PASSWORD=your-strong-password" | sudo tee /etc/rnacos/env >/dev/null
> sudo systemctl edit rnacos        # 加一行：EnvironmentFile=/etc/rnacos/env
> sudo systemctl restart rnacos
> ```
>
> （`RNACOS_INIT_ADMIN_PASSWORD` 仅在**首次初始化**时生效；若已用默认 `admin/admin` 启动过，改它无效，请到控制台改。）

完整参数列表见 [r-nacos 运行参数说明](https://r-nacos.github.io/docs/notes/env_config/)。

---

## 🔑 GPG 公钥

| 字段 | 值 |
| --- | --- |
| UID | `qyzhg (rnacos) <qyzhg@qyzhg.com>` |
| 算法 | RSA |
| 指纹 | `2F93 F5D0 5240 2A37 497E AF9C B041 1D82 307F 5ABB` |
| Key ID (long) | `B0411D82307F5ABB` |
| 创建日期 | 2026-07-22 |

**在线获取：** <https://qyzhg.github.io/r-nacos-apt/KEY.gpg>

**手动校验指纹：**

```bash
curl -fsSL https://qyzhg.github.io/r-nacos-apt/KEY.gpg | gpg --show-keys --fingerprint
```

<details>
<summary><b>完整公钥（点击展开）</b></summary>

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

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
=EG0z
-----END PGP PUBLIC KEY BLOCK-----
```

</details>

---

## ⚙️ 工作原理

本仓库只有一个核心文件：[`.github/workflows/apt-repo.yml`](.github/workflows/apt-repo.yml)。它定义了一条完整流水线：

```
GitHub Actions 定时触发 (UTC 02:00)
        │
        ▼
 ① 获取上游最新 release tag          (nacos-group/r-nacos)
        │
        ▼
 ② checkout 对应版本源码
        │
        ▼
 ③ cargo build --release             (amd64 原生 + arm64 交叉编译)
        │
        ▼
 ④ 打包 .deb → public/pool/.../     (dpkg-deb)
        │
        ▼
 ⑤ 导出公钥 + 生成索引               (dpkg-scanpackages / apt-ftparchive)
        │
        ▼
 ⑥ GPG 签名 Release                  (InRelease / Release.gpg)
        │
        ▼
 ⑦ 部署到 GitHub Pages               (peaceiris/actions-gh-pages)
```

### 发布后的仓库结构

```
https://qyzhg.github.io/r-nacos-apt/
├── KEY.gpg                                  # 公钥（ASCII 装甲）
├── pool/main/r/rnacos/
│   ├── rnacos_<version>_amd64.deb          # x86_64 安装包
│   └── rnacos_<version>_arm64.deb          # aarch64 安装包
└── dists/stable/
    ├── InRelease                            # 内联签名索引
    ├── Release                              # 元数据 + 校验和（Architectures: amd64 arm64）
    ├── Release.gpg                          # 分离式签名
    ├── main/binary-amd64/
    │   ├── Packages                         # amd64 软件包索引
    │   └── Packages.gz
    └── main/binary-arm64/
        ├── Packages                         # arm64 软件包索引
        └── Packages.gz
```

### 触发方式

| 方式 | 说明 |
| --- | --- |
| `schedule` | `cron: '0 2 * * *'`，每天 UTC 02:00 自动运行 |
| `workflow_dispatch` | 在 [Actions 页面](https://github.com/qyzhg/r-nacos-apt/actions/workflows/apt-repo.yml) 手动点击 `Run workflow` |

---

## ❓ 常见问题

**Q：为什么 `apt update` 报签名错误？**
请确认公钥已导入且指纹为 `2F93F5D0 5240 2A37 497E AF9C B041 1D82 307F 5ABB`。若系统较旧（无 `/etc/apt/keyrings/`），可改用：

```bash
curl -fsSL https://qyzhg.github.io/r-nacos-apt/KEY.gpg | sudo tee /etc/apt/trusted.gpg.d/rnacos.asc
echo "deb https://qyzhg.github.io/r-nacos-apt stable main" | sudo tee /etc/apt/sources.list.d/rnacos.list
```

**Q：安装的版本不是最新？**
流水线每天 UTC 02:00 才检查上游。手动到 Actions 页面触发一次 `Run workflow` 即可立即重建，或等待次日自动更新。

**Q：支持哪些架构？**
同时支持 `amd64` 与 `arm64`(aarch64)。`apt` 会根据本机 CPU 架构自动选取对应的 `.deb`，无需在源行里指定 `arch=`。

**Q：这个仓库和 r-nacos 是什么关系？**
本仓库**不包含** r-nacos 源码，仅为方便 Debian/Ubuntu 用户安装而做的自动打包分发。r-nacos 本体版权归 [nacos-group](https://github.com/nacos-group) 所有，遵循其 Apache-2.0 许可证。

---

## 🔗 相关链接

- 上游项目：[nacos-group/r-nacos](https://github.com/nacos-group/r-nacos)
- 构建流水线：[Actions](https://github.com/qyzhg/r-nacos-apt/actions/workflows/apt-repo.yml)
- APT 源地址：<https://qyzhg.github.io/r-nacos-apt/>
- 问题反馈：[Issues](https://github.com/qyzhg/r-nacos-apt/issues)

---

## 📄 许可证

本仓库的**打包脚本与工作流**本身不附加额外许可证，按需自由使用。
其中分发的 **r-nacos 二进制**来自上游，遵循其 [Apache License 2.0](https://github.com/nacos-group/r-nacos/blob/main/LICENSE)。
