有三个方案：

Kotlin/Js
成本相对低
性能差
只有单线程，CPU密集型任务处理慢

Kotlin/WasmJs
尚未正式发布
生态不够成熟
性能介于Kotlin/Js和Kotlin/Native之间

Kotlin/Native
成本高
性能好（反序列化json性能大约是Kotlin/Js的50-100倍）
有两套Runtime（Kotlin Native Runtime、 ArkJs Runtime），需要设计内存管理（可参考Bilibili的方案）
