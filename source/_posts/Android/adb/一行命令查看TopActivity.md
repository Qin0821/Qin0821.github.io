以往我都是运行自己写的CurrentActivity app来实时查看当前Activity，如今换电脑了，暂时没有该工具项目代码，上网找一个又嫌麻烦和不安全，偶然发现可以通过命令来查询
# 查询命令
 adb shell dumpsys activity top | grep ACTIVITY
 # 可能遇见的问题
 * 找不到 adb
 原因是没有把adb配置到环境变量中，把adb.exe的path配置即可（可以通过everyThing来快速查找adb.exe的位置）
 * win 11配置环境变量
 设置 - 系统 - 系统信息 - 高级系统设置 - 高级 - 环境变量
 如新建一个变量：adb，值：C:\Users\liuqin\AppData\Local\Android\Sdk\platform-tools
 然后在path中新增一栏：%adb%
 然后重新打开命令行输入adb，若不提示找不到adb就表示成功啦
 * 找不到grep
 需要[下载](http://gnuwin32.sourceforge.net/packages/grep.htm)，选择`Complete package，except source`，下载安装即可，记住安装目录，待会在后面加个`\bin`新增到path即可
 如：C:\Program Files (x86)\GnuWin32\bin