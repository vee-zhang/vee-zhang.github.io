---
title: Android自定义ViewGroup
date: 2021-07-12 17:49:10
tags: [Android]
---

自定义ViewGroup与自定义View不同，一般不需要重写`onDraw`，而是需要重写`onLayout`。

```kotlin
import android.content.Context
import android.util.AttributeSet
import android.util.Log
import android.view.MotionEvent
import android.view.View
import android.view.ViewGroup
import kotlin.math.max

class MyViewGroup(context: Context, attrs: AttributeSet?, defStyleAttr: Int, defStyleRes: Int) :
    ViewGroup(context, attrs, defStyleAttr, defStyleRes) {

    constructor(context: Context, attrs: AttributeSet?, defStyleAttr: Int) : this(context, attrs, defStyleAttr, 0)
    constructor(context: Context, attrs: AttributeSet?) : this(context, attrs, 0)
    constructor(context: Context) : this(context, null)

    private val TAG = "测试"

    override fun generateLayoutParams(attrs: AttributeSet?): LayoutParams {
        return MarginLayoutParams(context, attrs)
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {

        //测量子view
        measureChildren(widthMeasureSpec, heightMeasureSpec)

        var width1 = 0
        var width2 = 0
        var height1 = 0
        var height2 = 0

        var child: View
        var childWidth: Int
        var childHeight: Int
        var childLp: LayoutParams

        //叠加所有子View的宽高
        for (i in 0 until childCount) {
            child = getChildAt(i)
            childWidth = child.measuredWidth
            childHeight = child.measuredHeight
            childLp = child.layoutParams as MarginLayoutParams

            if (i < 2) {
                width1 += childWidth + childLp.leftMargin + childLp.rightMargin
                height1 += childHeight + childLp.topMargin + childLp.bottomMargin
            } else {
                width2 += childWidth + childLp.leftMargin + childLp.rightMargin
                height2 += childHeight + childLp.topMargin + childLp.bottomMargin
            }

        }

        // 获取父容器推荐的尺寸
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val measureWidth = MeasureSpec.getSize(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val measureHeight = MeasureSpec.getSize(heightMeasureSpec)

        // 根据情况计算出可容纳所有child的尺寸
        val width = if (widthMode == MeasureSpec.EXACTLY) measureWidth else max(width1, width2)
        val height = if (heightMode == MeasureSpec.EXACTLY) measureHeight else max(height1, height2)

        setMeasuredDimension(width, height)
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {

        var childLp: LayoutParams
        var child: View
        var cl: Int
        var ct: Int
        var cr: Int
        var cb: Int

        // 依次为所有的child布局
        for (i in 0 until childCount) {
            child = getChildAt(i)
            childLp = child.layoutParams as MarginLayoutParams
            when (i) {
                0 -> {
                    cl = 0 + childLp.leftMargin
                    ct = 0 + childLp.topMargin
                }
                1 -> {
                    cl = width - child.measuredWidth - childLp.leftMargin - childLp.rightMargin
                    ct = 0 + childLp.topMargin
                }
                2 -> {
                    cl = 0 + childLp.leftMargin
                    ct = height - child.measuredHeight - childLp.topMargin - childLp.bottomMargin
                }
                else -> {
                    cl = width - child.measuredWidth - childLp.leftMargin - childLp.rightMargin
                    ct = height - child.measuredHeight - childLp.topMargin - childLp.bottomMargin
                }
            }
            cr = cl + child.measuredWidth
            cb = ct + child.measuredHeight

            //布局
            child.layout(cl, ct, cr, cb)
        }
    }
}
```