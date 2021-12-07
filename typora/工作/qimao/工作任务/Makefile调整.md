# 工作任务

- 迁移命令行依赖包至Makefile

## 问题

- github.com/googleapis/googleapis无版本号，每天拉取版本号都会变

- 有的只在生成proto文件或其他命令行命令的时候有用到，go.mod里会提示unused，执行`go mod tidy` 时会被清理掉![image-20211117140109401](/Users/qinfuling/Library/Application Support/typora-user-images/image-20211117140109401.png)

- `github.com/iancoleman/strcase@v0.2.0`,`github.com/lyft/protoc-gen-star@v0.6.0`是`github.com/envoyproxy/protoc-gen-validate`的依赖项

- `golang.org/x/tools/cmd/stringer`不知道之前的版本号，取最新版本号0.1.7，问题不大.且全局只有一个位置用到，用于初始化事件名称，是否需要？

- `github.com/tklauser/go-sysconf`从v0.3.7升级到了v0.3.9

- github.com/grpc-ecosystem/grpc-gateway/runtime@v1.16.0有什么用

  