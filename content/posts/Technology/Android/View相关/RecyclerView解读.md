---
title: 对SpringGateway+Security+OAuth2.0的认识
date: 2021-05-10 17:49:10
tags: [Android]
---

## RV是什么

```java
public class RecyclerView extends ViewGroup implements ScrollingView{
    ...
}
```

就是个`ViewGroup`，并且遵循了`ScrollingView`接口，所以支持滑动。

那么，既然是VG，那么我们就从VG的核心方法入手，他们分别是：

- onMeasure
- onLayout
- onDrow

## RV的状态

在开始RV的内窥之前，我们先了解一下RV有几种状态，这样有利于我们的理解。

```java
@IntDef(flag = true, value = {
        STEP_START, STEP_LAYOUT, STEP_ANIMATIONS
})
@Retention(RetentionPolicy.SOURCE)
@interface LayoutState {}

@LayoutState
int mLayoutStep = STEP_START;
```

- STEP_START：开始状态
- STEP_LAYOUT：布局状态
- STEP_ANIMATIONS：动画状态



## onMeasure

```java
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    //如果没有设置layoutManager，则采用padding+最小尺寸
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    //判断layoutManager中的一个标志位，但是这个标志位已经标明过期了
    if (mLayout.isAutoMeasureEnabled()) {
        //拿到mode
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);

        //让layoutManager测量
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

        final boolean measureSpecModeIsExactly =
                widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;

        //如果是match_parent或这是个确定的值，而此时没有adapter，那就不玩了，等下次measure时再处理
        if (measureSpecModeIsExactly || mAdapter == null) {
            return;
        }

        //如果是开始状态，
        if (mState.mLayoutStep == State.STEP_START) {
            //第一次layout过程
            dispatchLayoutStep1();
        }
        // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
        // consistency
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;

        //第二次layout过程
        dispatchLayoutStep2();

        // 从children中获取宽高
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        //如果RV存在不精确的宽高，或者存在child也不精确，我们不得不再次measure
        if (mLayout.shouldMeasureTwice()) {

            //用当前已测量过的宽高，重新生成MeasureSpec，并设置给layoutMnager
            mLayout.setMeasureSpecs(
                    MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            //重启过程2
            dispatchLayoutStep2();
            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }
    } else {
        if (mHasFixedSize) {
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            return;
        }
        // custom onMeasure
        if (mAdapterUpdateDuringMeasure) {
            startInterceptRequestLayout();
            onEnterLayoutOrScroll();
            processAdapterUpdatesAndSetAnimationFlags();
            onExitLayoutOrScroll();

            if (mState.mRunPredictiveAnimations) {
                mState.mInPreLayout = true;
            } else {
                // consume remaining updates to provide a consistent state with the layout pass.
                mAdapterHelper.consumeUpdatesInOnePass();
                mState.mInPreLayout = false;
            }
            mAdapterUpdateDuringMeasure = false;
            stopInterceptRequestLayout(false);
        } else if (mState.mRunPredictiveAnimations) {
            // If mAdapterUpdateDuringMeasure is false and mRunPredictiveAnimations is true:
            // this means there is already an onMeasure() call performed to handle the pending
            // adapter change, two onMeasure() calls can happen if RV is a child of LinearLayout
            // with layout_width=MATCH_PARENT. RV cannot call LM.onMeasure() second time
            // because getViewForPosition() will crash when LM uses a child to measure.
            setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
            return;
        }

        if (mAdapter != null) {
            mState.mItemCount = mAdapter.getItemCount();
        } else {
            mState.mItemCount = 0;
        }
        startInterceptRequestLayout();
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        stopInterceptRequestLayout(false);
        mState.mInPreLayout = false; // clear
    }
}

// 如果没有设置layoutManager，则采用padding+最小尺寸
void defaultOnMeasure(int widthSpec, int heightSpec) {
    final int width = LayoutManager.chooseSize(widthSpec,getPaddingLeft() + getPaddingRight(),
            ViewCompat.getMinimumWidth(this));
    final int height = LayoutManager.chooseSize(heightSpec,getPaddingTop() + getPaddingBottom(),
            ViewCompat.getMinimumHeight(this));
    setMeasuredDimension(width, height);
}
```

### Measure两次的原因

```java
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
 
int width = host.getMeasuredWidth();
int height = host.getMeasuredHeight();

//是否需要再次测量，默认为false
boolean measureAgain = false;
 
if (lp.horizontalWeight > 0.0f) {
        width += (int) ((mWidth - width) * lp.horizontalWeight);
        childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
        MeasureSpec.EXACTLY);
        measureAgain = true;
}
if (lp.verticalWeight > 0.0f) {
       height += (int) ((mHeight - height) * lp.verticalWeight);
       childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
       MeasureSpec.EXACTLY);
       measureAgain = true;
}
 
if (measureAgain) {
    if (DEBUG_LAYOUT) Log.v(mTag,
            "And hey let's measure once more: width=" + width + " height=" + height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

在第一次 performMeasure() 方法调用后， **如果子View 需要的空间大于父容器为它测量的大小**，那么对应的 verticalWeight 和 horizontalWeight 将会大于0，即这两个字段分别对应垂直和水平的情况下子 View 需要的额外空间。

这时候会将 measureAgain 设置为 true， 并且开始第二次测量。

未完待续...