### 背景
前段时间想写个脚本简化开游戏的过程，具体什么游戏就不说了。其中最主要的步骤是需要找到游戏对应进程的某些句柄，然后删除指定关键字。  
消耗了大量精力后，通过某位[好兄弟](https://github.com/kihlh)的帮助，终于找到了核心工具handle，这是微软提供的一个非常底层的工具，比起tasklist有着更详情的内容。  
有了工具，思路就很简单了，搜索进程，根据pid再次搜索句柄，删除句柄，OK
```
const child_process = require('child_process');

child_process.exec("tasklist | findstr xxx")
  // ...
  child_process.exec("handle -a -vt -p xxx")
    // ...
    child_process.exec("handle -c -p xxx")
```
但是有个很麻烦的问题，这个handle指令需要管理员模式下才能运行，正常运行很多内核信息不展示！  
虽然通过github找到了sudo-prompt插件，但是具体原理我也很好奇，必须研究一下。

### 原理
想通过在cmd指令中加某些参数来达到管理员模式运行是不可行的，只能另辟蹊径。在翻阅windows官方文档时，有一个指令刚好有管理员权限参数：Start-Process。在这个指令后面加上-Verb RunAs可以用管理员模式运行某个文件，比如Start-Process -FilePath "xxx" -Verb RunAs。  
这时候可能会想，直接将这里的文件路径改成上面的JS文件行不行呢，答案是不可以，bat脚本可不认识什么node。这个指令一般情况只能作用于文件，所以现在的思路  
- 一个bat脚本作为启动文件，执行Start-Process
- 一个bat脚本作为实际指令执行文件，路径作为第一个脚本的参数 

handle指令执行后，新的问题就是如何获取返回的内容，刚好bat脚本有个语法
call "xxx.bat" > "path1" 2> "path2"  
这里的path会分别写入xxx.bat指令的正常输出与错误时的内容，而对于一条指令的执行状态，也有一个全局参数%ERRORLEVEL%来表示，返回为0表示成功，1表示失败。  
综上所述，整个执行过程如下：  
1. 启动指令
```
powershell.exe Start-Process -FilePath "exec.cmd" -WindowStyle hidden -Verb runAs
```
2. exec.cmd
```
call "command.cmd" > "out" 2> "err"
(echo %ERRORLEVEL%) > "stat"
```
3. command.cmd
```
handle -a -vt -p xxx
```
通过启动cmd，可以将输出内容写入out、err、stat三个文件下，通过轮询stat文件的内容，得到对应的状态码，如果是0，读取out的内容将其作为输出，完成了管理员模式下执行指令的需求。
```
// 伪代码
fs.writeFileSync("exec.cmd")
fs.writeFileSync("command.cmd")
child_process.exec("powershell.exe Start-Process...").then
  fs.readFileSync("stat").then
    if (code === 0) return fs.readFileSync("out")
```
完整的代码可见[github](https://github.com/pflhm2005/Interest/blob/master/sudo.js)