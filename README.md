# KeePassWinHelloPluginFixed

基于 [KeePassWinHello 3.3.1](https://github.com/sirAndros/KeePassWinHello) 的兼容性修复分支，版本号 **3.3.2**。

用于解决新版 Windows 上 `KeePassWinHelloPlugin.plgx` 无法唤起 Windows Hello（人脸 / PIN）、重启 KeePass 后快速解锁失效的问题。

> 本项目是非官方修复分支，与插件原作者无关。原项目采用 MIT 许可证，本分支保留原许可证与版权声明。

---

## 问题现象

在 Windows 11（实测 Build 26300）上：

- 打开 KeePass 数据库时不会弹出 Windows Hello 验证；
- 即使之前成功录入过人脸，KeePass 仍只能输入主密码；
- 重启 KeePass 后，前一次会话中可用的快速解锁全部失效；
- 系统本身 Windows Hello 正常（可用人脸 / PIN 登录，生物识别服务与红外摄像头工作正常）。

## 根因

两个问题叠加：

1. **插件自建持久密钥被新版 Windows 拒绝**

   KeePassWinHello 3.3.1 通过 `NCryptCreatePersistedKey` 在
   `Microsoft Passport Key Storage Provider` 中创建名为
   `SID//KeePassWinHello///KeePassWinHello` 的 NGC 持久密钥。

   在新版 Windows 上，该调用对所有常见名称格式、算法与标志组合均返回
   `NTE_INVALID_PARAMETER (0x80090027)`，插件因此认为持久化存储不可用。

2. **默认的 Local 模式只把密钥放在内存里**

   `WindowsHello.QuickUnlock.WindowsStorage.Enabled` 默认是 `false`，此时插件使用
   `KeyMemoryStorage`：只有同一次 KeePass 进程内「锁定 → 解锁」才能用到快速解锁，
   关闭 KeePass 后密钥即丢失。

详细取证与验证矩阵见 [docs/ROOT-CAUSE.md](docs/ROOT-CAUSE.md)。

## 修复方案

不再尝试创建插件自有的 NGC 持久密钥，改为：

| 环节 | 修复后的行为 |
| --- | --- |
| 加解密密钥 | 使用系统默认 NGC 密钥（`NgcGetDefaultDecryptionKeyName` 返回的密钥） |
| 密钥密文存储 | `WindowsStorage.Enabled=true` 时持久化到 Windows Credential Manager |
| 用户验证 | 每次解密仍由 Windows Hello 弹窗验证（人脸 / PIN），并设置 `PinCacheIsGestureRequired` |
| 存储模式语义 | `Persistent` / `Local` 只决定「Credential Manager」还是「内存」，不再决定密钥类型 |
| NGC 密钥操作 | 不再创建、删除或校验插件自有 NGC 密钥 |

这样既绕开了 `0x80090027`，又保留跨进程 / 重启后的快速解锁能力。

代码差异见 [docs/DIFF.md](docs/DIFF.md)。

## 安装

1. 关闭 KeePass。
2. 备份原插件：把 `Plugins\KeePassWinHelloPlugin.plgx` 复制一份。
3. 用 [dist/KeePassWinHelloPlugin.plgx](dist/KeePassWinHelloPlugin.plgx) 覆盖
   `D:\KeePass Password Safe 2\Plugins\KeePassWinHelloPlugin.plgx`。
4. 删除旧编译缓存（可选，KeePass 会按新插件重新生成）：

   ```powershell
   Remove-Item "$env:LOCALAPPDATA\KeePass\PluginCache\*" -Recurse -Force -ErrorAction SilentlyContinue
   ```

5. 启动 KeePass，先用主密码打开一次数据库；随后锁定工作区，再次打开时即应弹出
   Windows Hello 验证。

完整步骤、验证清单与排障见 [docs/INSTALL.md](docs/INSTALL.md)。

## 必需配置

插件会把快速解锁密钥持久化到 Credential Manager，需要以下配置项
（`%APPDATA%\KeePass\KeePass.config.xml` 的 `<Custom>` 节点）：

```xml
<Item>
    <Key>WindowsHello.QuickUnlock.WindowsStorage.Enabled</Key>
    <Value>true</Value>
</Item>
<Item>
    <Key>WindowsHello.QuickUnlock.ValidPeriod</Key>
    <Value>0</Value>
</Item>
```

- `WindowsStorage.Enabled=true`：启用 Credential Manager 持久化。
- `ValidPeriod=0`：不按时间过期（每次解锁仍需 Windows Hello 验证）。
  如需定期强制输入主密码，可改为毫秒值（例如 `86400000` 表示 24 小时）。

也可以通过 KeePass「工具 → 选项 → WindowsHello」界面勾选
「Windows Storage」启用。

## 构建

源码位于 [src](src)，构建与打包步骤见 [docs/BUILD.md](docs/BUILD.md)。

已验证的构建环境：

- Windows 11 Pro Build 26300
- KeePass 2.61.1
- .NET Framework 4.x `csc.exe`（C# 5）

当前产物：

- `dist/KeePassWinHelloPlugin.plgx`
- 大小：72,838 字节
- 内部程序集版本：3.3.2.0
- SHA256：`9a94d66627494f711d6b55ed5b86fa0ce8c55a7eda07e65912602b64f164f946`

## 已验证 / 未验证

已验证：

- 修改后的 33 个 C# 源文件通过 `csc` 编译，0 错误；
- 新 plgx 能被 KeePass 2.61.1 成功编译并加载；
- 编译缓存中的 `KeePassWinHello.dll` 版本为 **3.3.2.0**；
- 系统默认 NGC 密钥可完成一次 `NCryptEncrypt` / `NCryptDecrypt` 往返，
  且解密过程成功触发 Windows Hello 验证（roundtrip MATCH）。

未在本仓库内自动验证：

- 「重启 KeePass 后在全新进程中仅凭人脸进入数据库」的端到端结果。
  这一步需要数据库主密码，必须由使用者本人完成一次。请按
  [docs/INSTALL.md](docs/INSTALL.md) 的验证清单确认。

## 已知限制

- 系统必须已经注册 Windows Hello（PIN 或人脸）；未注册时插件仍不可用。
- 首次启用仍需用主密码解锁一次数据库，插件才能保存加密后的主密钥。
- 如果之后重置了 Windows Hello 或系统默认 NGC 密钥失效，需要清除旧凭据并重新保存：
  在 KeePass 选项的 WindowsHello 页勾选撤销全部密钥，或用主密码打开一次并锁定。
- 每一个数据库路径对应一条 Credential Manager 凭据；移动数据库路径后需要重新保存。
- `%APPDATA%\KeePass\ProtectedUserKey.bin` 是旧版本 / 旧环境遗留文件，
  3.3.x 不使用它，可备份后删除。

## 安全说明

- 不在磁盘上保存明文主密码；Credential Manager 中保存的是经 NGC 密钥加密的密文。
- 每次解锁都会调用 Windows Hello 验证，未降低验证强度。
- 密文由系统默认 NGC 密钥加密，密钥受 TPM / VBS 与 Windows Hello 保护。
- 本修复不改变 KeePass 主密码、数据库加密格式或 key file 内容。

## 上游与致谢

- 上游项目：[sirAndros/KeePassWinHello](https://github.com/sirAndros/KeePassWinHello)
- 上游版本：3.3.1
- 原作者：Andrei Osipkov, Alexander Sitnikov

## 许可证

[MIT](LICENSE)，保留原始版权声明。