#### view的setOnTouchListener 、OnTouchEvent、OnClickListener回调的执行顺序:

onTouch --> onTouchEvent --> OnClick

view 事件分发的相关源码如下：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
	...
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null
        && (mViewFlags & ENABLED_MASK) == ENABLED
        //此处执行 onTouchListener的 onTouch 方法
        && li.mOnTouchListener.onTouch(this, event)) {
        result = true;
    }
    //此处回调onTouchEvent
    if (!result && onTouchEvent(event)) {
                result = true;
    }
    ...
}

public boolean onTouchEvent(MotionEvent event) {
     ...
     switch (action) {
         case MotionEvent.ACTION_UP:
             if (!post(mPerformClick)) {
                 //此处执行onCLick
                performClickInternal();
             }
     }
    ...
}

private boolean performClickInternal() {
    ...

    return performClick();
}

public boolean performClick() {
    ...
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        //此处调用OnClickListener的onClick方法
        li.mOnClickListener.onClick(this);
        result = true;
    } 

    ...
        return result;
 }

```

由源码可知：  onTouchListener的回调 onTouch 方法是在dispatchTouchEvent中回调的， 并且是在onTouchEvent之前，而OnClickListener的onClick回调时机是在onTouchEvent中  所以调用顺序可想而知；