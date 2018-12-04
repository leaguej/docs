# [解决$ go get google.golang.org/grpc上的包被墙的问题](https://www.cnblogs.com/hsnblog/p/9608934.html)

今天get grpc包的时候

```bash
$ go get google.golang.org/grpc
```

发现拉不下来被墙了，在github.com上搜索grpc，clone到工程目录中，运行命令

```bash
go install google.golang.org/grpc
```

拿到了一些丢失的依赖包，比如：

![img](https://images2018.cnblogs.com/blog/797701/201809/797701-20180908130726658-74400198.png)

进入https://github.com/golang仓库找到对应的包，git clone下来，放到指定的目录中，比如上图缺少的golang.org/x/net/http2包，在github上把net包clone下来，如下：

```bash
git clone https://github.com/golang/net.git $GOPATH/src/golang.org/x/net
git clone https://github.com/golang/text.git $GOPATH/src/golang.org/x/text

git clone https://github.com/golang/net.git %GOPATH%/src/golang.org/x/net
git clone https://github.com/golang/text.git %GOPATH%/src/golang.org/x/text
git clone https://github.com/grpc/grpc-go.git %GOPATH%/src/google.golang.org/grpc
git clone https://github.com/google/go-genproto.git %GOPATH%/src/google.golang.org/genproto
```

其他包也如此操作，全部完成后，再运行

```bash
go install google.golang.org/grpc
```

成功，问题解决。

另参考：[go如何下载golang.org的包](https://www.jianshu.com/p/096c5c253f75)

