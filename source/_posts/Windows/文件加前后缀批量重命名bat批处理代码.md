文件名加前缀以及文件后缀批量重命名在电脑操作中经常会用到，一个两个文件还好说，要是多个文件，一个一个的改，不知要改到何年何月，最简单又不占资源的方法就是用批处理代码来实现。

1.文件名加前缀的批处理代码

@echo off 
for /f "delims=" %%i in ('dir /a-d/b/s *.html') do ( 
if not "%%i"==%0 ren "%%i" "20221023%%~nxi") 
echo 命名完毕 
pause 
【应用说明】

这是文件名中加前缀的实例，其中“*.html”代表只对后缀为html的文件进行前缀修改，如果想对所有后缀的文件进行前缀加入，只需把“*.html”改为“*.*”，这里的20221023%%~nxi代表在原有文件名上加入20221023时间前缀，可以更改为其他想加入的前缀。

【注意】

该批处理代码会连同子文件夹下面的html文件一起修改。

2.文件后缀批量更改的批处理代码

@echo off  
set now=html 
set after=txt  
setlocal enabledelayedexpansion  
for /f "delims=" %%i in ('dir /s /b *.%now%') do (  
set a=%%~fi& set b=%%~ni  
ren "!a!" "!b!.%after%")  
exit  
【应用说明】

这是一个批量将后缀为html文件批量修改为txt文件的批处理代码，其中：

now=html

代表修改前文件的后缀为html。

after=txt

代表修改后文件的后缀为txt。

【注意】

该批处理代码会连同子文件夹下面的html文件的后缀一起修改。