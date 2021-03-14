这里是面向**OpenGL版本小于3.2**又想运行**21w10a+**的Minecraft玩家的页面。

有关于这个程序：
1. 能进入世界并能正确渲染
2. 不需要OpenGL3.2，最低限制为2.1

使用警告：
1. 你**不能**用鼠标点击物品栏（创造模式的方块选取物品栏除外）。在你点击物品栏的那一刻游戏会**瞬间崩溃**。所以你无法在这里正常生存，只能用创造/旁观查看。如果你要从创造模式中拿出方块，请直接丢出等待物品被吸回，一定不能直接将物品放到自身背包，否则游戏会崩溃。
2. 不要使用材质包，即使是老版本的材质包规范
3. 因为这个JAR文件是被修改的，在这里发现的任何bug都无法汇报到<https://bugs.mojang.com>，mod将会以`invalid`打回你的报告
4. 这个页面会在每次新快照发布那一周的周日更新最新的文件

<h2>下载最新快照的降级修改版</h2>

<https://nickid2018.github.io/resources/20210314/21w10a.jar>

<h2>生成程序</h2>
<https://nickid2018.github.io/resources/20210314/demotion.jar>

命令行： [--output < file >] < mapping > < mc_jar >

完整命令行： java -jar demotion.jar [--output < file >] < mapping > < mc_jar >

--output ：可选项，输出文件位置

mapping ：混淆映射表位置，混淆映射表下载在下方

mc_jar ：MC主文件JAR位置

请确保运行此程序时你的java可被访问到（比如设置了环境变量）或在同一文件夹

下载： <https://nickid2018.github.io/resources/20210314/demotion.jar>

混淆映射表：

21w10a - <https://launcher.mojang.com/v1/objects/5f15280e2823d988faa4c3b83db33d11f82f6afd/client.txt>