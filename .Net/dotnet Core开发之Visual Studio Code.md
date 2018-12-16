# 使用Visual Studio Code开发.NET Core

在本文中，我将带着大家一步一步的通过图文的形式来演示如何在Visual Studio Code中进行.NET Core程序的开发，测试以及调试。尽管Visual Studio Code的部分功能还达不到Visual Studio的水平，但它实际上已经足够强大来满足我们的日常开发。而且其轻量化，插件化以及跨平台的特性则是VS所不具备的。而且Visual Studio Code还可以通过社区来创建一系列的扩展来增强其功能，且社区已经足够活跃。我们可以期待更多很酷的扩展和功能来增强VS Code，这将使在这个轻量级，跨平台编辑器中的开发.NET Core应用程序更加流畅和有趣。

## 安装准备

1. 安装.NET Core SDK
2. 安装Visual Studio Code
3. 在Visual Studio Code 中安装C# 扩展以便让Visual Studio Code 支持C#的开发。为了安装c#的扩展，你可以通过Visual Studio Code左侧工具栏中的Extensions图标或使用键盘快捷键Ctrl + Shift + X打开Extensions视图。在搜索框中搜索C＃并从列表中安装扩展程序。重点安装的扩展：

  C#语言扩展: https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp
  这个是使用VSCode编写C#代码必须的，安装之后在默认打开.cs文件时 还会自动下载调试器等（不过过程可能比较慢，在墙外的原因）

  [C# XML注释]: https://marketplace.visualstudio.com/items?itemName=k--kato.docomment
  这个可以插件可以快速的帮你添加注释，选择安装吧

  [C# Extensions]: https://marketplace.visualstudio.com/items?itemName=jchannon.csharpextensions
  这个插件，强烈推荐，可以帮你在建立文件的时候初始化文件内容包括对应的命名空间等

  还有一些其他辅助类的，比如EditorConfig,Guildes,One Dark Theme,Project Manager ,Setting Sync等。

## 使用Visual Studio Code开发基本的.NET Core程序

VS code 打开某个目录，作为工作目录。

接下来我们使用 `dotnet new console --name DotNetCoreSample` 命令来在这个打开的终端里面创建一个基础的控制台程序并进行restore。

dotnet new sln # 生成空的sln文件

```c#
dotnet new xunit -n MathOperationTests  
dotnet add MathOperationTests\MathOperationTests.csproj reference MathOperations\MathOperations.csproj  
dotnet sln SimpleCalculator.sln add MathOperationTests\MathOperationTests.csproj
```


## 参考：

[使用Visual Studio Code开发.NET Core看这篇就够了](https://www.cnblogs.com/yilezhu/p/9926078.html)
[VsCode编写和调试.NET Core](https://www.cnblogs.com/Leo_wl/p/6732242.html)
