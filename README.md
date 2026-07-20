# OpenWRT 云编译

基于 [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI) 定制的个人 OpenWRT 云编译仓库，专门为 **IPQ60XX 平台无 WIFI 设备（如 ZN-M2）** 编译 ImmortalWrt 旁路由固件，通过 GitHub Actions 一键编译，开箱即用。

本仓库不包含 OpenWRT 源码，仅存放云编译所需的 CI 配置、设备配置与自定义脚本。编译时拉取 [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt) 源码，应用定制后产出固件发布到 Releases。

## 定制要点

- **单配置精简**：只编译 `IPQ60XX-WIFI-NO`（裁剪无线驱动，适合旁路由/软路由），移除其它平台与 WIFI 配置。
- **关闭自动构建**：工作流仅手动触发（`workflow_dispatch`）。
- **预置旁路由模式**：固件首次启动自动配置为旁路由，无需手动设置。
- **精简插件**：删除 dockerman，启用 OpenClash、AdGuard Home、HomeProxy。
- **全套主题**：8 套 LuCI 主题全部编译进固件，可随时切换。
- **定制固件信息**：主机名 `OpenWrt`，管理 IP `192.168.50.254`，固件标识 `huxiaozhangz`。

## 目录结构

```
OpenWRT/
├── .github/workflows/       # GitHub Actions 工作流
│   ├── Auto-Clean.yml       # 清理旧 Releases 与 Workflows（手动）
│   ├── Cache-Clean.yml      # 清理 Actions 缓存（手动）
│   ├── QCA-ALL.yml          # 编译入口（固定 IPQ60XX-WIFI-NO）
│   └── WRT-CORE.yml         # 公用编译核心（被 QCA-ALL 调用）
├── Config/                  # .config 配置片段
│   ├── GENERAL.txt          # 通用配置（所有设备共用）
│   └── IPQ60XX-WIFI-NO.txt  # IPQ60XX 无线裁剪版设备配置
├── Scripts/                 # 自定义脚本
│   ├── Packages.sh          # 第三方插件源码拉取与更新
│   ├── Handles.sh           # 补丁与源码处理
│   └── Settings.sh          # 主题、IP、WIFI、旁路由等设置注入
├── LICENSE                  # MIT
└── README.md
```

## 支持设备

仅编译 **IPQ60XX-WIFI-NO** 配置，无线驱动已裁剪，适合做旁路由/软路由。包含设备（节选）：ZN-M2、小米 AX1800、红米 AX5、360 V6、GL.iNet AX1800/AXT1800、Linksys MR7350/MR7500、京东云 RE-CS-02/RE-SS-01、中移 AX18 等。完整列表见 `Config/IPQ60XX-WIFI-NO.txt`。

## 默认配置

| 项目 | 默认值 |
|------|--------|
| 编译配置 | IPQ60XX-WIFI-NO（无 WIFI 裁剪版） |
| 编译源码 | VIKINGYFY/immortalwrt（main 分支） |
| 主机名 | OpenWrt |
| 默认主题 | aurora（另编译 argon / kucat / noobwrt / shadcn / fluent 共 8 套） |
| 固件标识 | huxiaozhangz（显示在 LuCI 状态页固件版本后） |
| 管理 IP | 192.168.50.254 |
| 登录密码 | 默认无（首次登录后请自行设置） |

## 旁路由模式

固件首次启动时，`/etc/uci-defaults/99-bypass-router` 自动执行，将设备配置为旁路由：

| 项目 | 配置 |
|------|------|
| LAN IP | 静态 `192.168.50.254` / `255.255.255.0` |
| 网关 | `192.168.50.1`（主路由） |
| DNS | `192.168.50.1`（主路由） |
| DHCP | 关闭，由主路由统一分配 |
| IPv6 | 关闭 LAN 的 RA / DHCPv6 |
| 防火墙 | LAN 区域开启 NAT 伪装（masquerade） |

使用要点：

1. 刷机后访问 `http://192.168.50.254` 进入管理界面。
2. 局域网设备的网关 / DNS 指向 `192.168.50.254` 即可走旁路由的 OpenClash 代理。
3. 若要让 AdGuard Home 接管设备 DNS，需在主路由 DHCP 中把下发 DNS 改为 `192.168.50.254`。

如需修改旁路由 IP / 网关，编辑 `Scripts/Settings.sh` 末尾的 `99-bypass-router` 脚本块。

## 插件清单

**编译进固件（已启用）**：luci-app-homeproxy、luci-app-openclash、luci-app-adguardhome、luci-app-autoreboot，以及 8 套 LuCI 主题。

**拉取但默认未启用**（在 `Config/GENERAL.txt` 加 `=y` 开启，或在触发编译时用 `PACKAGE` 输入框临时追加）：

- 代理类：passwall、passwall2、nikki、momo、luci-app-tailscale
- 网络工具：mosdns、ddns-go、easytier、netspeedtest、netwizard、openlist2、vnt、qmodem
- 文件共享：quickfile、qbittorrent
- 系统工具：timecontrol、diskman
- 综合包（VIKINGYFY/packages）：axonhub、gecoosac、sing-box、timewol、wolplus、wolultra

## 私有扩展

- `Scripts/PRIVATE.sh`：私有脚本，在 `Packages.sh` 末尾被 source 执行。
- `Config/PRIVATE.txt`：私有配置，在 `Settings.sh` 中被追加到 `.config`。

## 使用方法

1. **Fork** 本仓库到自己的 GitHub 账号。
2. 进入 Fork 仓库的 **Actions** 页面，启用工作流。
3. 选择 **QCA-ALL** 工作流，点击 **Run workflow**，按需填写：
   - `PACKAGE`：临时追加插件配置，例如 `CONFIG_PACKAGE_luci-app-passwall=y`，多个用换行隔开。
   - `TEST`：勾选则只输出 `.config` 不编译固件，用于验证配置。
4. 等待编译完成（约 1-2 小时），到 **Releases** 页面下载对应设备固件。
5. 辅助工作流：`Auto-Clean` 清理旧 Releases 与历史运行记录，`Cache-Clean` 清空 Actions 缓存。

> 编译产物按 `IPQ60XX-WIFI-NO-VIKINGYFY-immortalwrt-main-日期` 打 Tag 发布到 GitHub Releases，内含多个设备固件，请按设备名选择。

## 编译流程

```
手动触发 QCA-ALL
      │
      ▼
WRT-CORE.yml（公用编译核心）
      │
      ├─ 拉取 immortalwrt 源码（main 分支）
      ├─ 执行 Packages.sh（拉取第三方插件）
      ├─ 执行 Handles.sh（补丁处理）
      ├─ 应用 Config/IPQ60XX-WIFI-NO.txt + GENERAL.txt
      ├─ 执行 Settings.sh（主题/IP/旁路由注入）
      ├─ make defconfig → make 编译
      └─ 上传固件到 Releases
```

## 相关资源

- 官方版 ImmortalWrt：https://github.com/immortalwrt/immortalwrt
- 自用版 ImmortalWrt：https://github.com/VIKINGYFY/immortalwrt
- 高通 U-BOOT（沉心）：https://github.com/chenxin527/uboot-ipq60xx-emmc-build 等
- 高通 U-BOOT（小猪）：https://github.com/1980490718/u-boot-2016
- 联发科 U-BOOT：https://github.com/VIKINGYFY/UBOOT-CI/releases

## 致谢

- [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI) —— 云编译框架
- [immortalwrt/immortalwrt](https://github.com/immortalwrt/immortalwrt) —— 固件源码
- 各第三方插件作者（vernesong、Openwrt-Passwall、sirpdboy、sbwml、eamonxg 等）

## 许可

MIT License
