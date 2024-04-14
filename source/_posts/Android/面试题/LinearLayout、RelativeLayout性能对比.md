### 结论
（1）RelativeLayout慢于LinearLayout是因为它会让子View调用2次measure过程，而LinearLayout只需一次，但是有weight属性存在时，LinearLayout也需要两次measure。

（2）RelativeLayout的子View如果高度和RelativeLayout不同，会导致RelativeLayout在onMeasure()方法中做横向测量时，纵向的测量结果尚未完成，只好暂时使用自己的高度传入子View系统。而父View给子View传入的值也没有变化就不会做无谓的测量的优化会失效，解决办法就是可以使用padding代替margin以优化此问题。

（3）在不响应层级深度的情况下，使用Linearlayout而不是RelativeLayout。

### 性能对比

问题的核心在于，当RelativeLayout和LinearLayout分别作为ViewGroup表达相同布局时谁的绘制过程更快一点。

Hierarchy Viewer是随Android SDK发布的工具，位于Android SDK/tools/hierarchyviewer.bat，使用它可以来检测View绘制的三大过程的耗时。

通过网上的很多实验结果我们得之，两者绘制同样的界面时layout和draw的过程时间消耗相差无几，关键在于measure过程RelativeLayout比LinearLayout慢了一些。我们知道ViewGroup是没有onMeasure方法的，这个方法是交给子类自己实现的。因为不同的ViewGroup子类布局都不一样，那么onMeasure索性就全部交给他们自己实现好了。

所以我们就分别来追踪下RelativeLayout和LinearLayout的onMeasure过程来探索耗时问题的根源。

##### 1.1    RelativeLayout的onMeasure分析

```java
@Override  
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
//...
View[] views = mSortedHorizontalChildren;
int count = views.length;
for (int i = 0; i < count; i++) {
    View child = views[i];
    if (child.getVisibility() != GONE) {
         LayoutParams params = (LayoutParams) child.getLayoutParams();
         applyHorizontalSizeRules(params, myWidth);
         measureChildHorizontal(child, params, myWidth, myHeight);
         if (positionChildHorizontal(child, params, myWidth, isWrapContentWidth)) {
               offsetHorizontalAxis = true;
         }
    }
}
 
views = mSortedVerticalChildren;
count = views.length;
for (int i = 0; i < count; i++) {
     View child = views[i];
     if (child.getVisibility() != GONE) {
           LayoutParams params = (LayoutParams) child.getLayoutParams();
           applyVerticalSizeRules(params, myHeight);
           measureChild(child, params, myWidth, myHeight);
           if (positionChildVertical(child, params, myHeight, isWrapContentHeight)) {
                 offsetVerticalAxis = true;
           }
           if (isWrapContentWidth) {
                 width = Math.max(width, params.mRight);
           }
           if (isWrapContentHeight) {
                 height = Math.max(height, params.mBottom);
           }
           if (child != ignore || verticalGravity) {
                 left = Math.min(left, params.mLeft - params.leftMargin);
                 top = Math.min(top, params.mTop - params.topMargin);
           }
           if (child != ignore || horizontalGravity) {
                 right = Math.max(right, params.mRight + params.rightMargin);
                 bottom = Math.max(bottom, params.mBottom + params.bottomMargin);
           }
       }
  }
  //...
}
```
根据源码我们发现RelativeLayout会根据2次排列的结果对子View各做一次measure。这是为什么呢？首先RelativeLayout中子View的排列方式是基于彼此的依赖关系，而这个依赖关系可能和Xml布局中View的顺序不同，在确定每个子View的位置的时候，需要先给所有的子View排序一下。又因为RelativeLayout允许ViewB在横向上依赖ViewA，ViewA在纵向上依赖B。所以需要横向纵向分别进行一次排序测量。

同时需要注意的是View.measure()方法存在以下优化：

```java

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        if ((mPrivateFlags & FORCE_LAYOUT) == FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {
        ...
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
}
```

即如果我们或者我们的子View没有要求强制刷新，而父View给子View传入的值也没有变化（也就是说子View的位置没变化），就不会做无谓的测量。RelativeLayout在onMeasure中做横向测量时，纵向的测量结果尚未完成，只好暂时使用myHeight传入子View系统。这样会导致在子View的高度和RelativeLayout的高度不相同时（设置了Margin），上述优化会失效，在View系统足够复杂时，效率问题就会很明显。

##### 1.2  LinearLayout的onMeasure过程

```java

@Override  
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
  if (mOrientation == VERTICAL) {  
    measureVertical(widthMeasureSpec, heightMeasureSpec);  
  } else {  
    measureHorizontal(widthMeasureSpec, heightMeasureSpec);  
  }  
}  
//LinearLayout会先做一个简单横纵方向判断，我们选择纵向这种情况继续分析
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
//...
for (int i = 0; i < count; ++i) {  
      final View child = getVirtualChildAt(i);  
      //... child为空、Gone以及分界线的情况略去
     //累计权重
      LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();  
      totalWeight += lp.weight;  
      //计算
      if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {  
            //精确模式的情况下，子控件layout_height=0dp且weight大于0无法计算子控件的高度
            //但是可以先把margin值合入到总值中，后面根据剩余空间及权值再重新计算对应的高度
            final int totalLength = mTotalLength;  
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);  
      } else {  
           if (lp.height == 0 && lp.weight > 0) {  
            //如果这个条件成立，就代表 heightMode不是精确测量以及wrap_conent模式
            //也就是说布局是越小越好，你还想利用权值多分剩余空间是不可能的，只设为wrap_content模式
                 lp.height = LayoutParams.WRAP_CONTENT;  
           }  
  
          // 子控件测量
          measureChildBeforeLayout(child, i, widthMeasureSpec,0, heightMeasureSpec,totalWeight== 0 ? mTotalLength :0);         
          //获取该子视图最终的高度，并将这个高度添加到mTotalLength中
          final int childHeight = child.getMeasuredHeight();  
          final int totalLength = mTotalLength;  
          mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin + lp.bottomMargin + getNextLocationOffset(child)); 
          } 
        //...
}
```
源码中已经标注了一些注释，需要注意的是在每次对child测量完毕后，都会调用child.getMeasuredHeight()获取该子视图最终的高度，并将这个高度添加到mTotalLength中。但是getMeasuredHeight暂时避开了lp.weight>0且高度为0子View，因为后面会将把剩余高度按weight分配给相应的子View。因此可以得出以下结论：

（1）如果我们在LinearLayout中不使用weight属性，将只进行一次measure的过程。

（2）如果使用了weight属性，LinearLayout在第一次测量时获取所有子View的高度，之后再将剩余高度根据weight加到weight>0的子View上。

由此可见，weight属性对性能是有影响的。



### 补充

在以前的版本，新建一个Android项目SDK会为我们自动生成的avtivity_main.xml布局文件，然后它的根节点默认是RelativeLayout，在我们的理解里貌似LinearLayout的性能是要比RelativeLayout更优的，比如作为顶级View的DecorView就是个垂直方向的LinearLayout，上面是标题栏，下面是内容栏，我们常用的setContentView()方法就是给内容栏设置布局。

为什么以前布局文件默认根节点是RelativeLayout呢？

结论中的第三条也解释了文章前言中的问题：DecorView的层级深度已知且固定的，上面一个标题栏，下面一个内容栏，采用RelativeLayout并不会降低层级深度，因此这种情况下使用LinearLayout效率更高。

而为开发者默认新建RelativeLayout是希望开发者能采用尽量少的View层级，很多效果是需要多层LinearLayout的嵌套，这必然不如一层的RelativeLayout性能更好。因此我们应该尽量减少布局嵌套，减少层级结构，使用比如viewStub，include等技巧。

