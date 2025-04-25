+++
date = '2025-04-23T09:12:19+08:00'
draft = true
title = '024Goland操作01_dlv'
+++

## 引子

## 手动更新Goland中较旧的delve

使用delve进行debug,但是在console页面显示警告信息 "Version of Delve is too old for Go version 1.23.0 (maximum supported version 1.24)"

解决方法:下载最新的delve，替换掉Goland之前携带的delve

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
which dlv
cp <which dlv results> /Applications/GoLand.app/Contents/plugins/go-plugin/lib/dlv/macarm/dlv
```

重启Goland，再次debug,console页面警告信息消失。方法来源于[StackOverflow](https://stackoverflow.com/questions/75585793/version-of-delve-is-too-old-for-go-version-1-20-0-maximum-supported-version-1-1)

debug闭包函数在走到闭包函数时会显示`internal debugger error: runtime error: invalid memory address or nil pointer dereference`

##

Vscode,Goland等IDE连接本机的终端是怎么连接的，为什么Goland第一次会用比较久的时间，而Vscode在Goland连接之后显示出的历史命令并不是之前的了？