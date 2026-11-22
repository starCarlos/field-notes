# 打开桌面客户端的 Code 就弹「未安装 WSL」：禁用 WSL 也关不掉

> 存档 ｜ 素材日期 2026-11-13 ～ 2026-11-16 ｜ 来源：个人实测记录
> 四问：AI 编得出吗？编不出——PATH 顺序加两个同名 `bash.exe` 的具体命中，得看机器。／有试错过程吗？有——设环境变量、禁用功能、想删 exe、最后 IFEO，一步步退。／有数字或可复现报错吗？有——`wsl -l` 返回未安装、弹窗 60 秒超时、监测返回 `NO_RUNNING_WSL_PROCESS`。／半年后还有人踩吗？会——System32 里那个 `bash.exe` 不会自己消失。

## 现象

打开桌面客户端的 code 功能，弹出一个 WSL 窗口。原文：

> 禁用 WSL 但仍弹窗，未安装适用于 Linux 的 Windows 子系统。可通过运行 "wsl.exe --install" 进行安装。……按任意键安装适用于 Linux 的 Windows 子系统。按 ESC 或 CTRL-C 取消。此提示将在 60 秒后超时。

问什么时候出现，用户答「都有」：打开 Codex、打开其他软件、开机后。

## 我以为是 X

以为是 WSL 没关干净。第一反应是 `wsl --shutdown`，再禁用功能、卸载，一套下来应该就安静了。

第二反应是：既然弹窗是某个 exe 弹的，把那个 exe 删了不就行了。

## 实际是 Y

WSL 本来就没装。`wsl -l` 返回的就是「未安装适用于 Linux 的 Windows 子系统」，功能没启用，也没有 Linux 发行版。

机器上有两个 `bash.exe`：

- `C:\Program Files\Git\usr\bin\bash.exe`：Git Bash，客户端要用的就是这个。
- `C:\Windows\System32\bash.exe`：WSL 的启动器，Windows 自带，运行它等于运行 `wsl`。

系统 PATH 里 `C:\Windows\system32` 排在 `C:\Program Files\Git\cmd` 前面，而且没设 `CLAUDE_CODE_GIT_BASH_PATH`。所以只要有程序按名字找 `bash`，命中的就是 System32 那个 WSL 启动器，于是去启动 WSL，弹出安装提示。

能触发弹窗的路径有三条，根子是同一个——`wsl.exe` / `bash.exe` 被调用，跟 WSL 功能开没开无关：

1. 客户端启动会话时按名字找 `bash`，命中 System32。
2. 其他程序直接调用 `wsl.exe`。
3. 手动运行 `wsl.exe` 或 `bash.exe`。

关键一点：**禁用 WSL 功能不会移除 `wsl.exe` 安装入口。** 别的程序继续调用它，照样弹。

## 怎么定位到的

先问弹窗时机，再起一个 60 秒的轮询，抓 `wsl` / `wslconfig` 进程和父进程链。结果返回 `NO_RUNNING_WSL_PROCESS`，exit_code 0，没抓到。

所以「客户端启动会话时误调用了 System32 的 bash」是按 PATH 顺序做的推断，**没有抓到弹窗那一刻的进程**。这点素材里反复标了未验证。

## 修法 / 结论

从轻到重三层。

**一、告诉客户端用 Git Bash**（治本，风险最小）：

```bash
setx CLAUDE_CODE_GIT_BASH_PATH "C:\Program Files\Git\usr\bin\bash.exe"
```

环境变量只对新启动的程序生效，要从托盘完全退出桌面版再重开，只关窗口不够。

**二、禁用 WSL 功能**（管理员 PowerShell）：

```powershell
wsl --shutdown
dism.exe /Online /Disable-Feature /FeatureName:Microsoft-Windows-Subsystem-Linux /NoRestart
```

不需要虚拟机平台的话再加 `VirtualMachinePlatform`。Store 装过的，去「设置 → 应用」卸载「Windows Subsystem for Linux」，重启。但这一步**只关功能，不解决弹窗**——安装入口还在。

**三、IFEO 拦截 `wsl.exe`**（变通，不是官方开关）：

```powershell
$key = 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\wsl.exe'
$dir = Join-Path $env:ProgramFiles 'WslLaunchBlock'
$script = Join-Path $dir 'block.vbs'
New-Item -ItemType Directory -Path $dir -Force | Out-Null
Set-Content -LiteralPath $script -Value 'WScript.Quit 1' -Encoding ASCII
$handler = '"{0}\System32\wscript.exe" //B //NoLogo "{1}"' -f $env:WINDIR, $script
New-Item -Path $key -Force | Out-Null
New-ItemProperty -Path $key -Name Debugger -PropertyType String -Value $handler | Out-Null
```

代价：所有调用 WSL 的软件都收到失败退出码，手动用、装、更新 WSL 也会被挡。恢复：

```powershell
$key = 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\wsl.exe'
Remove-ItemProperty -Path $key -Name Debugger
```

**删 exe 不推荐。** `C:\Windows\System32\wsl.exe` 通常受系统保护，管理员也未必删得掉；Windows 更新或修复可能恢复；调用方会改弹「找不到 wsl.exe」。想验证的话改名备份比删好恢复，但同样碰系统文件权限。

## 能拿走的 6 条

1. 弹「未安装 WSL」不等于 WSL 开着，可能只是有人按名字调了 System32 的 `bash.exe`。
2. 先分清两个 `bash.exe`：Git 的和 System32 的。
3. 禁用 WSL 功能 ≠ 移除 `wsl.exe` 安装入口。
4. 别用 `wsl --unregister` 治弹窗，它会删掉发行版数据。
5. 删系统 exe 是最差解，改名备份次之，IFEO 拦截可撤销。
6. 定位靠父进程链；抓不到就写「未验证」，别把推断当结论。

## 素材缺口

- 没抓到弹窗那一刻的父进程，具体哪个程序调用的，未证实。
- 设完环境变量后弹窗是否真消失，素材里没复测（原话「我还没确认弹窗是不是真的消失了」）。
- IFEO 方案没在机器上实际应用或验证。
- 桌面客户端的具体名字和版本未记录，素材只写「claude 桌面版」「codex」。

> 本文所有数字均来自个人实测记录，不附内部坐标。
