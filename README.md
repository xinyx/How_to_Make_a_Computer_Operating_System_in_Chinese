如何做一个计算机操作系统
====================

这是一本介绍用C/C++从零开始写一个操作系统的在线书籍

**注意**:这个仓库是我以前课程的改作,写于几年前.[是我高中时最开始做的几个项目之一](https://github.com/SamyPesse/devos).我仍然在重构一些部分.最开始这门课是法语版,并且我的母语并非英语. 我将在闲暇时间持续改进这门课程.

**书籍**:在线版本位于[http://samypesse.gitbooks.io/how-to-create-an-operating-system/](http://samypesse.gitbooks.io/how-to-create-an-operating-system/) (PDF,Mobi和 ePub).它是用 [GitBook](https://www.gitbook.io)生成的.

**源代码**:所有系统源码位于[src](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/tree/master/src)目录.每一步都包含指向不同相关文件的链接.

**贡献**:本课程是开放的,可以自由地用issues标记错误,或者用pull-request直接修改错误.

**问题**:利用issues自由提问. 请不要使用邮件.

你可以在twitter[@SamyPesse](https://twitter.com/SamyPesse)上找到我.也可以在[Flattr](https://flattr.com/profile/samy.pesse)或[Gittip](https://www.gittip.com/SamyPesse/)上顶我.

###我们在构建一个什么样的系统?

目标是用C++写一个非常简单的基于UNIX的操作系统,而不仅仅是"概念模型".这个系统能够启动,打开一个用户空间shell,并且可扩展.

![Screen](./preview.png)