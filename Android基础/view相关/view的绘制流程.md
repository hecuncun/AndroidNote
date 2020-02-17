#### view的绘制流程：

首先我们需要清楚view绘制是从哪里开始的? 每个Activity都会创建一个PhoneWindow对象，每个window对应着一个view(DecorView) 和一个viewRootImpl， ViewRootImpl是连接WindowManager 和 DecorView 的纽带， 绘制入口是从ViewRootImpl的performTraversals() 方法发起的：

```java
private void performTraversals() { 
...... 
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width); 
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height); 
...... 
// mView 代表的就是DecorView(FramLayout)
//测量    
mView.measure(childWidthMeasureSpec, childHeightMeasureSpec); 
......
//布局
mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
......
//绘制    
mView.draw(canvas); 
......
}

private static int getRootMeasureSpec(int windowSize, int rootDimension) { 
   int measureSpec; 
   switch (rootDimension) { 
   case ViewGroup.LayoutParams.MATCH_PARENT: 
   // Window can't resize. Force root view to be windowSize.   
   measureSpec = MeasureSpec.makeMeasureSpec(windowSize,MeasureSpec.EXACTLY);
   break; 
   ...... 
  } 
 return measureSpec; 
}
```

由源码可以直到 主要的绘制流程是measure --- layout --- draw；

measure 主要负责测量view 大小；里面主要参数是MeasureSpec，MeasureSpec构造方法是size 和mode  其中mode 代表的是view的测量模式 EXACTLY(match_parent和指定大小)  AT_MOST(wrap_content) UNSPECIFIED(系统内部会用到)

layout主要负责view 的布局工作（viewGroup中onLayout有实现, view中由于没有子view所以是空实现)

draw主要负责view的绘制工作，主要关键的是对canvas 和paint的一些使用；view中重写onDraw方法 viewGroup中重写dispatchDraw方法；



