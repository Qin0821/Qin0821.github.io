## Android 10适配

### 前言

2019 年 9 月 3 日，Google 发布了 Android 10 正式版。Android 10 聚焦移动创新、安全隐私和数字健康三大主题，全面打造最佳用户体验。

在Android 10(API =29) 版本中，官方的改动较大，相应的开发者适配成本还是很高的。

我们主要基于以下几方面进行Android 10的适配：

* Android X
* 分区存储
* 设备ID
* 明文HTTP限制

### 一、AndroidX

AndroidX 对原始 Android Support库进行了重大改进，后者现在已不再维护。AndroidX 软件包完全取代了支持库，不仅提供同等的功能，而且提供了新的库。

**1.1 什么是AndroidX**

Android系统在刚刚面世的时候，可能连它的设计者也没有想到它会如此成功。随着Android系统版本不断地迭代更新，每个版本中都会加入很多新的API进去，但是新增的API在老版系统中并不存在，因此这就出现了一个向下兼容的问题。

于是Android团队推出了一个鼎鼎大名的Android Support Library，用于提供向下兼容的功能。比如我们熟知的support-v4库，appcompat-v7库都是属于Android Support Library的。4在这里指的是Android API版本号，对应的系统版本是1.6。support-v4的意思就是这个库中提供的API会向下兼容到Android 1.6系统。类似地，appcompat-v7指的是将库中提供的API向下兼容至API 7，也就是Android 2.1系统。

随着时间的推移，Android1.6、2.1系统早已被淘汰了，现在Android官方支持的最低系统版本已经是4.0.1，对应的API版本号是15。support-v4、appcompat-v7库也不再支持那么久远的系统了，但是它们的名字却一直保留了下来，虽然它们现在的实际作用已经对不上当初命名的原因了。

Android团队也意识到这种命名已经非常不合适了，于是对这些API的架构进行了一次重新的划分，推出了AndroidX。因此，AndroidX本质上其实就是对Android Support Library进行的一次升级。

**1.2 为什么要升级AndroidX**

- 版本 28.0.0 是Android Support 库的最后一个版本。官方将不再发布 android.support 库版本。所有新功能都将在 AndroidX命名空间中开发。
- 长远来看。AndroidX重新设计了包结构，旨在鼓励库的小型化，支持库和架构组件包的名字进行了简化。而且这也是减轻Android生态系统碎片化的有效方式。
- 与Android Support库不同，AndroidX软件包是单独维护和更新的。这些AndroidX包使用严格的语义版本控制，从版本1.0.0开始，您可以单独更新项目中的AndroidX库。

## **1.3 适配步骤**



### 1.3.1 环境准备



- AndroidStudio 3.2.0+
- gradle：gradle-4.6+



另外修改相关app、library模块中build.gradle的compileSdkVersion、targetSdkVersion、buildToolsVersion的配置，都设置为29，示例如下：

```groovy
android {   
    compileSdkVersion 29   
    buildToolsVersion 29.0.2   
    defaultConfig {      
        targetSdkVersion 29   
    }   
    ...
}
```

 

1.3.2 修改当前项目的 gradle.properties

```groovy
android.useAndroidX=true
android.enableJetifier=true
```

 其中：

- android.useAndroidX=true 表示当前项目启用 AndroidX；
- android.enableJetifier=true 表示将依赖包也迁移到AndroidX 。如果取值为 false ,表示不迁移依赖包到AndroidX，但在使用依赖包中的内容时可能会出现问题，如果你的项目中没有使用任何三方依赖，此项可以设置为 false。

### 1.3.3 修改项目中的build.gradle依赖库

```groovy
implementation 'com.android.support:appcompat-v7:28.0.0'
→ implementation 'androidx.appcompat:appcompat:1.0.2'
implementation 'com.android.support:design:28.0.0'
→implementation 'com.google.android.material:material:1.0.0'
implementation 'com.android.support.constraint:constraint-layout:1.1.3'
→ implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
```

映射关系：



https://developer.android.com/jetpack/androidx/migrate/artifact-mappings



### 1.3.4 修改支持库类

将原来import的android.**包删除，重新import新的androidx.**包；

```groovy
import android.support.v7.app.AppCompatActivity; 
→import androidx.appcompat.app.AppCompatActivity;
```

### 1.3.5 迁移

官方迁移指南：

https://developer.android.com/jetpack/androidx/migrate#migrate

在 AndroidStudio 3.2 或更高版本（截图中 AndroidStudio 为 3.5 版本）中执行如下操作：菜单>Refactor > Migrate to AndroidX（如果迁移失败，就需要重复上面1，2，3，4步手动去修改迁移）

![img](https://mmbiz.qpic.cn/mmbiz_png/kEeDgfCVf1c3u3uJ0ahmXm5cpiboCDXQkCGyVfSQZPNllSxjrajrqbHugxM00qbUb4jzBnxafOjKc2ctGgEknAg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 注意：

- 使用AS迁移工具并不能完全修改完毕，需要手动修改
- support包名涉及到资源修改，切记检查资源中的类路径

## **二、分区存储**

**2.1 背景介绍**

为了更好的保护用户数据并限制设备冗余文件增加，以 Android 10（API 级别 29）及更高版本为目标平台的应用在默认情况下被赋予了对外部存储设备的分区访问权限（即分区存储）

应用只能看到本应用专有的目录（通过 Context.getExternalFilesDir() 访问）以及特定类型的媒体。除非您的应用需要访问存放在应用的专有目录以及 MediaStore 之外的文件，否则最好使用分区存储。

要点：

- Android Q文件存储机制修改成了沙盒模式
- APP只能访问自己目录下的文件和公共媒体文件
- Android Q版本以下机型，还是使用老的文件存储方式
- Android Q及以上版本机型，所有应用均需要分区存储, 所以应用需要提前确保支持分区存储

需要注意：在适配AndroidQ的时候还要兼容Q系统版本以下的，使用SDK_VERSION区分

## **2.2 新特性概览** 

### 2.2.1 外部存储

外部存储被分为应用私有目录以及共享目录两个部分：

- 应用私有目录：存储应用私有数据，外部存储应用私有目录对应Android/data/packagename，内部存储应用私有目录对应data/data/packagename；
- 共享目录：存储其他应用可访问文件， 包含媒体文件、文档文件以及其他文件，对应设备DCIM、Pictures、Alarms, Music, Notifications,Podcasts, Ringtones、Movies、Download等目录

### 1）私有目录

应用私有目录文件访问方式与之前Android版本一致，可以通过File path获取资源。

### 2）共享目录

共享目录文件需要通过MediaStore API或者Storage Access Framework方式访问。

- MediaStore API在共享目录指定目录下创建文件或者访问应用自己创建文件，不需要申请存储权限
- MediaStore API访问其他应用在共享目录创建的媒体文件(图片、音频、视频)， 需要申请存储权限，未申请存储权限，通过ContentResolver查询不到文件Uri，即使通过其他方式获取到文件Uri，读取或创建文件会抛出异常；
- MediaStore API不能够访问其他应用创建的非媒体文件(pdf、office、doc、txt等)， 只能够通过Storage Access Framework方式访问；

## **2.3 受影响的变更**

### 2.3.1 图片位置信息

一些图片会包含位置信息，因为位置对于用户属于敏感信息， Android 10应用在分区存储模式下图片位置信息默认获取不到，应用通过以下两项设置可以获取图片位置信息：

- 在manifest中申请ACCESS_MEDIA_LOCATION
- 调用MediaStore setRequireOriginal(Uri uri)接口更新图片Uri

### 2.3.2 访问数据

MediaStore.Files应用分区存储模式下，MediaStore.Files 集合只能够获取媒体文件信息(图片、音频、视频)， 获取不到非media(pdf、office、doc、txt等)文件。

### 2.3.3 File Path路径访问受影响接口

开启分区存储新特性， Andrioid 10不能够通过File Path路径直接访问共享目录下资源，以下接口通过File 路径操作文件资源，功能会受到影响，应用需要使用MediaStore或者SAF方式访问。



|       类名称       |                  受影响的接口                   |
| :----------------: | :---------------------------------------------: |
|       `File`       |                `createNewFile()`                |
|                    |                   `delete()`                    |
|                    |              `renameTo(File dest)`              |
|                    |                    `mkdir()`                    |
|                    |                   `mkdirs()`                    |
| `FileInputStream`  |          `FileInputStream(File file)`           |
|                    |         `FileInputStream(String name)`          |
| `FileOutputStream` |         `FileOutputStream(String name)`         |
|                    | `FileOutputStream(String name, boolean append)` |
|                    |          `FileOutputStream(File file)`          |
|                    |  `FileOutputStream(File file, boolean append)`  |
|  `BitmapFactory`   |          `decodeFile(String pathName)`          |
|                    |   `decodeFile(String pathName, Options opts)`   |

### 2.3.4 存储特性Android版本差异概览

![储存特性版本差异](/Users/qin/ting/blog/hexo/qin0821.github.io/source/_posts/Android/版本适配/Android 10/QQ20200417-133026.png)

## **2.4 兼容模式**



应用未完成外部存储适配工作，可以临时以兼容模式运行， 兼容模式下应用申请存储权限，即可拥有外部存储完整目录访问权限，通过Android10之前文件访问方式运行，以下两种方法设置应用以兼容模式运行。



### 2.4.1 AndroidManifest中申明

tagretSDK 大于等于Android 10（API level 29）， 在manifest中设置requestLegacyExternalStorage属性为true。

```xml
<manifest ...>
...
    <application android:requestLegacyExternalStorage="true" ... >
...
</manifest>
```

### 2.4.2、判断兼容模式接口

```java
//返回值
//true : 应用以兼容模式运行
//false：应用以分区存储特性运行
Environment.isExternalStorageLegacy();
```

备注：应用已完成存储适配工作且已打开分区存储开关，如果当前应用以兼容模式运行，覆盖安装后应用仍然会以兼容模式运行，卸载重新安装应用才会以分区存储模式运行

## **2.5 适配方案**

### 2.5.1 方案概览

分区存储适配包含文件迁移以及文件访问兼容性适配两个部分：

#### 1）文件迁移

文件迁移是将应用共享目录文件迁移到应用私有目录或者Android10要求的media集合目录。

- 针对只有应用自己访问并且应用卸载后允许删除的文件，需要迁移文件到应用私有目录文件，可以通过File path方式访问文件资源，降低适配成本。
- 允许其他应用访问，并且应用卸载后不允许删除的文件，文件需要存储在共享目录，应用可以选择是否进行目录整改，将文件迁移到Android10要求的media集合目录。

#### 2）文件访问兼容性

共享目录文件不能够通过File path方式读取，需要使用MediaStore API或者Storage Access Framework框架进行访问。

### 2.5.2 适配指导

AndroidQ中使用ContentResolver进行文件的增删改查。

### 1）获取(创建)私有目录下的文件夹

```java
//在自身目录下创建apk文件夹
File apkFile = context.getExternalFilesDir("apk");
```

### 2）创建私有目录文件

生成需要下载的路径，通过输入输出流读取写入

```java
String apkFilePath = context.getExternalFilesDir("apk").getAbsolutePath();
File newFile = new File(apkFilePath + File.separator + "demo.apk");
OutputStream os = null;
try {    
    os = new FileOutputStream(newFile);    
    if (os != null) {        
        os.write("file is created".getBytes(StandardCharsets.UTF_8));        
        os.flush();    
    }
} catch (IOException e) {
} finally {    
    try {        
        if (os != null) {        
            os.close();    
        }catch (IOException e1) {    
        }
}
```

### 3）创建共享目录文件夹

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {    
    ContentResolver resolver = context.getContentResolver();    
    ContentValues values = new ContentValues();
    values.put(MediaStore.Downloads.DISPLAY_NAME, fileName);
    values.put(MediaStore.Downloads.DESCRIPTION, fileName);    
    //设置文件类型    
    values.put(MediaStore.Downloads.MIME_TYPE, "application/vnd.android.package-archive");
    //注意MediaStore.Downloads.RELATIVE_PATH需要targetVersion=29,    
    //故该方法只可在Android10的手机上执行    
    values.put(MediaStore.Downloads.RELATIVE_PATH, "Download" + File.separator + "apk");    
    Uri external = MediaStore.Downloads.EXTERNAL_CONTENT_URI;    
    Uri insertUri = resolver.insert(external, values);    
    return insertUri;
}else{
    ...
}
```

### 4）在共享目录指定文件夹下创建文件

主要是在公共目录下创建文件或文件夹拿到本地路径uri，不同的Uri，可以保存到不同的公共目录中。接下来使用输入输出流就可以写入文件。

重点：AndroidQ中不支持file://类型访问文件，只能通过uri方式访问。

```java
/**
  * 创建图片地址uri,用于保存拍照后的照片 Android 10以后使用这种方法
  */
private Uri  createImageUri() {    
    String status = Environment.getExternalStorageState();    
    // 判断是否有SD卡,优先使用SD卡存储,当没有SD卡时使用手机存储    
    if (status.equals(Environment.MEDIA_MOUNTED)) {        
        return getContext().getContentResolver().insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, new ContentValues());    
    } else {        
        return getContext().getContentResolver().insert(MediaStore.Images.Media.INTERNAL_CONTENT_URI, new ContentValues());    
    }
}
```

 

### 5）通过MediaStore API读取公共目录下的文件

```java
if (cursor != null && cursor.moveToFirst()) {    
    do {        
        ...        
        int _id = cursor.getInt(cursor.getColumnIndex(MediaStore.Images.Media._ID));        
        Uri imageUri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, _id);        
        ...    
    } while (!cursor.isLast() && cursor.moveToNext());
} else {
    ...
}
```

```java
// 通过uri获取bitmap
public Bitmap getBitmapFromUri(Context context, Uri uri) {    
    ParcelFileDescriptor parcelFileDescriptor = null;    
    FileDescriptor fileDescriptor = null;    
    Bitmap bitmap = null;    
    try {        
        parcelFileDescriptor = context.getContentResolver().openFileDescriptor(uri, "r");  
        if (parcelFileDescriptor != null && parcelFileDescriptor.getFileDescriptor() != null) { 
            fileDescriptor = parcelFileDescriptor.getFileDescriptor();            
            //转换uri为bitmap类型            
            bitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor);        
        }    
    } catch (Exception e) {        
        e.printStackTrace();    
    }finally {        
        try {            
            if (parcelFileDescriptor != null) {            
                parcelFileDescriptor.close();        
            }catch (IOException e) {        
            }    
        }    
        return bitmap;
    }
```

6）使用MediaStore删除文件

```java
context.getContentResolver().delete(fileUri, null, null);
```

## **三、设备ID**

从Android 10开始已经无法完全标识一个设备，曾经用mac地址、IMEI等设备信息标识设备的方法，从Android 10开始统统失效。而且无论你的APP是否适配过Android 10。

### **3.1 IMEI等设备信息**

从Android10开始普通应用不再允许请求权限android.permission.READ_PHONE_STATE。而且，无论你的App是否适配过Android Q（既targetSdkVersion是否大于等于29），均无法再获取到设备IMEI等设备信息。

受影响的API：

```java
Build.getSerial();
TelephonyManager.getImei();
TelephonyManager.getMeid();
TelephonyManager.getDeviceId();
TelephonyManager.getSubscriberId();
TelephonyManager.getSimSerialNumber();
```

- targetSdkVersion<29 的应用，其在获取设备ID时，会直接返回null
- targetSdkVersion>=29 的应用，其在获取设备ID时，会直接抛出异常SecurityException

如果您的App希望在Android 10以下的设备中仍然获取设备IMEI等信息，可按以下方式进行适配：

```xml
<uses-permission 
     android:name="android.permission.READ_PHONE_STATE"        
     android:maxSdkVersion="28"
/>
```

**3.2 Mac地址随机分配**

从Android10开始，默认情况下，在搭载 Android 10 或更高版本的设备上，系统会传输随机分配的 MAC 地址。（即从Android 10开始，普通应用已经无法获取设备的真正mac地址，标识设备已经无法使用mac地址）

**3.3 如何标识设备唯一性**

### 3.3.1 Google解决方案：如果您的应用有追踪非登录用户的需求，可用ANDROID_ID来标识设备。

- ANDROID_ID生成规则：签名+设备信息+设备用户
- ANDROID_ID重置规则：设备恢复出厂设置时，ANDROID_ID将被重置

```java
String androidId = Settings.Secure.getString(this.getContentResolver(), Settings.Secure.ANDROID_ID);
```

### 3.3.2 信通院统一SDK（OAID）

统一标识依据电信终端产业协会(TAF)、移动安全联盟(MSA)联合推 出的团体标准《移动智能终端补充设备标识规范》开发，移动智能终端补充设备标识体系统一调用 SDK 集成设备厂商提供的接口，并获得主流设备厂商的授权。

移动安全联盟(MSA)组织中国信息通信研究院(以下简称“中国信通院”)与终端生产企业、互联网企业共同研究制定了“移动智能终端补充设备标识体系”，定义了移动智能终端补充设备标识体系的体系架构、功能要求、接口要求以及安全要求，使设备生产企业统一开发接口，为移动应用开发者提供统一调用方式，方便移动应用接入，降低维护成本。

#### 1）SDK获取

MSA 统一 SDK 下载地址：

移动安全联盟官网，http://www.msa-alliance.cn/

#### 2）接入方式

- 解压miit_mdid_sdk_v1.0.13.rar，
- 把 miit_mdid_1.0.13.aar 拷贝到项目中，并设置依赖。
- 将 supplierconfig.json 拷贝到项目 assets 目录下，并修改里边对应 内容，特别是需要设置 appid 的部分。需要设置 appid 的部分需要去对应的厂 商的应用商店里注册自己的 app。

```json
{  
	"supplier":{    
        "xiaomi":{      
            "appid":"***"    
        },    
        "huawei":{      
            "appid":"***"    
        }    
        ...  
    }
}
```

- 在初始化方法中调用JLibrary.InitEntry

```java
try {    
    JLibrary.InitEntry(FoundationContextHolder.getContext());
} catch (Throwable e) {
}
```

- 实例化MSA SDK

```java
public static void initMSASDK(Context context){    
    int code = 0;    
    try {        
        code =  MdidSdkHelper.InitSdk(context,true,listener);        
        if (code == ErrorCode.INIT_ERROR_MANUFACTURER_NOSUPPORT){
            //1008611,不支持的厂商        
        }else if (code == ErrorCode.INIT_ERROR_DEVICE_NOSUPPORT){
            //1008612,不支持的设备        
        }else if (code == ErrorCode.INIT_ERROR_LOAD_CONFIGFILE){
            //1008613,加载配置文件失败        
        }else if (code == ErrorCode.INIT_ERROR_RESULT_DELAY){
            //1008614,信息将会延迟返回，获取数据可能在异步线程，取决于设备        
        }else if (code == ErrorCode.INIT_HELPER_CALL_ERROR){
            //1008615,反射调用失败        
        }        
        //code可记录异常供分析    
    }catch (Throwable throwable){    
    }
}

static IIdentifierListener listener = new IIdentifierListener() {    
    @Override    
    public void OnSupport(boolean support, IdSupplier idSupplier) {        
        try{            
            isSupport  = support;            
            if (null != idSupplier && isSupport){                
                //是否支持补充设备标识符获取                
                oaid = idSupplier.getOAID();                
                aaid = idSupplier.getAAID();                
                vaid = idSupplier.getVAID();           
            }else {           
                ...         
            }     
        }catch (Exception e){    
        }  
    }
};
```

- 通过以上方法获取到OAID等设备标识之后，即可作为唯一标识使用。

## **四、明文HTTP限制**

当SDK版本大于API 28时，默认限制了HTTP请求，并出现相关日志“[java.net](http://java.net/).UnknownServiceException: CLEARTEXT communication to xxx not permitted by network security policy“。

#### 该问题有两种解决方案：

### 1）在AndroidManifest.xml中Application节点添加如下代码

```xml
<application android:usesCleartextTraffic="true">
```

 ### 2）在res目录新建xml目录，已建的跳过 在xml目录新建一个xml文件network_security_config.xml，然后在AndroidManifest.xml中Application添加如下节点代码。

```xml
android:networkSecurityConfig="@xml/network_config"
```

network_config.xml（命名随机）

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>   
	<base-config cleartextTrafficPermitted="true" />
</network-security-config>
```



## **参考文档：**

0、[携程Android 10适配踩坑指南](https://mp.weixin.qq.com/s?__biz=MjM5MDI3MjA5MQ==&mid=2697269503&idx=2&sn=f5505724dcee64ebd9904ee16a2bfedb&scene=21#wechat_redirect)

1、[AndroidX 概览](https://developer.android.google.cn/jetpack/androidx)

2、[Android 10介绍](https://developer.android.com/about/versions/10)

3、[Android 11预览版介绍](https://developer.android.com/preview)

4、[Android Q Adaptation Guide](https://chinesefoodstudio.com/index.php/2019/11/21/android-q-adaptation-guide/)

5、[Android 10分区存储介绍及百度APP适配实践](https://segmentfault.com/a/1190000021760036)