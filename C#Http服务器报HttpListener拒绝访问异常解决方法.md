# [WIN7中HttpListener拒绝访问异常解决方法](https://blog.csdn.net/swordsf/article/details/77413738)


## 问题:Win7下在尝试搭建简单http服务器的时候，执行httpListener.Start();报错HttpListener拒绝访问异常

## 代码如下：

  HttpListener httpListener = new HttpListener();//创建服务器监听

  httpListener.Prefixes.Add("http://+:9000/");//配置监听地址。+代表本机可能的IP如localhost、127.0.0.1、192.168.199.X(本机IP)等；

  httpListener.Start(); //开启对指定URL和端口的监听,开始处理客户端输入请求。

## 解决方法：

以管理员CMD命令行执行：


①先删除可能存在的错误urlacl，这里的*号代指localhost、127.0.0.1、192.168.199.X本地地址和+号等。

命令：netsh http delete urlacl url=http://+:9000/ 

②将上面删除的*号地址重新加进url，user选择所有人

命令：netsh http add urlacl url=http://+:9000/  user=Everyone

③CMD配置防火墙

netsh advfirewall firewall Add rule name=\"命令行Web访问9000\" dir=in protocol=tcp localport=9000 action=allow


经过如上设置服务端就可以以httpListener.Prefixes.Add("http://+:9000/");监听地址开启监听。客户端可以通过访问服务端9000端口。服务端本机也可以在浏览器中以localhost和127.0.0.1访问自身http服务器。
