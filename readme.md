# clock-tweak

Windows 系统时间修改工具,纯 Go + Win32 API 实现,单文件 EXE(约 2.5MB),零外部依赖。

## 项目背景

一些软件的证书、license 存在有效期,过期后需要把系统时间改回有效期内才能继续使用。手动修改操作复杂麻烦,改完还要记得恢复对时。clock-tweak 把这套流程做成一键操作:点「设置日期」自动修改系统时间并处理好时间同步相关设置,用完点「恢复时间」一键恢复网络对时。

## 功能特性

- 修改系统日期,自动停用 NTP 同步与 w32time 服务,程序退出、系统重启后假时间依然保持
- 一键恢复:还原全部同步配置并立即 NTP 网络对时
- 纯 Go 实现,仅调用 Win32 API,无 CGO、无第三方依赖
- 原生 Win32 控件(日期选择器、按钮、日志区),按系统 DPI 自动缩放
- 自动请求管理员权限,双击即可使用
- 耗时操作后台执行,界面不卡顿,带时间戳日志

## 使用方法

1. 双击运行 `clock-tweak.exe`,在 UAC 弹窗中确认
2. 在日期选择器中选择目标日期
3. 点击 **「设置日期」**,修改系统日期(时分秒保持不变)
4. 需要恢复时点击 **「恢复时间」**,还原同步配置并立即对时

> 假时间会一直保持,直到显式点击「恢复时间」。

## 构建

要求:Windows + Go 1.21+。

一键构建(安装 go-winres → 按 `winres\winres.json` 生成资源,嵌入 `requireAdministrator` 清单、版本信息、DPI 声明 → 编译为无控制台的单文件 EXE):

```bat
build.bat
```

或直接编译(无版本信息等资源):

```bash
go build -ldflags="-s -w -H windowsgui" -o clock-tweak.exe .
```

## 技术实现

| 组件 | 实现方式 |
|------|---------|
| GUI | Win32 API(user32.dll / gdi32.dll),微软雅黑字体 |
| 日期选择器 | `SysDateTimePick32` 原生控件 |
| 设置时间 | `SetSystemTime`(kernel32.dll),改前先禁用 NtpClient、安全时间种子与 w32time 服务 |
| 恢复时间 | 还原同步配置后执行 `w32tm /resync /force` |
| 同步控制 | `reg.exe` 写注册表 + `sc` / `net` 控制 w32time 服务 |
| 提权 | 令牌检测 + `ShellExecuteW "runas"` 自我重启;构建脚本另嵌入 `requireAdministrator` 清单 |
| UI 线程模型 | 消息循环 + goroutine 执行耗时命令,`WM_APP` 消息回传结果 |

## 项目结构

```
├── main.go            # 全部实现:GUI、提权、时间同步控制、NTP 恢复
├── build.bat          # 一键构建脚本
├── winres/winres.json # EXE 资源清单(UAC 级别、版本信息、DPI 声明)
└── go.mod
```

## 注意事项

- 仅支持 Windows,需管理员权限
- 假时间生效期间系统时间同步被禁用,HTTPS 证书校验、软件许可证等可能受影响,建议用完及时「恢复时间」
- 「恢复时间」需要网络能访问 NTP 时间服务器
