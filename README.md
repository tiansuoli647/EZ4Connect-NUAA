# EZ4Connect-NUAA

*针对南京航空航天大学 (NUAA) VPN 优化的 EZ4Connect 专属分支*

![Action](https://github.com/chenx-dust/EZ4Connect/actions/workflows/build.yml/badge.svg)
![Release](https://img.shields.io/github/v/release/chenx-dust/EZ4Connect)
![Downloads](https://img.shields.io/github/downloads/chenx-dust/EZ4Connect/total)
![License](https://img.shields.io/github/license/chenx-dust/EZ4Connect)

改进的 ZJU-Connect 图形界面，默认配置预设为南京航空航天大学易连 (EasyConnect) 协议连接参数。

## 使用方式

1. 在本项目的 [Releases](https://github.com/chenx-dust/EZ4Connect/releases) 页面下载最新版本：

- **Windows 用户**：下载 `EZ4Connect-vX.X.X-windows-ARCH.zip` ，解压至同一目录下，双击运行 `EZ4Connect.exe` ；
  - 如果遇到缺少 DLL 等问题，请先下载安装 Microsoft Visual C++ 可再发行程序包版本（[x64](https://aka.ms/vs/17/release/vc_redist.x64.exe) | [arm64](https://aka.ms/vs/17/release/vc_redist.arm64.exe)），再运行程序；

- **Linux 用户**：下载 `EZ4Connect-vX.X.X-linux-ARCH.AppImage` ，赋予执行权限，运行即可；
  - AppImage x64 仅支持系统 `glibc >= 2.31` 的发行版，Ubuntu 22.04 及以上版本可以正常运行（受限于 GitHub Actions Runner）；
  - AppImage arm64 仅支持系统 `glibc >= 2.38` 的发行版，Ubuntu 24.04 及以上版本可以正常运行（受限于 Qt 官方：[参考](https://doc.qt.io/qt-6/supported-platforms.html)）；
  - Arch Linux 用户推荐使用 [AUR](https://aur.archlinux.org/packages/ez4connect) 安装；
  - 如果遇到因依赖问题无法运行的情况，请自行编译运行。

<div align="center">
<img src="docs/main.png" width="600px">
</div>

2. 在 “文件”-“设置”-“核心” 页面中设置好服务器地址；

<div align="center">
<img src="docs/proxy_setting.png" width="400px">
</div>

3. 在 “文件”-“设置”-“认证” 页面中，选择对应的服务器类型，配置认证信息；

> [!NOTE]
> aTrust 服务器支持多种认证方式，如需选择请点击“获取认证方式”按钮。除非您十分了解登录信息，否则不建议手动配置。

<div align="center">
<img src="docs/auth_method.png" width="400px">
</div>

4. 在主界面中点击“连接服务器”。如果只需进行校园网页浏览，则选择“设置系统代理”后即可使用。

## 配合 Clash Verge 联合使用方法

若希望在使用 Clash Verge 的同时，自动将南航内网流量分流到 EZ4Connect-NUAA，可采用以下两种全局扩展配置方式之一。

### 准备工作
1. 运行并连接 **EZ4Connect-NUAA**。
2. **务必确保**在 EZ4Connect-NUAA 主界面中**关闭**“自动设置系统代理”和“TUN 模式”开关，以防止与 Clash Verge 的代理发生冲突。
3. 确保 Clash Verge 运行在 **规则模式 (Rule)** 并已启用 **系统代理 (System Proxy)**。

---

### 方式 A：使用 YAML 全局扩展覆写 (Merge)
1. 打开 Clash Verge，进入 **订阅 (Profiles)** 页面。
2. 找到右侧的 **全局扩展覆写配置** 磁贴，点击其右侧的蓝色 **Merge** 按钮。
3. 在弹出的文本编辑器中，清空原有内容（若有），复制并粘贴以下 YAML 配置：

```yaml
prepend-proxies:
  - name: "NUAA-VPN"
    type: socks5
    server: 127.0.0.1
    port: 11080

prepend-rules:
  - DOMAIN-SUFFIX,nuaa.edu.cn,NUAA-VPN
  - IP-CIDR,10.0.0.0/8,NUAA-VPN
```

4. 保存并关闭编辑器，右键点击该磁贴选择 **启用 (Enable)** 激活。

---

### 方式 B：使用 JS 全局扩展脚本 (Script)
1. 打开 Clash Verge，进入 **订阅 (Profiles)** 页面。
2. 找到右侧的 **全局扩展脚本** 磁贴，点击其右侧的蓝色 **Script** 按钮。
3. 在弹出的文本编辑器中，粘贴以下 JavaScript 代码：

```javascript
function main(config) {
  // 1. 配置 DNS 策略解析，对南航域名使用南航内网 DNS (10.10.10.10) 进行解析，防范 fake-ip 模式下的 DNS 泄漏中断
  if (!config.dns) {
    config.dns = {};
  }
  config.dns.enable = true;
  config.dns["enhanced-mode"] = "fake-ip";
  
  if (!config.dns["nameserver-policy"]) {
    config.dns["nameserver-policy"] = {};
  }
  config.dns["nameserver-policy"]["*.nuaa.edu.cn"] = ["10.10.10.10"];

  // 2. 定义南航 VPN 本地 SOCKS5 节点
  const nuaaProxy = {
    name: "NUAA-VPN",
    type: "socks5",
    server: "127.0.0.1",
    port: 11080
  };

  // 3. 将节点插入到 proxies 列表最前面
  if (!config.proxies) {
    config.proxies = [];
  }
  config.proxies.unshift(nuaaProxy);

  // 4. 定义分流规则
  const nuaaRules = [
    "DOMAIN-SUFFIX,nuaa.edu.cn,NUAA-VPN",
    "IP-CIDR,10.0.0.0/8,NUAA-VPN"
  ];

  // 5. 将规则插入到 rules 列表最前面（优先级最高）
  if (!config.rules) {
    config.rules = [];
  }
  config.rules = nuaaRules.concat(config.rules);

  return config;
}
```

4. 保存并关闭编辑器，右键点击该磁贴选择 **启用 (Enable)** 激活。

---

若需了解更详细的原理及分流参数调整，可参考：[高级使用方式 (docs/ADVANCED_USAGE.md)](docs/ADVANCED_USAGE.md)

## 路线图

如有更多好的建议，可以在 Issue 中或是 OSA 群里提出！


- [X] 支持 Linux 系统
- [X] 支持手动设置 Proxy Bypass
- [X] 上传 AUR 包
- [ ] 使用密钥链存储密码等信息

## 编译要求

编译本项目需要以下环境和依赖：
- **CMake**: 最低要求 `3.12`
- **C++ 编译器**: 支持 `C++17` 的编译器（如 MSVC 2019+，GCC 9+，Clang 10+）
- **Qt6 SDK**: 需安装以下组件：
  - `Qt6Core`
  - `Qt6Gui`
  - `Qt6Widgets`
  - `Qt6Network`
  - `Qt6Svg`
  - `Qt6Core5Compat`
  - `Qt6WebEngine` (包含 `WebEngineWidgets`)

### 编译步骤
1. 克隆本项目：
   ```bash
   git clone https://github.com/your-username/EZ4Connect-NUAA.git
   cd EZ4Connect-NUAA
   ```
2. 使用 CMake 配置并构建：
   ```bash
   mkdir build
   cd build
   cmake ..
   cmake --build . --config Release
   ```
3. 运行前准备：
   - 运行程序需要依赖 `zju-connect` 底层核心可执行文件。编译完成后，请将适用于您平台版本的 `zju-connect` (或 `zju-connect.exe`) 放置于生成的可执行文件同级目录下。
   - Windows 用户如果需要使用 TUN 模式，还必须将 `wintun.dll` 放置在同级目录下。

## 致谢

- [Mythologyli/ZJU-Connect-for-Windows](https://github.com/Mythologyli/ZJU-Connect-for-Windows)
- [Mythologyli/zju-connect](https://github.com/Mythologyli/zju-connect)

> 欢迎加入 HITSZ 开源技术协会 [@hitszosa](https://github.com/hitszosa)
