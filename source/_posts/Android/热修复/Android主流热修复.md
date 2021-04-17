近几年以来，Android开发领域里对热修复技术的讨论和分享越来越多，同时也出现了一些不同的解决方案，如QQ空间补丁方案、阿里AndFix、微信Tinker、美团Robust，它们在原理各有不同，适用场景各异，到底采用哪种方案，通过介绍QQ空间补丁、Tinker、Rubust以及基于AndFix的阿里百川HotFix技术的原理分析和横向比较。

#### QQ空间超级补丁技术

超级补丁技术基于DEX分包方案，使用了多DEX加载的原理，大致的过程就是：

把BUG方法修复以后，放到一个单独的DEX里，插入到dexElements数组的最前面，让虚拟机去加载修复完后的方法。当patch.dex中包含Test.class时就会优先加载，在后续的DEX中遇到Test.class的
话就会直接返回而不去加载，这样就达到了修复的目的。


但是有一个问题是，当两个调用关系的类不在同一个DEX时，就会产生异常报错。我们知道，在APK安装时，虚拟机需要将classes.dex优化成odex文件，然后才会执行。在这个过程中，会进行类的verify
操作，如果调用关系的类都在同一个 DEX中的话就会被打上CLASS_ISPREVERIFIED的标志，然后才会写入odex文件。


所以，为了可以正常的进行打补丁修复，必须避免类被打上CLASS_ISPREVERIFIED 标志，具体的做法就是单独放一个类在另外DEX中，让其他类调用。

修复的主要步骤：

* 可以看出是通过获取到当前应用的Classloader
* 通过反射调用pathList的dexElements方法把patch.dex转化为Element[]
* 两个Element[]进行合并，把patch.dex放到最前面去
* 加载Element[]，达到修复目的

![](https://qin0821.github.io/images/Android/热修复/QQ空间热修复顺序.jpg)
![](https://qin0821.github.io/images/Android/热修复/QQ空间热修复流程图.jpg)

优势：

没有整合包（这是和下面微信的Tinker比起来），**产物表较小**，比较灵活
可以实现类替换，**兼容性高**。（某些三星手机不起作用）

不足：

不会及时生效，**必须重启**才能生效。
**耗时严重**，会增加启动时间，导致**ANR**的概率明显增大

#### [微信Tinker](https://github.com/Tencent/tinker)

微信针对QQ空间超级补丁技术的不足提出了一个提供DEX差量包，整体替换DEX的方案。主要的原理是与QQ空间超级补丁技术基本相同，区别在于不再将 patch.dex增加到elements数组中，而是差量
的方式给出patch.dex，然后将patch.dex与应用的classes.dex合并，然后整体替换掉旧的DEX，达到修复的目的。

![](https://qin0821.github.io/images/Android/热修复/WX顺序.jpg)
![](https://qin0821.github.io/images/Android/热修复/WX流程图.jpg)

优势：

合成整包，不用在构造函数插入代码，防止verify，verify和opt在编译期间就已经完成，不会在运行期间进行
性能提高。**兼容性和稳定性比较高**
开发者透明，**不需要对包进行额外处理**

劣势：

与超级补丁技术一样，不支持即时生效，**必须重启**应用的方式才能效
需要给应用开启新的进程才能进行合并，并且**很容易因为内存消耗等原因合并失败**。
合并时占用额外磁盘空间，对于多DEX的应用来说，如果修改了多个DEX文件，就需要下发多个patch.dex与对应的classes.dex进行合并操作时这种情况会更严重，因此合并过程的**失败率也会更高**

#### [阿里百川HotFix](https://help.aliyun.com/document_detail/181015.html)

阿里百川推出的热修复HotFix服务，相对于QQ空间超级补丁技术和微信Tinker来说，定位于紧急bug修复的场景下，能够最及时的修复bug，下拉补丁立即生效无需等待。
实现的原理：


AndFix不同于QQ空间超级补丁技术和微信Tinker通过增加或替换整个DEX的方案，提供了一种运行时在Native修改Filed指针的方式，实现方法的替
换，达到即时生效无需重启，对应用无性能消耗的目的

实现的过程步骤：

1. 打开链接库操作句柄，获得native层内部函数，得到ClassObject对象
2. 修改访问权限属性为public
3. 得到新旧方法的指针，新的方法指向目标方法，实现方法的替换

![](https://qin0821.github.io/images/Android/热修复/HotFix、AndFix关系.jpg)
![](https://qin0821.github.io/images/Android/热修复/HotFix原理.jpg)
![](https://qin0821.github.io/images/Android/热修复/HotFix顺序.jpg)

优势：

bug的修复的**及时性**
补丁包同样采用差量技术，生成patch**体积最小**
对应用无入侵，**几乎无性能损耗**

不足：

**不支持新增字段，以及修改<init>方法，也不支持对资源的替换。**
由于厂商的自定义ROM，**对少数机型暂不支持。**
**收费**

##### [美团Robust](https://github.com/Meituan-Dianping/Robust)

Robust 是 即时生效的Java层实现的热修复实现方案.
美团的Robust方案，是参考了谷歌的InstantRun方案（之前在androidStudio里面有这个选项，可以选择打开instant run运行app)的思路而设计出来的。
其主要设计思想就是一句话：编译打包时，在程序的某些方法里面，都插入一段代码（全自动操作）：
``` java
if (changeQuickRedirect != null) {
    return 修复的实现；
}
```
当 changeQuickRedirect不为空的时候，该方法就会命中 if(changeQuickRedirect!=null)，从而执行修复的实现代码。当为空的时候，则正常执行原逻辑。
而平时我们自己编码，只需要加上一个 @modify注解,来标记这个方法需要打补丁包，也就是需要插入上面这个 if(changeQuickRedirect!=null)代码段。

![](https://qin0821.github.io/images/Android/热修复/Robust原理.jpg)

上图解析：利用ClassLoader，当客户端手机收到补丁包 patch.dex的时候，执行补丁包，把指定方法的 changeQuickRedirect用反射的手段赋值，让它变成非空。从而让下一次程序逻辑走到这里的时候，走修复之后的逻辑。
Robust是如何将这么多 if(changeQuickRedirect!=null)代码段插入到代码逻辑中的?
上图中的 if(changeQuickRedirect!=null)代码段，并非我们手动编写，而是由Robust框架自动插入的。这个技术叫做`字节码插桩`，意为：对class进行操作，按照class文件的格式，插入自己想要的逻辑。目前Robust支持两种字节码插桩方案 `AspectJ` 和 `Javasist`.

优势：

bug的修复的**及时性**
**成功率最高**
**免费**

不足：

**不支持类、so、资源替换**

#### 总结

QQ空间超级补丁技术和微信Tinker 支持新增类和资源的替换，功能更为强大，但对应用的性能和稳定有一定的影响；

阿里百川HotFix功能有限，但能无感知即时修复，同时保证对应用性能不产生不必要的损耗。

![](https://qin0821.github.io/images/Android/热修复/技术对比.jpg)
