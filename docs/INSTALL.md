# 安装、验证与回滚

## 1. 安装

1. 完全退出 KeePass（确认任务管理器中无 `KeePass.exe`）。
2. 备份现有插件：

   ```powershell
   Copy-Item "D:\KeePass Password Safe 2\Plugins\KeePassWinHelloPlugin.plgx" `
             "D:\KeePass Password Safe 2\Plugins\KeePassWinHelloPlugin.plgx.bak"
   ```

3. 复制本仓库产物：

   ```powershell
   Copy-Item ".\dist\KeePassWinHelloPlugin.plgx" `
             "D:\KeePass Password Safe 2\Plugins\KeePassWinHelloPlugin.plgx" -Force
   ```

4. 清理旧编译缓存：

   ```powershell
   Remove-Item "$env:LOCALAPPDATA\KeePass\PluginCache\*" -Recurse -Force -ErrorAction SilentlyContinue
   ```

5. 启动 KeePass。首次启动时插件会被重新编译，KeePass 可能短暂停顿，属正常现象。

## 2. 配置

确认 `%APPDATA%\KeePass\KeePass.config.xml` 的 `<Custom>` 下包含：

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

修改前请退出 KeePass，否则退出时会被内存中的配置覆盖。

也可以在 KeePass 中操作：

1. 「工具 → 选项」；
2. 打开「WindowsHello」页；
3. 勾选“Windows Storage / 使用 Windows 凭据管理器存储”；
4. 点击确定。

## 3. 首次使用（必须由本人完成一次）

1. 用**主密码**（以及 key file，如果数据库需要）打开数据库；
2. 打开成功后按 `Ctrl+L` 锁定工作区，或关闭数据库；
3. 再次打开该数据库。

此时应出现 Windows Hello 人脸 / PIN 验证，验证通过后直接进入。

首次用主密码解锁是必要的：插件需要拿到数据库主密钥，才能加密后写入
Credential Manager。

## 4. 验证清单

### 4.1 插件版本

```powershell
Get-ChildItem "$env:LOCALAPPDATA\KeePass\PluginCache" -Recurse -Filter KeePassWinHello.dll |
  ForEach-Object { $_.VersionInfo.FileVersion }
```

预期：`3.3.2.0`。

### 4.2 凭据是否写入

```powershell
cmdkey /list | Select-String "KeePassWinHello"
```

预期出现类似：

```text
目标: KeePassWinHello_E:\JianGuoYun\keepass\KeePass.kdbx
```

每个数据库路径对应一条凭据。

### 4.3 重启后验证

1. 完全退出 KeePass；
2. 重新启动；
3. 打开数据库。

预期：不再要求主密码，直接弹出 Windows Hello，验证后进入。

### 4.4 系统 Hello 可用性

```powershell
Get-Service WbioSrvc
```

或在「设置 → 账户 → 登录选项」确认已注册人脸 / PIN。

## 5. 排障

### 5.1 完全不弹 Hello

依次检查：

1. `WinStorageEnabled` 是否为 `true`；
2. Credential Manager 是否有对应数据库路径的 `KeePassWinHello_*` 凭据；
3. 是否至少成功用主密码打开并锁定过一次数据库；
4. `WinHelloProvider` 初始化是否因系统未注册 Hello 被禁用
   （KeePass「工具 → 选项 → WindowsHello」页会提示
   “Windows Hello is disabled on your system”）。

### 5.2 弹窗后提示验证失败

- 先在 Windows 设置中确认人脸 / PIN 可用；
- 如果近期重置过 Windows Hello，旧凭据中的密文已无法解密：
  用主密码打开数据库，在插件选项中撤销全部密钥，再锁定一次重新保存。

### 5.3 插件加载失败 / 无 WindowsHello 选项页

- 确认只放入了新的 `KeePassWinHelloPlugin.plgx`，不要混入旧的 `KeePassWinHello.dll`；
- 删除 `%LOCALAPPDATA%\KeePass\PluginCache\*` 后重启 KeePass；
- 查看 KeePass 是否弹出插件编译错误，并检查 .NET Framework 4.x 是否完整。

### 5.4 `0x80090027` 相关问题

该错误属于旧版插件自建 NGC 密钥的路径。3.3.2 已不再调用
`NCryptCreatePersistedKey`，正常使用不会出现。若仍看到该错误，说明加载的仍是旧插件，
请检查 `PluginCache` 中的 DLL 版本。

## 6. 回滚

```powershell
Copy-Item "D:\KeePass Password Safe 2\Plugins\KeePassWinHelloPlugin.plgx.bak" `
          "D:\KeePass Password Safe 2\Plugins\KeePassWinHelloPlugin.plgx" -Force
Remove-Item "$env:LOCALAPPDATA\KeePass\PluginCache\*" -Recurse -Force -ErrorAction SilentlyContinue
```

回滚后建议在 Credential Manager 中删除 `KeePassWinHello_*` 凭据，避免新旧版本
使用不同密钥格式导致解密失败：

```powershell
cmdkey /list | Select-String "KeePassWinHello"
# 逐条删除：
cmdkey /delete:KeePassWinHello_<数据库路径>
```

## 7. 关于旧文件

`%APPDATA%\KeePass\ProtectedUserKey.bin` 是旧版本 / 旧环境残留，3.3.x 不使用它。
确认新方案工作正常后，可备份并删除：

```powershell
Move-Item "$env:APPDATA\KeePass\ProtectedUserKey.bin" `
          "$env:APPDATA\KeePass\ProtectedUserKey.bin.old"
```