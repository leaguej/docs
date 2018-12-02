# GO 语言的有用资源

### [Go 资源大全中文版](https://github.com/jobbole/awesome-go-cn)

我想很多程序员应该记得 GitHub 上有一个 Awesome - XXX 系列的资源整理。本列表翻译自awesome-Go Awesome 系列虽然挺全，但基本只对收录的资源做了极为简要的介绍，如果有更详细的中文介绍，对相应开发者的帮助会更大。这也是我们发起这个开源项目的初衷。

### [lorca: Build cross-platform modern desktop apps in Go + HTML5](https://github.com/zserge/lorca/)

A very small library to build modern HTML5 desktop apps in Go. It uses Chrome browser as a UI layer. Unlike Electron it doesn't bundle Chrome into the app package, but rather reuses the one that is already installed. Lorca establishes a connection to the browser window and allows calling Go code from the UI and manipulating UI from Go in a seamless manner. 

### [athens: A Go module datastore and proxy https://docs.gomods.io](https://github.com/gomods/athens)

a proxy server for the Go Modules download API.

github上还有另外一个go module proxy的实现：[goproxy](https://github.com/goproxyio/goproxy)。该项目目前看仅是一个public module proxy，并未提供对private repo中module获取的支持。

不过该项目提供的global proxy: https://goproxy.io/ 却是可以在国内使用的，并且速度还很快！Gopher们只需将该proxy配置到GOPROXY中即可：

```bash
export GOPROXY=https://goproxy.io
```

相关文章：[Hello，Go module proxy](https://tonybai.com/2018/11/26/hello-go-module-proxy/)

### [GoFrame, Go 应用开发框架](https://gitee.com/johng/gf)

GF(Go Frame)是一款模块化、松耦合、轻量级、高性能的Go应用开发框架。支持热重启、热更新、多域名、多端口、多服务、HTTP/HTTPS、动态路由等特性 ，并提供了Web服务开发的系列核心组件，如：Router、Cookie、Session、服务注册、配置管理、模板引擎、数据校验、分页管理、数据库ORM等等等等， 并且提供了数十个内置核心开发模块集，如：缓存、日志、时间、命令行、二进制、文件锁、内存锁、对象池、连接池、数据编码、进程管理、进程通信、文件监控、定时任务、TCP/UDP组件、 并发安全容器等 [介绍链接](https://www.oschina.net/news/102125/goframe-1-2-11-released)

### [TeaWeb - 可视化智能Web服务](https://gitee.com/liuxiangchao/build)

TeaWeb是一款集静态资源、缓存、代理、统计、监控于一体的可视化智能WebServer。
TeaWeb使用Go语言实现，在高可定制化前提下，保证高性能、高并发。

