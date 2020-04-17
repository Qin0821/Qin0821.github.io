### Fragment详细生命周期

##### 结论

无|创建|按Home键|按Home键之后切回应用|按返回键
:-:|:-:|:-:|:-:|:-:
onAttach|✅|||
onCreate|✅|||
onCreateView|✅|||
onActivityCreated|✅|||
onStart|✅||✅|
onResume|✅||✅|
onPause||✅||✅
onStop||✅||✅
onDestroyView||||✅
onDestroy||||✅
onDetach||||✅

在下列方法中可以用Bundle对象保存fragment对象，以供场景还原等操作：

* onCreate()
* onCreateView()
* onActivityCreated()

fragments的大部分状态都和activitie很相似，但fragment有一些新的状态。

* onAttached() —— 当fragment和activity关联之后，调用这个方法。
* onCreateView() —— 创建fragment中的视图的时候，调用这个方法。
* onActivityCreated() —— 当activity的onCreate()方法被返回之后，调用这个方法。
* onDestroyView() —— 当fragment中的视图被移除的时候，调用这个方法。
* onDetach() —— 当fragment和activity分离的时候，调用这个方法。

https://blog.csdn.net/asdf717/article/details/51383750