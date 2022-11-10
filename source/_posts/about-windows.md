---
title: about-windows
date: 2022-11-10 19:24:09
tags:
- Uncategorized
---

## 背景
目前 `windows` 还是使用的比较多的，在日常使用中会重复遇到一些问题，但总是解决后过段时间就忘了，所以在此记录下来。

## 环境变量
需要知道的是，`cmd` 和 `PowerShell` 打印某个环境变量的方式是不同的
- cmd
```cmd
echo %Path%
```
- PowerShell
```PowerShell
$env:path
```
### 刷新环境变量
[choco](https://github.com/chocolatey/choco) 内置的 `refreshenv` 可以帮助我们在不重启终端的情况下方便的刷新环境变量。

需要注意的是虽然旧版命令行可以通过重启终端的手段刷新环境变量，但在 [win11 默认的 terminal](https://github.com/microsoft/terminal) 中由于存在 [某些问题](https://github.com/microsoft/terminal/issues/1125) 导致无法确保重启终端能够刷新环境变量。在这种时候，使用 `choco` 中的 `refreshenv` 几乎就是必须的了。

- cmd
```cmd
refreshenv
```

- 可能会发现在 `PowerShell` 中使用该命令并没有生效，这是因为还要进行 [some additional work](https://docs.chocolatey.org/en-us/troubleshooting#why-does-choco-tab-not-work-for-me)