如何制作一个操作系统
=======================================

这是一部关于使用C/C++从无到有编写一个计算机操作系统的在线书籍。

**注意**：这个 repo 是我以前的课程的重制。它是我几年前[上高中时候写的第一个 project ](https://github.com/SamyPesse/devos)，我仍然在重构其中的一些部分。原课程是法语教授的，而且我的母语也并非英语。我打算在业余时间里继续将它改进。

**书籍**：[这里](http://samypesse.github.io/How-to-Make-a-Computer-Operating-System/)是一个在线版本，使用GitBook生成。

**源码**：所有系统源码都放在`src`文件夹内。每一步会包含其相关文件的链接。

**贡献**：这个课程是开放的，如果发现错误，可随时作出标记并发起 issue，或者直接通过 pull-request 来改正它。

**问题**：同样地，可以随意发起 issue 来提问。请不要给我发邮件。

**翻译**：这里有一个[中文翻译版本](https://github.com/Kroderia/How-to-Make-a-Computer-Operating-System-in-Chinese)

你可以在 Twitter [@SamyPesse](https://twitter.com/SamyPesse)上关注我，[Flattr](https://flattr.com/profile/samy.pesse)或[Gittip](https://www.gittip.com/SamyPesse/)上进行资助.

### 我们在制作怎样的操作系统?

目标是使用 C++ 来构建一个非常简单，基于 UNIX 的操作系统，而不只是一个“概念验证”。这个操作系统可以启动，运行一个 userland shell，并且可以扩展。

[![Screen](https://raw.github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/preview.png)](https://raw.github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/preview.png)
