# AGENTS.md

## 项目概述

clock-tweak：Windows 系统时间修改工具。纯 Go + Win32 API（syscall 手工绑定），无 CGO、无第三方依赖，单文件 EXE。用于把系统时间改到 license 有效期内，用完一键恢复 NTP 网络对时。

- 仅支持 Windows，需管理员权限（运行时令牌检测 + `runas` 自我重启；构建时嵌入 `requireAdministrator` 清单）。
- 代码注释、UI 文案、日志、提交信息均使用简体中文，保持一致。

## 构建与验证

```bat
build.bat    :: 一键构建：自动安装 go-winres → 按 winres\winres.json 生成资源（UAC 清单/版本信息/DPI 声明）→ 编译
```

或直接编译（无资源）：

```bash
go build -ldflags="-s -w -H windowsgui" -o clock-tweak.exe .
```

构建参数固定：`GOOS=windows GOARCH=amd64 CGO_ENABLED=0`，`-H windowsgui` 去掉控制台窗口。

- 无测试、无 lint 配置。验证方式：`go vet ./...`、`go build` 通过，再手动运行 EXE 冒烟。
- 产物 `clock-tweak.exe` 与 `*.syso` 已在 .gitignore 中，不要提交。

## 架构

全部实现在 `main.go`（单文件，按注释分区）：

- **GUI**：raw Win32（user32/gdi32），`SysDateTimePick32` 日期选择器，微软雅黑，DPI 用 `scale()` 按 96 基准换算，布局集中在 `layoutControls`（WM_SIZE 触发）。
- **线程模型**：`main` 里 `runtime.LockOSThread()` 跑消息循环；`wndProc` 必须保持轻量，耗时命令（reg/sc/net/w32tm）一律放 goroutine，结果经 `asyncPostResult` 用 `WM_APP+1`（WM_ASYNC_RESULT）回传 UI 线程（消息体走 mutex 保护的 `asyncResultMsg/Code` 全局变量）。
- **并发护栏**：`tryBeginOp()/endOp()`（opMu + opInProgress）防止重复操作，按钮禁用只是辅助。

## 关键约束（改代码前必读）

- **`cleanup()` 退出时绝不能恢复 w32time**：假时间必须在程序退出、系统重启后继续生效，恢复只能由「恢复时间」按钮显式触发。
- **时间回跳问题**（曾出过 bug）：仅禁用 NTP 不够，`disableTimeSync` 必须按顺序做四件事——NtpClient `Enabled=0`、W32Time\Config `UtilizeSslTimeData=0`（Secure Time Seeding 会用 TLS 缓存时间强行纠偏）、`sc config w32time start= disabled`、停服务并轮询确认 STOPPED。调整顺序/删步骤会导致系统时间被自动拉回。
- **失败回滚**：`setSystemTime` 中 `SetSystemTime` 调用失败要 `restoreSyncConfigBestEffort()` 恢复同步配置（此时系统时间未被改动）。
- **命令超时不能随意调短**：`w32tm /resync /force` 在服务忙时阻塞很久（60s），服务启停 30s，reg/sc 10s。
- 注册表写、服务控制、对时全部走 `reg.exe` / `sc` / `net` / `w32tm` 子进程（`runCmdWithTimeout`，`HideWindow: true`），不用别的库。

## 版本资源

改版本号或 UAC/DPI 声明时编辑 `winres/winres.json`（当前 `requireAdministrator` + `permonitorv2`），重新跑 `build.bat` 生效。
