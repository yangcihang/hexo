---
title: ScrollView嵌套ListView问题
date: 2017-10-21 23:11:20
tags: Android
categories: Android
---
## 问题情形
 当ListView外面嵌套一层ScrollView时，内层的ListView会有这样的问题：
 1. ListView的尺寸无法正确测量。如果不将高度固定的话，ListView只会显示一个Item。
 2. ListView的滑动事件被截止。如果内容超过了屏幕的大小的话，会截止ListView的滑动事件。<!--more-->

## 源码分析

 通常情况下，ViewGroup是不拦截事件的，但是ScrollView在onInterceptTouchEvent方法中，有这么几句：
```
if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
        return true;
    }
//在动作事件中:
  case MotionEvent.ACTION_MOVE: {

            final int activePointerId = mActivePointerId;
            if (activePointerId == INVALID_POINTER) {
                // If we don't have a valid id, the touch down wasn't on content.
                break;
            }

            final int pointerIndex = ev.findPointerIndex(activePointerId);
            if (pointerIndex == -1) {
                Log.e(TAG, "Invalid pointerId=" + activePointerId+ " in onInterceptTouchEvent");
                break;
            }

            final int y = (int) ev.getY(pointerIndex);
            final int yDiff = Math.abs(y - mLastMotionY);
            if (yDiff > mTouchSlop && (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
                mIsBeingDragged = true;
                mLastMotionY = y;
                initVelocityTrackerIfNotExists();
                mVelocityTracker.addMovement(ev);
                mNestedYOffset = 0;
                if (mScrollStrictSpan == null) {
                    mScrollStrictSpan = StrictMode.enterCriticalSpan("ScrollView-scroll");
                }
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }

         case MotionEvent.ACTION_DOWN: {
            final int y = (int) ev.getY();
            if (!inChild((int) ev.getX(), (int) y)) {
                mIsBeingDragged = false;
                recycleVelocityTracker();
                break;
            }

            /*
             * Remember location of down touch.
             * ACTION_DOWN always refers to pointer index 0.
             */
            mLastMotionY = y;
            mActivePointerId = ev.getPointerId(0);

            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);

            mIsBeingDragged = !mScroller.isFinished();
            if (mIsBeingDragged && mScrollStrictSpan == null) {
                mScrollStrictSpan = StrictMode.enterCriticalSpan("ScrollView-scroll");
            }
            startNestedScroll(SCROLL_AXIS_VERTICAL);
            break;
        }
```
这两段代码的意思是：如果ScrollView接收到move事件，并且正在滑动，那么它的onInterceptTouchEvent方法直接就返回true了，也就是拦截了滑动事件。当接收到了按下的事件，那么将 mIsBeingDragged直接置false，
mIsBeingDragged就表示当前ScrollView是否在滑动。里面还有这么一句mIsBeingDragged = !mScroller.isFinished();即如ScrollView还在滑动的时候，你想去触摸嵌在里面的ListView，ScrollView滑动还没结束呢，直接return true;
再来看看switch里面case MotionEvent.ACTION_MOVE，如果y轴方向上的滑动距离大于最小滑动距离，则将mIsBeingDragged设置为true。也就是说虽然我手指落在了子View里面，但是如果我要滑动的话，根据第一个if判断，会直接被拦截，交给ScrollView去处理。

## 处理方法

 - ListView滑动拦截的处理，思路很简单：我们可以自定义一个ListView，去重写里面的dispatchTouchEvent方法，在它的 MotionEvent.ACTION_DOWN的事件调用:getParent().requestDisallowInterceptTouchEvent(true)方法，意思是通知其父组件不要拦截它的滑动事件。这时候一定要记着当ListView滑动到最底部和最顶部的时候要将滑动事件交给其父组件来处理。自定义的ListView代码如下：

```
public class MyListView extends ListView {
    float y,downY;
    float mTouchSlop = ViewConfiguration.getTouchSlop(); //获取滑动最小位移量

    public MyListView(Context context) {
        super(context);
    }

    public MyListView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    /**
     * 对事件分发的重写
     * @param ev
     * @return
     */
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                downY = ev.getRawY();
                y = downY;
                getParent().requestDisallowInterceptTouchEvent(true);
                break;
            case MotionEvent.ACTION_MOVE:
                y = ev.getRawY();
                /*
                  判断滑动到顶部
                 */
                if (getFirstVisiblePosition() == 0) {
                    if (y - downY > mTouchSlop) {
                        getParent().requestDisallowInterceptTouchEvent(false); 
                        return false;
                    } else if (y - downY < -mTouchSlop) {
                        getParent().requestDisallowInterceptTouchEvent(true); //滑到顶部再向上，自己处理

                    }

                }
                /*
                  判断滑动到底部
                 */
                if (getFirstVisiblePosition()+getChildCount()==getCount()) {
                    if (y - downY < -mTouchSlop) {
                        getParent().requestDisallowInterceptTouchEvent(false);
                        return false;
                    } else if (y - downY > mTouchSlop) {
                        getParent().requestDisallowInterceptTouchEvent(true); //滑到底部再向下，自己处理
                    }
                }
                break;
            case MotionEvent.ACTION_UP:
                break;
            default:
                break;
        }
        return super.dispatchTouchEvent(ev);
    }

}
```

- 对显示只有一项的情况处理：出现这种情况的原因是ScrollView嵌套ListView时，ListView无法正确计算大小，因此可以通过在其初始化的时候固定其大小来解决，但有时候并不知道View应该多大，这时候就得使用代码来控制了，具体的实现方法如下：

```
public void setListViewHeight(ListView listView) {
        ListAdapter adapter = listView.getAdapter();
        int totalHeight = 0; //子View的总高度
        int len  = adapter.getCount();
        for (int i = 0; i < len; i++) {
            View listItem = adapter.getView(i, null, listView);
            listItem.measure(0, 0); //对子View进行测量
            totalHeight += listItem.getMeasuredHeight();
        }
        ViewGroup.LayoutParams params = listView.getLayoutParams();
        params.height = totalHeight+ (listView.getDividerHeight() * (adapter.getCount() - 1));//加上分割线的高度
        listView.setLayoutParams(params);
    }
```

在ListView加载完adapter后，可以通过调用上面的方法来实现对ListView的大小控制。其中的思路是先通过获得ListView的itemView的总数量，然后进行测量，测量后在重新设置ListView的参数。