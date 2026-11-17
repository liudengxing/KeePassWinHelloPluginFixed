# Changelog

本分支基于上游 KeePassWinHello 3.3.1。

## [3.3.2] - 2026-11-17

### Fixed

- 修复新版 Windows 上无法唤起 Windows Hello 的问题。
  旧版通过 `NCryptCreatePersistedKey` 创建自有 NGC 持久密钥，在 Windows 11
  Build 26300 上返回 `NTE_INVALID_PARAMETER (0x80090027)`，导致 Persistent
  模式初始化失败。
- 修复默认配置下关闭 KeePass 后快速解锁失效的问题。
  旧版默认使用 `KeyMemoryStorage`，密钥仅存在于当前进程内存。
- 修复 `WindowsStorage.Enabled=true` 仍需自建 NGC 密钥才能生效的问题。

### Changed

- 加解密改为复用系统默认 NGC 密钥（`NgcGetDefaultDecryptionKeyName`）。
- `AuthCacheType` 只决定密钥密文的存储位置：
  - `Persistent` → Windows Credential Manager；
  - `Local` → 进程内存。
- `KeyStorageFactory.Create` 只检查 `WindowsHello.QuickUnlock.WindowsStorage.Enabled`。
- `ClaimCurrentCacheType` 不再创建或删除 NGC 密钥。
- 移除加密 / 解密路径中的插件自有密钥完整性校验。
- 版本号提升至 `3.3.2.0`。

### Verified

- 33 个 C# 源文件通过 .NET Framework 4.x `csc` 编译，0 错误。
- 重新打包的 plgx 可被 KeePass 2.61.1 成功编译并加载；
  `PluginCache` 中的 DLL 版本为 `3.3.2.0`。
- 系统默认 NGC 密钥完成一次 `NCryptEncrypt` / `NCryptDecrypt` 往返，
  解密阶段成功触发 Windows Hello 验证，结果逐字节一致。
- 安装后 KeePass 启动无插件编译 / 加载错误对话框。

### Known limitations

- 端到端「重启 KeePass 后仅用人脸进入」仍需使用者用主密码完成首次保存后验证。
- 系统必须已注册 Windows Hello。
- 移动数据库路径后需要重新保存凭据。
- 重置 Windows Hello 后需撤销旧凭据并重新保存。