# 根因分析

本文记录 2026-11-17 在故障机器上的诊断过程与证据。结论已通过代码与系统 API 双向验证。

## 1. 测试环境

| 项目 | 值 |
| --- | --- |
| 操作系统 | Windows 11 专业版，Build 26300 |
| KeePass | 2.61.1.0 |
| 插件 | KeePassWinHello 3.3.1.0（原始 plgx） |
| 账户 | 本地账户 `STARSTORM\StarStorm` |
| 摄像头 | HP IR Camera + HP Wide Vision FHD Camera |
| Hello 设备 | Facial Recognition (Windows Hello) Software Device，状态 OK |
| 生物识别服务 | `WbioSrvc` 已启动；`KeyIso`、`VaultSvc` 运行中 |
| VBS | 运行中（HVCI 已启用） |

系统级 Windows Hello 是可用的：

- `UserConsentVerifier.CheckAvailabilityAsync()` 返回 `Available`；
- `KeyCredentialManager.IsSupportedAsync()` 返回 `True`；
- 生物识别事件日志中存在 `1605`（安全组件授权）与 `1702`（取消数据保护）记录；
- 直接使用系统默认 NGC 密钥做加解密，解密阶段成功弹出验证并完成（见第 4 节）。

因此问题不在摄像头、生物识别服务或 Windows Hello 注册本身。

## 2. 插件 3.3.1 的真实工作机制

阅读上游 3.3.1 源码后确认：

- 密钥存储：Windows Credential Manager（`CredWrite` / `CredRead`，target 为
  `KeePassWinHello_<数据库路径>`）。
- 加解密：`Microsoft Passport Key Storage Provider` 中的 NGC 密钥。
- **不使用 DPAPI，也不读取 `ProtectedUserKey.bin`。**

`%APPDATA%\KeePass\ProtectedUserKey.bin`（294 字节，2025-09-29）：

- 文件头是标准 DPAPI blob（`01000000 D08C9DDF0115D1118C7A00C004FC297EB`）；
- 使用当前用户、空 entropy 以及常见字符串 entropy 均无法解密
  （`0x8007000D ERROR_INVALID_DATA`）；
- 该文件不属于 3.3.1 的任何代码路径。

结论：它是旧版本 / 旧机器 / 其他工具留下的残留，与当前故障无关。可备份后删除，不要把它当作正在使用的密钥。

## 3. 关键失败点：自建 NGC 持久密钥被拒绝

3.3.1 的 `WinHelloProvider` 在 Persistent 模式下调用：

```csharp
NCryptCreatePersistedKey(
    provider,
    out key,
    "RSA",
    SID + "//KeePassWinHello///KeePassWinHello",
    0,
    CngKeyCreationOptions.None);
```

在本机实际探测结果：

| 参数组合 | 结果 |
| --- | --- |
| 名称 `SID//KeePassWinHello///KeePassWinHello`，RSA，legacy=0，flags=0 | `0x80090027` |
| 名称 `SID//name///GUID` | `0x80090027` |
| 名称 `SID//GUID///name` | `0x80090027` |
| 名称 `SID/GUID///name` | `0x80090027` |
| 使用系统默认密钥前缀 `SID/<default-tenant-GUID>///...` | `0x80090027` |
| RSA / ECDSA_P256 / ECDSA_P384 | 全部 `0x80090027` |
| legacy key spec 0 / 1 / 2 | 全部 `0x80090027` |
| flags 0 / `NCRYPT_MACHINE_KEY_FLAG` | 全部 `0x80090027` |

`0x80090027` 即 `NTE_INVALID_PARAMETER`。这不是名称拼写问题，而是该版本 Windows /
当前 Hello 配置下，第三方进程通过 `NCryptCreatePersistedKey` 创建 NGC 持久密钥被整体拒绝。

插件随后无法取得持久密钥，Persistent 模式初始化失败；而默认配置走 Local 模式：

- `KeyStorageFactory.Create` 只有
  `WinStorageEnabled && CurrentCacheType == Persistent` 时才使用 `KeyWindowsStorage`；
- 否则使用 `KeyMemoryStorage`，密钥仅存在于当前进程内存；
- 因此关闭 KeePass 后密钥丢失，下一次只能输入主密码。

同时检查到：Credential Manager 中**没有任何** `KeePassWinHello_*` 凭据，
说明这台机器上从未成功执行过持久化保存。

## 4. 系统默认 NGC 密钥可用（修复方案的基础）

通过 `NgcGetDefaultDecryptionKeyName` 取得系统默认密钥：

```text
S-1-5-21-...-1001/b6497f97-60e4-49a5-b8bb-aa62191894bc///9DDC52DB-DC02-4A8C-B892-38DEF4FA748F
```

随后执行：

1. `NCryptOpenStorageProvider("Microsoft Passport Key Storage Provider")` → 成功；
2. `NCryptOpenKey(default, CngKeyOpenOptions.Silent)` → 成功；
3. `NCryptEncrypt` 32 字节 → 成功（输出 256 字节）；
4. `NCryptOpenKey(default, CngKeyOpenOptions.None)` → 成功；
5. 设置 `PinCacheIsGestureRequired=1`；
6. `NCryptDecrypt` → **成功**，并且触发了 Windows Hello 验证；
7. 解密结果与原文逐字节一致（roundtrip MATCH）。

这说明：

- 系统默认 NGC 密钥可以被第三方进程用于加解密；
- 解密会正常弹出 Windows Hello；
- 不创建自建 NGC 密钥也能实现「人脸 / PIN 解锁」。

## 5. 最终结论

| 层级 | 结论 |
| --- | --- |
| 硬件 / 系统 | 正常，Windows Hello 可用 |
| 旧 `ProtectedUserKey.bin` | 无效残留，与 3.3.1 无关 |
| 插件持久化 | 自建 NGC 密钥被新版 Windows 拒绝（`0x80090027`） |
| 插件默认模式 | Local 内存存储，进程结束即丢失 |
| 修复方向 | 复用系统默认 Hello 密钥 + Credential Manager 持久化 |

## 6. 相关文件

- `src/AuthProviders/WinHelloProvider.cs`
- `src/KeyManagement/Storage/IKeyStorage.cs`
- `src/Properties/AssemblyInfo.cs`

具体改动见 [DIFF.md](DIFF.md)。