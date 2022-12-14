# MyChineseCheckers 设计文档

Author：张诗颖（经12-计18，2021011056，shiying-21@mails.tsinghua.edu.cn）

 *本设计文档仅供程设小学期Qt+Socket+多线程部分大作业使用*



### MyChineseCheckers项目设计综述

MyCheckers是基于Qt开发的网络对战跳棋游戏，主要分为**客户端（即client）** 和 **服务器端（即server）** 两个部分。其中，**客户端**主要负责和玩家的交互，包括发送和接收网络服务器消息、游戏中的走棋等。**服务器端**协调客户端的网络数据同步，包括计时、评判胜负等。

##### 游戏玩法

+ 本游戏为双人联网跳棋游戏，进行游戏时需开启且仅开启两个`client`端可执行文件（`MyCheckers_v2.exe`）和一个`server`端可执行文件（`My_Checkers_Server.exe`）。注意，进行游戏时不能破坏`MyCheckers_release_final_version`的层级结构
+ 进入游戏界面后`client`端需要接入服务器。点击菜单栏页面`Connect->Connect to Server`，根据提示输入IP（在同一电脑下使用默认的`IP=127.0.0.1`）和端口（端口由服务器设置），即可完成连接
+ 选择棋子游戏后可以点击`Now start your Game`，两位玩家都点击开始（且选择棋子颜色不同）后开始游戏
+ 游戏期间双方交替走子，有如下玩法细节：
  + 己方时间可以拖动己方棋子，不能拖动对方棋子。鼠标位于合法棋子上方时会呈现“抓取”形态，提示玩家可以点击拖动。
  + 走子时点击鼠标选中棋子，拖动鼠标到终点位置并松开鼠标完成“下一步棋”操作。若终点位置不合法棋子回复起点位置，还有剩余时间时可以继续选择棋子走子。
  + 游戏过程中可以按`Pause`暂停键暂停游戏，该按键触发后`Server`会自动暂停双方计时，`Pause`按钮变为写有`Continue`字样的按钮，期间任意一方可以随时选择`Continue`游戏，解除游戏暂停。
  + 一方认输且认输合法（在20轮及以后可以认输）时，收到判负弹窗，另一方同时收到判胜弹窗，且`Connect to Server`的提示框中会提示对方认输。
  + 在游戏任意期间内若一方玩家掉线（主动`disconnect`或者关闭退出游戏），对方都会收到提示并返回开始界面。
  + 在游戏期间可以拖动窗口，棋盘会随窗口拖动而缩放到适应性尺寸。
  + 其它技术细节与作业要求实现的一致
+ 棋盘界面之外的其它功能性配件：
  + `Pause`按钮，已在上述“游戏期间玩法细节”阐释
  + `Help`按钮，提示游戏规则
  + `Quit`按钮，点击后己方退出游戏，对方收到己方掉线信息后回到主界面
  + 菜单栏处`Connect`下有`Create the connection`和`Connect to Server`两个选项，前者由于`client`端默认没有`Server/Host`功能因而无实质性作用，后者可以控制客户端和服务端的连接情况，收到服务端返回的信息并在对话框中显示
  + 菜单栏处`Play`下有`Start`和`Admit defeat`两个选项，前者将回到开始界面准备进入新的游戏，后者已在上述“游戏期间玩法细节”部分阐释
  + 左下部分会显示鼠标当前的屏幕位置坐标以及内部构建坐标系的坐标，主要方便开发时调试。

##### 源代码结构

以下为网络学堂提交文件夹`MyChineseCheckers`的文件夹主要层级树状图示：

```
|-images //该文件夹包括了游戏界面所用的棋盘和棋子素材，此处不再展开
|-My_Checkers_Server //服务器端源代码
    |-main.cpp
    |-mainwindow(.h/.cpp/.ui)  
|-MyCheckers_v2 //客户端源代码
    |-main.cpp
    |-mainwindow(.h/.cpp/.ui)
    |-connectserver(.h/.cpp/.ui)
    |-chess.h
    |-admitdefeatdialog(.h/.cpp/.ui)
    |-createconnection(.h/.cpp/.ui)
    |-playdialog(.h/.cpp/.ui)
    |-helpdialog(.h/.cpp/.ui)
    |-enemywindialog(.h/.cpp/.ui)
    |-mewindialog(.h/.cpp/.ui)
|-MyCheckers_release_final_version //该文件夹单独下载到本地可以直接获取游戏的release版本
    |-images //运行游戏程序时必须将其至于上一级目录下
    |-client
        |-MyCheckers_v2.exe //客户端可执行文件
    |-server
        |-My_Checkers_Server.exe //服务器端可执行文件
```

接下来将就此详细介绍项目设计。






### 客户端功能实现

#### 走棋功能实现

##### 1. chess类

棋盘和棋子都是一个`chess`结构体的容器，即`vector<chess>`类型。该`chess`结构体记录有位置坐标、（待下棋子）棋子状态/棋子序号/所处棋盘对应序号，（棋盘棋坑）的序号/是否存放棋子及棋子状态等信息

##### 2. paintEvent()函数

主要由`paintEventChessboard()`和`paintEventChessesPlayed()`两个函数构成。顾名思义，前者负责棋盘绘制，后者负责棋子绘制。

##### 3. 鼠标跟踪

由`mouseMoveEvent()`， `mousePressEvent()`和`mouseReleaseEvent()`共同完成棋子状态的记录。判断走棋合法与否主要由`mouseReleaseEvent()`调用`disobeyRules()`实现。后者是基于`bfs`（广度优先算法）搜索算法实现的。

##### 4. 胜负判断

每一步走棋之后双方的`client`端都会自行判断基本胜负条件，包括但不限于：① 己方棋子是否已全部进入对方营区，② 指定回合是否己方营区至少已经走出若干棋子，③ 三次超时判负。后续客户端功能实现部分将介绍客户端主要负责的胜负判断情况（认输，掉线等）

##### 5. 算法和接口

为了使得代码更加简洁，本项目设计了多个常用函数接口，包括屏幕像素坐标和内置坐标的转换函数，坐标与棋子/棋盘序号对应函数，己方/敌方的己方/敌方棋子计数函数等。方便反复调用和直观理解代码。



#### 客户端UI界面

##### 1. 棋子与棋盘，背景

棋子与棋盘，背景的素材都是通过PPT手工绘制的。棋盘的棋坑为白色，在黑色的背景下更加显眼。棋子提供六种不同的颜色选择，对战双方可以自行选择（但不能重复，否则后选的一方会接到`Server`的提示要求重新选择）。所有的棋子制作都采用渐变颜色效果，富有立体感。棋子的大小略大于棋盘棋坑的大小，符合常规实践操作。棋子、棋盘、背景三者都支持随窗口伸缩自动适应大小。

所有的设计素材都位于`images`文件夹中。

##### 2. 按钮设计

按钮主要包括了`Select A Color`，`Now Start Your Game!`等。按钮皆在按下和松开鼠标时有不同的特效表现，用来呈现更好的真实性和动态感。

##### 3. 鼠标形态设计

鼠标位于待点击的按钮或待挪动的棋子之上时会呈现“抓取”形态，其他情况下为“光标”模式，提示玩家按键或走棋。

##### 4. 对话框类交互设计

`MainWindow`类中采用不同的指针管理不同的`Dialog`派生类。各`Dialog`派生类与`MainWindow`类的交互主要通过信号与槽机制实现。具体为`Dialog`会发出信号，由`MainWindow`主窗口执行槽函数。

##### 5. 对战双方信息提示窗口

左侧信息栏显示双方实战信息（实践、回合）、所持棋子颜色信息等






### 服务器端功能实现

#### 信息同步和客户端交互

服务端（`Server`）主要承担了建立和断开连接、同步时钟、玩家走棋数据和其他行为交互等重要功能。

##### 1. 建立和断开连接

使用`Qt`自带的`TCPSocket`派生类进行。服务器会为先后连接的两个`Client`自动分配对应的`socket`指针记录地址。且精细的`onNewConnection()`和`onDisconnected()`槽函数设计使得客户端任何突如其来的连接和断开连接操作都能被正确处理，并影响服务器对另一方客户端的即时判断。

包括但不限于实现了：若出现双方都选择相同颜色棋子的情况，服务器将指定后选择的客户端重新选择颜色；一方断开连接时另一方也会收到提示

##### 2. 同步时钟

游戏以`Server`内置时钟的方式同步双方时间，每过一秒`Server`会将信息发给双方`Client`更新时间或轮次，保障双方进度完全同步。

##### 3. 数据处理

玩家的走棋数据采用`QJson`进行打包并通过`Server`处理传递给另一个玩家。对于非常规走子的其它行为（例如：认输，指定回合离开本营棋子数不达标判负，累计三次超时，暂停，结束暂停等）也都有特异性识别和处理。



#### `client`和`server`的ui设计

##### 1. 信息显示框

除了规定的`ip`和`port`设计外，还采用了信息显示框（`textEdit`）用来显示和提示服务器端和客户端的信息交互情况。该功能既提示了客户端服务器端发来的信息从而对产生情况有更精细化的理解，也方便在开发时观察和调试。同时信息显示框的内容输出格式和布局符合美观设计。

##### 2. 提示IP和端口

`Server`的ui设计包括调用自身`getLocalIP()`函数获取自身`Host`的IP，方便客户端连接。`IP`默认输入`127.0.0.1`，是因为这通常是同一台电脑及其显示器上双人对战时输入的`IP`（此时`IP`输入主机`IP`无效）



具体设计界面、游戏过程中的设计信息详见可执行文件。
