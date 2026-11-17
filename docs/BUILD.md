# 构建与打包

本仓库的 plgx 由 KeePass 自身在启动时编译，因此**不需要 Visual Studio 或 .NET SDK 项目构建**。
下面步骤用于在本地做语法校验并重新生成 `dist/KeePassWinHelloPlugin.plgx`。

## 1. 前置条件

- Windows 10/11
- KeePass 2.x（本项目以 2.61.1 验证）
- .NET Framework 4.x 自带的 C# 编译器：
  `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe`
- PowerShell 5.1+

## 2. 目录结构

```text
.
├─ src/                       # 插件源码（3.3.2）
├─ dist/                      # 发布产物
│  └─ KeePassWinHelloPlugin.plgx
├─ docs/
├─ Pack-Plugin.ps1            # 上游打包脚本（可选）
├─ keepass.version
└─ LICENSE
```

## 3. 编译校验

在仓库根目录打开 PowerShell：

```powershell
$root = (Get-Location).Path
$src  = Join-Path $root "src"
$lib  = Join-Path $root "lib"
New-Item $lib -ItemType Directory -Force | Out-Null
Copy-Item "D:\KeePass Password Safe 2\KeePass.exe" $lib -Force

$files = Get-ChildItem $src -Recurse -Filter *.cs | ForEach-Object { $_.FullName }
& "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe" `
  -nologo -target:library -langversion:5 `
  "-out:$root\KeePassWinHello_test.dll" `
  "-r:$lib\KeePass.exe" `
  -r:System.dll -r:System.Core.dll -r:System.Data.dll `
  -r:System.Drawing.dll -r:System.Windows.Forms.dll -r:System.Xml.dll `
  $files

$LASTEXITCODE   # 必须为 0
```

说明：

- 参数必须使用短横线形式（`-r:` / `-out:`），不要用 `/r:`；在 git-bash 下 MSYS
  路径转换会破坏 `/` 开头的参数。
- `Resources.Designer.cs` 在未嵌入 `.resources` 时也可通过编译，仅用于语法校验。
- 生成的 `KeePassWinHello_test.dll` 不应提交。

## 4. 生成 plgx

KeePass 的 `--plgx-create` 接受一个源码目录，输出同名 `.plgx`：

```powershell
$root = (Get-Location).Path
$pack = Join-Path $root "releases\KeePassWinHelloPlugin"
if (Test-Path $pack) { Remove-Item $pack -Recurse -Force }
New-Item $pack -ItemType Directory -Force | Out-Null

# 复制源码，排除构建残留
Get-ChildItem "$root\src" -Force |
  Where-Object { $_.Name -notin @('bin','obj') } |
  Copy-Item -Destination $pack -Recurse -Force

& "D:\KeePass Password Safe 2\KeePass.exe" --plgx-create "$pack"
```

生成的 `releases\KeePassWinHelloPlugin.plgx` 即新插件。

也可以使用上游脚本：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\Pack-Plugin.ps1 -SkipChoco
```

该脚本默认读取 `lib\KeePass.exe`，否则回退到
`C:\Program Files (x86)\KeePass Password Safe 2\KeePass.exe`；使用前请确认路径。

## 5. 产物校验

```powershell
$plgx = ".\releases\KeePassWinHelloPlugin.plgx"
Get-Item $plgx | Select-Object Length
[BitConverter]::ToString([IO.File]::ReadAllBytes($plgx)[0..3])
```

预期：

- 文件头为 `19-07-D9-65`（KeePass plgx magic，小端 `0x65D90719`）；
- 纯源码包大小约 72 KB（旧 3.3.1 包 265 KB 是因为混入了 bin/obj 残留）。

将产物复制到 `dist/` 后提交。

## 6. 版本号

- `src/Properties/AssemblyInfo.cs`：`AssemblyVersion("3.3.2")`
- `keepass.version`：`KeePassWinHello:3.3.2`

发布新版本时请同步修改以上两处，并在 [CHANGELOG](../CHANGELOG.md) 记录。