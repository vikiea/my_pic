# 在Mac上编译可运行在Linux, Windows上的GO程序

## 编译运行在 amd64位 linux系统

```shell
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
```

## 编译运行在 amd64位 windows系统

```
CGO_ENABLED=0 GOOS=windows  go build 
```
