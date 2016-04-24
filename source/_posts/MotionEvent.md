title: 【非日常】Android事件体系
---
先提几个问题，然后去找找答案。首先要知道的是，Android的View的构成是树型的。
 
- view提供了setOnClickListener()方法,和setOnTouchListener()方法给我们设置监听，哪个先执行
- view传递的是什么？
- view(区别于viewGroup)中的touch事件是如何处理的？
- viewGroup是如何传递事件的？

### 设置的两种监听谁先执行？
 以TextView为例，其他的view也一个道理。

	textView = (TextView) findViewById(R.id.textview);
	        textView.setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View v) {
	                Log.d("onTouch", "textView is onClick");
	            }
	        });
	        textView.setOnTouchListener(new View.OnTouchListener() {
	            @Override
	            public boolean onTouch(View v, MotionEvent event) {
	                Log.d("onTouch", "textView is on touch");
	                return false;
	            }
	        });

	01-17 19:16:36.455  27855-27855/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 19:16:36.475  27855-27855/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 19:16:36.565  27855-27855/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 19:16:36.575  27855-27855/com.ycc.myapplication D/onTouch﹕ textView is onClick
结果先打印了3个onTouch，然后打印了onClick。到这结果就出来了。看个先后有什么意义？这会很大程度上帮助后面看源码时的理解。

### view传递的是什么

可以从上面的代码看出来，在onClick，onTouch方法里传了个MotionEvent对象。事件传递，传递的就是他。他里面传递了什么呢？找几个重要的出来。

 - Action。即行为。有ACTION_MOVE,ACTION_UP,ACTION_DOWN这3个。咦，联系上面打印的3个onTouch，是不是想到了什么。触摸屏幕可以分解成，按下，移动，抬起这3个过程。即Action对应的这3种，所以说每个过程都会触发一个MotionEvent。
 - getX(),getY(),getRawX(),getRawY()。
 
### view如何处理Touch事件。

先去找我们设置的监听在哪里被调用了。先找下onTouch()在哪被调用。

	public void setOnTouchListener(OnTouchListener l) {
 		//给ListenerInfo设置了一个listener
        getListenerInfo().mOnTouchListener = l;
    }
	public boolean dispatchTouchEvent(MotionEvent event) {
	//去掉了些暂时不考虑的东西。留下关键点。
	       	  	boolean result = false;
	          	ListenerInfo li = mListenerInfo;
	            if (li != null && li.mOnTouchListener != null
	                 && (mViewFlags & ENABLED_MASK) == ENABLED
	                 && li.mOnTouchListener.onTouch(this, event)) {
	//如果设置了监听事件，那么必然执行onTouch()，如果onTouch()返回true，那么result为true。
	                result = true;
				}
	            if (!result && onTouchEvent(event)) {//表明onTouch返回true，就不执行onTouchEvent
	                result = true;
	            }
	       	    return result;
	    }

如果onTouch()返回true，那么不执行onTouchEvent()，这个方法又是啥。想想第一次打印的时候先打印3次onTouch，再打印onClick，会不会onClick就是在onTouchEvent里执行的呢？假如说我们在onTouch里返回true，如果不打印onClick，是不是可以猜到什么了。
	
	01-17 20:13:38.111  26891-26891/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 20:13:38.131  26891-26891/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 20:13:38.191  26891-26891/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 20:13:38.231  26891-26891/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 20:13:38.561  26891-26891/com.ycc.myapplication D/onTouch﹕ textView is on touch
	01-17 20:13:38.571  26891-26891/com.ycc.myapplication D/onTouch﹕ textView is on touch
	
果然。去找一个onClick();

	public boolean performClick() {
	        final boolean result;
	        final ListenerInfo li = mListenerInfo;
	        if (li != null && li.mOnClickListener != null) {
	            playSoundEffect(SoundEffectConstants.CLICK);
		----------------------------------------------------------
	            li.mOnClickListener.onClick(this);
		----------------------------------------------------------
	            result = true;
	        } else {
	            result = false;
	        }
	
	        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
	        return result;
	    }


	public boolean onTouchEvent(MotionEvent event) {
		//只留我们想找的
	        final float x = event.getX();
	        final float y = event.getY();
	        final int viewFlags = mViewFlags;
	        final int action = event.getAction();
			if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {

 			if (mPerformClick == null) {
	             mPerformClick = new PerformClick();
	        }
	        if (!post(mPerformClick)) {
	----------------------------------------------------------------
	             performClick();
	----------------------------------------------------------------
	        }
			//只要CLICKABLE就返回true
				return true;
			}	
	       
	        rturn false;
	    }

onTouchEvent是view中对滑动事件做处理的函数。比如说viewPager,他自己处理事件的代码就是放在这里面的。如果我们setOnTouchListerner并且返回true，那viewPager自己处理事件的代码就失效了。因为onTouch返回true就不会执行onTouchEvent了。true在事件传递的概念可以理解为**消耗**。

到这里为止，事件在view中的传递过程就结束啦。


### ViewGroup如何传递事件

ViewGroup继承了View。

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        
        boolean handled = false;
       
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
	//可以通过requestDisallowInterceptTouchEvent方法设置是否允许viewGroup拦截
	------------------------------------------------------------
                    intercepted = onInterceptTouchEvent(ev);
	------------------------------------------------------------

                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } 
				intercepted = true;
            }

	//这里是遍历子view的过程，如果viewgroup不拦截的话执行
            if (!canceled && !intercepted) {
        		final View[] children = mChildren;
                for (int i = childrenCount - 1; i >= 0; i--) {
                     final int childIndex = customOrder
				     ? getChildDrawingOrder(childrenCount, i) : i;
                     final View child = (preorderedList == null)
                     ? children[childIndex] : preorderedList.get(childIndex);
		----------------------------------------------------------------------
                     if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                
                     }
  		------------------------------------------------------------------------                        
            }
        return handled;
    }

	//精简版
	private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
			//调用子view的dispatch方法
            handled = child.dispatchTouchEvent(transformedEvent);
        }
        return handled;
    }

	
viewgroup处理事件的过程是这样的：首先根据是否设置了不允许拦截事件和onInterceptTouchEvent的返回结果判断是否要拦截事件。默认不拦截。

	public boolean onInterceptTouchEvent(MotionEvent ev) {
	        return false;
	    }
如果不拦截的话就调用每个子view的dispatch方法。否则调用super.dispatchTouchEvent方法。
这里盗用下郭神的图，一目了然。

![盗下官方的图 (￣_,￣ )](http://7xpp4m.com1.z0.glb.clouddn.com/blog20130629200236578.png)

感觉viewGroup和view在事件处理上的不同就在于dispatchTouchEvent上，viewGroup强调的是由上层往下层的分发，而view是专注于自己这层内部的分发。
