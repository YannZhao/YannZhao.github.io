---
layout: post
title:  "自定义OverlapViewPager"
desc: "拥有三个page页的自定义View"
keywords: "Custom,Viewpage,Overlap"
date: 2016-12-08
categories: [Android]
tags: [android, Customize, ViewPager]
---
### 写在前面
Overlap，顾名思义，是一个page可以重叠的的ViewPager，一共有左中右三页，中间页固定，左右两边的page分别可以划出重叠到中间。话不多说，下面开始。

### 实现思路
自定义一个ViewGroup，往其中添加3个child；按照左中右的顺序，在onLayout中，设置好children的位置；在从左往右划动的时候，找到被drag的child，通过Scroller，根据手指的滑动距离，把对应的child移动相应的位置；有对滑动速度的检测，超过阈值就触发完全覆盖，以及对松手时滑动距离的阈值检测，判断是还原还是覆盖。见下图：

<img src="/static/img/blog/overlapviewpager/layout.png" width="50%" height="50%">


#### 1.一些重要的常量
```java
    public static final int PAGE_LEFT = -1; //left page 标识
    
    public static final int PAGE_MIDDLE = 0;//middle page 标识
    
    public static final int PAGE_RIGHT = 1;//right page 标识
    
    private static final int PAGE_COUNT = 3;// page数量
    
    public static final int VIEW_INDEX_LEFT = 0;//left child 下标
    
    public static final int VIEW_INDEX_MIDDLE = 1;//middle child 下标
    
    public static final int VIEW_INDEX_RIGHT = 2;//middle child 下标
    
    private int downX = 0;
    
    private int mCurrentFront = PAGE_MIDDLE;//默认当前页为middle
    
    private int mCurrentMovedItemIndex = Integer.MIN_VALUE;
```
#### 2.添加child
```java
public void addViews(View[] views) {
    removeAllViews();
    this.views = views;
    int seq[] = { 1, 0, 2 };
    for (int i : seq) {
        if (views[i] != null) {
            addView(views[i], new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT));
        }
    }
}
```
对应在onLayout时注意顺序，代码如下：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int height = b - t;
    int width = r - l;
    //根据当前页不同，应该按照不同顺序layout
    if (mCurrentFront == PAGE_MIDDLE) {
        for (int i = 0; i < 3; ++i) {
            View viewItem = views[i];
            if (viewItem != null) {
                viewItem.layout(0 - width + i * width, 0, i * width, height);
            }
        }
    } else if (mCurrentFront == PAGE_LEFT) {
        for (int i = 0; i < PAGE_COUNT; i++) {
            View viewItem = views[i];
            if (viewItem != null) {
                if (i == VIEW_INDEX_MIDDLE) {
                    viewItem.layout(0, 0, width, height);
                } else if (i == VIEW_INDEX_RIGHT) {
                    viewItem.layout(width, 0, i * width, height);
                } else if (i == VIEW_INDEX_LEFT) {
                    viewItem.layout(0, 0, width, height);
                }
            }
        }
    } else if (mCurrentFront == PAGE_RIGHT) {
        for (int i = 0; i < PAGE_COUNT; i++) {
            View viewItem = views[i];
            if (viewItem != null) {
                if (i == VIEW_INDEX_MIDDLE) {
                    viewItem.layout(0, 0, width, height);
                } else if (i == VIEW_INDEX_RIGHT) {
                    viewItem.layout(0, 0, width, height);
                } else if (i == VIEW_INDEX_LEFT) {
                    viewItem.layout(0 - width, 0, 0, height);
                }
            }
        }
    }
}
```
#### 	3.滑动
当前页是中间页时，在onTouchEvent中根据x轴移动数据判断滑动方向、滑动的child以及计算出offset，对应的view再进行移动。

```java
...
int deltaX = (int) event.getX() - downX;
int offset = (int) event.getX() - lastPointX;
lastPointX = (int) event.getX();
switch (mCurrentFront) {
    case PAGE_LEFT:
    ...
    case PAGE_MIDDLE:
        if (deltaX > 0) {
            mCurrentMovedItemIndex = VIEW_INDEX_LEFT;
        } else if (deltaX < 0) {
            mCurrentMovedItemIndex = VIEW_INDEX_RIGHT;
        } else {
            break;
        }
        dragLayout = getDragView(mCurrentMovedItemIndex);
        if (dragLayout != null) {
            if (mCurrentMovedItemIndex == VIEW_INDEX_LEFT) {
                if (dragLayout.getRight() >= 0 && dragLayout.getRight() <= getWidth()) {
                    dragLayout.offsetLeftAndRight(offset);
                    invalidate();
                }
            } else if (mCurrentMovedItemIndex == VIEW_INDEX_RIGHT) {
                if (dragLayout.getLeft() >= 0 && dragLayout.getLeft() <= getWidth()) {
                    dragLayout.offsetLeftAndRight(offset);
                    invalidate();
                }
            }
        }
        checkToResetLastMovedView(mCurrentMovedItemIndex);
        break;
    case PAGE_RIGHT:
        ...
}
```

写到这，主要的内容就完成了，可以对这个自定义的ViewPager中做很多拓展，在公司的一些功能开发中已用到，就不赘述了。

[代码地址](https://github.com/YannZhao/OverlapViewPager)

