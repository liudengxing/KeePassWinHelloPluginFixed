# 与上游 3.3.1 的差异

本分支只做最小必要修改，共 3 个文件，不改变 `ProtectedKey` 序列化格式、
不改变 Credential Manager 的凭据命名、不改变 KeePass 主密码或数据库格式。

## 1. `src/AuthProviders/WinHelloProvider.cs`

### 1.1 固定使用系统默认 NGC 密钥

修改前：

```csharp
private string CurrentKeyName
{
    get { return CurrentCacheType == AuthCacheType.Local ? LocalKeyName : PersistentKeyName; }
}
```

修改后：

```csharp
private string CurrentKeyName
{
    get { return LocalKeyName; }
}
```

原因：新版 Windows 拒绝第三方创建 NGC 持久密钥，插件不应再依赖
`PersistentKeyName`。

### 1.2 构造函数不再要求/创建自建持久密钥

修改前（Persistent 分支）：

```csharp
if (authCacheType == AuthCacheType.Local)
{
    DeletePersistentKey();
}
else
{
    SafeNCryptKeyHandle ngcKeyHandle;
    if (!TryOpenPersistentKey(out ngcKeyHandle))
        throw new AuthProviderKeyNotFoundException("Persistent key does not exist.");
    ...
}
```

修改后：

```csharp
// Newer Windows versions reject NCryptCreatePersistedKey for
// third-party NGC keys. Always use the system default Hello key;
// cross-process persistence is handled by Credential Manager.
if (string.IsNullOrEmpty(LocalKeyName))
    throw new AuthProviderIsUnavailableException("Windows Hello is not available.");
```

同时移除了构造函数中删除持久密钥的副作用。

### 1.3 缓存类型切换只改变存储位置

修改前：`ClaimCurrentCacheType` 会删除或创建 NGC 持久密钥，并可能触发 Hello 创建流程。

修改后：

```csharp
public void ClaimCurrentCacheType(AuthCacheType authCacheType)
{
    if (CurrentCacheType == authCacheType)
        return;

    lock (_mutex)
    {
        // The cache type now only selects the key storage
        // (memory vs. Credential Manager).
        CurrentCacheType = authCacheType;
    }
}
```

### 1.4 加密 / 解密移除持久密钥完整性校验

`Encrypt` 与 `PromptToDecrypt` 中删除：

```csharp
if (CurrentCacheType == AuthCacheType.Persistent && !VerifyPersistentKeyIntegrity(ngcKeyHandle))
    throw new AuthProviderInvalidKeyException(InvalidatedKeyMessage);
```

因为此时使用的已是系统默认密钥，而不是插件自有密钥。

> 未使用的 `PersistentKeyName`、`TryOpenPersistentKey`、`CreatePersistentKey`、
> `VerifyPersistentKeyIntegrity` 方法被保留，以减少改动面；它们不再被调用。

## 2. `src/KeyManagement/Storage/IKeyStorage.cs`

修改前：

```csharp
if (Settings.Instance.WinStorageEnabled && authProvider.CurrentCacheType == AuthCacheType.Persistent)
    return new KeyWindowsStorage();

return new KeyMemoryStorage();
```

修改后：

```csharp
if (Settings.Instance.WinStorageEnabled)
    return new KeyWindowsStorage();

return new KeyMemoryStorage();
```

原因：持久化只应取决于用户是否启用 Windows Storage，而不应要求插件已经拥有
一个自建 NGC 持久密钥。

## 3. `src/Properties/AssemblyInfo.cs`

```diff
-[assembly: AssemblyVersion("3.3.1")]
+[assembly: AssemblyVersion("3.3.2")]
```

同时 `keepass.version` 更新为 `KeePassWinHello:3.3.2`。

## 4. 没有采用的方案

| 方案 | 未采用原因 |
| --- | --- |
| 修改 NGC 密钥名格式（单斜杠 / GUID） | 实测所有格式仍返回 `0x80090027`，无法解决问题 |
| 使用 DPAPI 加密并持久化 | 3.3.x 已不使用 DPAPI；会引入新的密钥保护面，且无法跨设备迁移 |
| 只开启 Persistent 模式 | 旧插件仍会在初始化时 `NCryptCreatePersistedKey` 失败并回退 |
| 改用 `KeyCredentialManager` 创建凭据 | 与插件现有 `ProtectedKey` 格式和 Credential Manager 存储不兼容，改动过大 |
| 仅使用 Local 内存模式 | 重启 KeePass 后快速解锁失效，不满足需求 |

## 5. 设计权衡

- **安全**：仍然每次解锁都调用 Windows Hello；`PinCacheIsGestureRequired=1`
  保证解密必须经过用户手势验证。
- **兼容**：不创建 NGC 密钥，因此不受新版 Windows 对第三方 NGC 持久密钥的限制。
- **可回滚**：凭据格式未变，但新旧插件使用不同 NGC 密钥；回滚时建议清除
  `KeePassWinHello_*` 凭据。
- **最小改动**：保留原有 KeyManager、OptionsPanel、ProtectedKey 逻辑，
  只调整密钥来源与存储判定。