# PowerShell 指南

PowerShell 主要在本仓库中用作基础设施的骨干。因此，PowerShell 脚本需要具有可移植性、可靠性，并采用遇错停止的方式。下面的指南旨在推动脚本朝这种执行思维模式发展。

## 编码风格

1. 开括号始终放在表达式/语句的末尾。
1. 闭括号始终出现在单独的空行上。
1. 使用两个空格缩进。
1. 对函数使用帕斯卡命名法，并尽可能遵循"动词-名词"约定。
1. 对所有其他标识符使用驼峰命名法。
1. 使用完整命令名称而不是别名。例如使用 `Get-Content` 而非 `gc`。别名可能被环境覆盖，因此不被视为可移植的。

## 通用指南

### 脚本模板

所有在 CI 中执行的 PowerShell 脚本都需要符合以下模板。

```powershell
[CmdletBinding(PositionalBinding=$false)]
param (
    [switch]$ci = $false,
    [string]$configuration = "Debug")

Set-StrictMode -version 2.0
$ErrorActionPreference="Stop"

try { 
  . (Join-Path $PSScriptRoot "build-utils.ps1")
  $prepareMachine = $ci

  # 脚本主体在此处

  ExitWithExitCode 0
}
catch {
  Write-Host $_
  Write-Host $_.Exception
  Write-Host $_.ScriptStackTrace
  ExitWithExitCode 1
}
```

这些部分的原理如下：

- `Set-StrictMode`：强制 PowerShell 进入更严格的解释模式，将"遇错继续"方式替换为"遇错停止"模式。这两者都有助于我们实现可靠性目标，因为它使错误成为强制停止（除非特别说明）。
- `$prepareMachine = $ci`：必要的，以确保 `ExitWithExitCode` 在 CI 环境中正确退出所有构建进程（符合 arcade 指南）。
- `ExitWithExitCode`：处理进程管理的脚本退出的 arcade 标准。
- `PositionalBinding=$false`：当调用者使用了不正确的参数调用脚本时发出警告。

### 编码指南

使用 `Exec-*` 函数执行程序或 `dotnet` 命令。这会在调用失败、参数不正确等情况下自动检测错误。

```powershell
# 不推荐
& msbuild /v:m /m Roslyn.slnx
& dotnet build Roslyn.slnx

# 推荐
Exec-Command "msbuild" "/v:m /m Roslyn.slnx"
Exec-DotNet "build Roslyn.slnx"
```

需要多次执行 `dotnet` 命令的脚本可以将 `dotnet` 命令存储在变量中，并改用 `Exec-Command`。

在调用 PowerShell 脚本后调用 `Test-LastExitCode` 以确保失败不被忽略。

```powershell
# 不推荐
& eng/make-bootstrap.ps1
Write-Host "Done with Bootstrap"

# 推荐
& eng/make-bootstrap.ps1
Test-LastExitCode
Write-Host "Done with Bootstrap"
```

与 `$null` 比较时，始终确保将 `$null` 放在运算符的左侧。对于非集合类型，这实际上不影响行为。但对于集合类型，将集合放在左侧会改变 `-ne` 和 `-eq` 的含义——它将比较集合内容，而不是检查是否为 `$null`。

```powershell
# 不推荐
if ($e -ne $null) { ... }
# 推荐
if ($null -ne $e) { ... }
```

### PowerShell 与 pwsh

Roslyn 基础设施应使用 `powershell` 执行而非 `pwsh`。通用 .NET 基础设施仍然使用 `Powershell` 并调用我们的脚本。在我们的脚本中改用 `pwsh` 会在源代码和统一构建中产生错误。在迁移到 `pwsh` 之前，我们的脚本需要继续使用 `Powershell`。

在非基础设施的情况下允许使用 `pwsh`，即不在 `eng/` 文件夹下且不期望由构建过程运行的脚本。这在需要跨平台运行脚本的场景中是必要的，例如：
- VS Code 辅助脚本
- AI 技能

### 支持 .cmd 文件

考虑添加一个调用 `.ps1` 文件的 `.cmd` 文件。这是为了支持未运行 PowerShell Shell 的用户。它还有助于确保脚本在本地运行方式与在 CI 中运行方式相同，通过控制用于调用脚本的 PowerShell 版本来实现。

test.cmd

```cmd
@echo off
set PSMODULEPATH=
powershell -noprofile -ExecutionPolicy Unrestricted -file "%~dp0\test.ps1" %*
```

注意：清除 `PSMODULEPATH` 是为了确保 pwsh Shell [不干扰][psmodulepath] PowerShell 的启动。

[psmodulepath]: https://github.com/PowerShell/PowerShell/discussions/24630
