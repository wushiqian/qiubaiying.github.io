---
layout:     post
title:      自定义可旋转、平移、缩放的ImageView
subtitle:   android自定义控件
date:       2018-10-06
author:     wushiqian
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - Android
---

# 自定义可旋转、平移、缩放的ImageView

## 实现原理概览

我们要实现手指控制图片的平移、旋转、缩放，首先得知道手指做了什么动作，比如用户两指间距离是变大还是变小，两指是否做了移动，只有获取到了用户的手势才可以根据手势执行相应了变换，这部分内容下文的多点触控原理与获取触摸事件两部分会进行介绍。获取到手势后我们就可以通过Matrix对图片进行相应的变换了。

## Matrix原理

如果没有通过Matrix类的API而是直接对其中的3×3矩阵进行操作需要注意下面两点

* 使用Matrix需要注意的是缩放操作不仅会影响MatrixValue中MSCALE_X和MSCALE_Y的值，还会影响到MTRANS_X和MTRANS_Y两个位置的值。

* 旋转操作则3×3矩阵的上面两行6个位置的值都会受到影响

* postXXX()为M' = other * M而preXXX()为M' = M * other,需要注意两者的计算顺序不同，结果也不同

## 多点触控原理

多点触控需要注意一下几点

* 通过getActionMasked()获取事件类型

* 各事件的触发条件

事件    |   简介   
---   |   ---    
ACTION_DOWN   |   第一个手指初次接触到屏幕时触发,后续的手指按下只会触发ACTION_POINTER_DOWN 
ACTION_MOVE	   |   手指在屏幕上滑动时触发，会多次触发   
ACTION_UP   |   最后一个手指离开屏幕时触发
ACTION_POINTER_DOWN   | 有非主要的手指按下(即按下之前已经有手指在屏幕上)，如果从始至终只有一个手指则不会触发此事件
ACTION_POINTER_UP    |  有非主要的手指抬起(即抬起之后仍然有手指在屏幕上)，如果从始至终只有一个手指则不会触发此事件
ACTION_CANCEL     |当我们手指还在屏幕上时，但是却因为某种原因导致事件流中断，如锁屏等，就会触发该事件，应将该事件作为一个UP事件对待

* Index 会变化，pointId 始终不变   

## 基本思路

由于是要自定义一个对图片进行操作的View，所以既可以使用View作为父类，也可以使用ImageView作为父类，由于ImageView提供了很多图片处理相关方法，还有各种图片加载库支持直接load图片进入ImageView，也不用自己处理View的测量布局绘制，可以少做很多事情，所以这里选择继承ImageView

接着就是判断当前的手势是否符合预先设置的旋转、平移、缩放手势，此处可以使用ScaleGestureDetector、GestureDetector两个系统提供的手势帮助类，也可以选则自己在onTouchEvent()方法中对触摸事件进行判断。这里我是在onTouchEvent()方法中对触摸事件进行判断。

获取到用户的手势后即可对图片进行操作，ImageView有提供方法setImageMatrix()非常方便，所以此处选择使用Matrix直接对图片进行操作

### 获取触摸事件

这边因为需要在控件的长按事件与点击事件做一些操作，在onTouch中进行判断直接返回true的话会导致这两个事件无法触发，所以选择在onTouchEvent()中对触摸事件进行判断

方法中主要是判断当前触摸的手指数为2的时候可以缩放、旋转，为1的时候可以平移，手指全部抬起后进行回弹操作，然后对一些成员变量进行维护，不用太关注具体变量什么意思，等下会和其他具体的手势判断方法在下文一起进行介绍

此处需要注意具体缩放、平移、旋转方法中只改变mMatrix的值，而将mMatrix应用到图片上只在onTouchEvent()或者动画中进行

```
@Override
public boolean onTouchEvent(MotionEvent event) {
    // 获取所有触点的中点
    PointF midPoint = getMidPointOfFinger(event);
    switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
        case MotionEvent.ACTION_POINTER_DOWN:
            // 每次触摸事件开始都初始化mLastMidPonit
            mLastMidPoint.set(midPoint);
            isTransforming = false;
            mRevertAnimator.cancel();
            // 新手指落下则需要重新判断是否可以对图片进行变换
            mCanRotate = false;
            mCanScale = false;
            mCanDrag = false;
            if (event.getPointerCount() == 2) {
                // 旋转、平移、缩放分别使用三个判断变量，避免后期某个操作执行条件改变
                mCanScale = true;
                mLastPoint1.set(event.getX(0), event.getY(0));
                mLastPoint2.set(event.getX(1), event.getY(1));
                mCanRotate = true;
                mLastVector.set(event.getX(1) - event.getX(0),
                        event.getY(1) - event.getY(0));
            } else if(event.getPointerCount() == 1) {
                mCanDrag = true;
            }
            break;
        case MotionEvent.ACTION_MOVE:
            if (mCanDrag) translate(midPoint);
            if (mCanScale) scale(event);
            if (mCanRotate) rotate(event);
            // 判断图片是否发生了变换
            if (!getImageMatrix().equals(mMatrix)) isTransforming = true;
            if (mCanDrag || mCanScale || mCanRotate) applyMatrix();
            break;
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_CANCEL:
            // 检测是否需要回弹，该变量可从xml文件中配置，默认开启
            if(mRevert) {
                mMatrix.getValues(mFromMatrixValue);/*设置矩阵动画初始值*/
                /* 旋转和缩放都会影响矩阵，进而影响后续需要使用到ImageRect的地方，
                 * 所以检测顺序不能改变
                 */
                checkRotation();
                checkScale();
                checkBorder();
                mMatrix.getValues(mToMatrixValue);/*设置矩阵动画结束值*/
                // 启动回弹动画
                mRevertAnimator.setMatrixValue(mFromMatrixValue, mToMatrixValue);
                mRevertAnimator.cancel();
                mRevertAnimator.start();
            }
        case MotionEvent.ACTION_POINTER_UP:
            mCanScale = false;
            mCanDrag = false;
            mCanRotate = false;
            break;
    }
    // 调用父类的onTouchEvent()方法，方便点击和长按的调用
    super.onTouchEvent(event);
    return true;
}
```

## 具体手势判断及实现

这部分会介绍如何根据手势获取平移的距离、旋转的角度、缩放的比例，相关成员变量的意义也会在此处进行说明，这边需要注意的是由于都是调用Matrix的postXXX()方法，所以虽然每次触摸只平移、旋转、缩放一点点，但是所有变化会在Matrix中累积。还有一点因为触摸事件会频繁触发，所以一些方法中本可以使用局部变量的却使用了全局变量，避免频繁创建对象

### 平移
直接计算当前触摸事件所有触点的中点与上次触摸事件中点的距离

```
private void translate(PointF midPoint) {
    // 分别计算在x轴与y轴上需要平移的距离
    float dx = midPoint.x - mLastMidPoint.x;
    float dy = midPoint.y - mLastMidPoint.y;
    mMatrix.postTranslate(dx, dy);
    // 更新最后一次触摸事件中点为本次事件中点
    mLastMidPoint.set(midPoint);
}
/**
 * 计算所有触点的中点
 * @param event 当前触摸事件
 * @return 本次触摸事件所有触点的中点
 */
private PointF getMidPointOfFinger(MotionEvent event) {
    /* 初始化当前触摸事件中点mCurrentMidPoint，由于触摸事件频繁触发，所以此处选择使用一个成员变量，避免频繁创建对象*/
    mCurrentMidPoint.set(0f, 0f);
    // 计算当前中点坐标
    int pointerCount = event.getPointerCount();
    for (int i = 0; i < pointerCount; i++) {
        mCurrentMidPoint.x += event.getX(i);
        mCurrentMidPoint.y += event.getY(i);
    }
    mCurrentMidPoint.x /= pointerCount;
    mCurrentMidPoint.y /= pointerCount;
    return mCurrentMidPoint;
}
```

### 缩放

以当前两指间距离与上次触摸事件的两指间距离之比作为图片的缩放比例

```
	/**
     * 获取图片的缩放中心，mScaleBy可在外部设置，或通过xml文件设置
     * 默认中心点为图片中心
     * @return 图片的缩放中心点
     */
    private PointF getScaleCenter() {
        // 使用全局变量避免频繁创建变量
        switch (mScaleBy) {
            case SCALE_BY_IMAGE_CENTER:
                // mImageRect为保存图片位置的RectF矩形
                scaleCenter.set(mImageRect.centerX(), mImageRect.centerY());
                break;
            case SCALE_BY_FINGER_MID_POINT:
                // mLastMidPoint 为最后一次触摸事件的中点
                scaleCenter.set(mLastMidPoint.x, mLastMidPoint.y);
                break;
        }
        return scaleCenter;
    }

    private void scale(MotionEvent event) {
        // 获取缩放中心
        PointF scaleCenter = getScaleCenter();

        // 初始化当前两指触点
        mCurrentPoint1.set(event.getX(0), event.getY(0));
        mCurrentPoint2.set(event.getX(1), event.getY(1));
        // 计算缩放比例
        float scaleFactor = distance(mCurrentPoint1, mCurrentPoint2)
                / distance(mLastPoint1, mLastPoint2);

        /* mScaleFactor中保存了当前图片总的缩放比例更新当前图片的缩放比例*/
        mScaleFactor *= scaleFactor;

        // 更新矩阵的值
        mMatrix.postScale(scaleFactor, scaleFactor,
                scaleCenter.x, scaleCenter.y);
        // 更新最后一次触摸事件的两个触点为当前事件的两个触点
        mLastPoint1.set(mCurrentPoint1);
        mLastPoint2.set(mCurrentPoint2);
    }

    /**
     * 获取两点间距离
     */
    private float distance(PointF point1, PointF point2) {
        float dx = point2.x - point1.x;
        float dy = point2.y - point1.y;
        return (float) Math.sqrt(dx * dx + dy * dy);
    }
```

### 旋转

首先需要保存上一次触摸事件两个手指触点连线所表示的向量，然后获取当前两指在屏幕上触点所表示的向量。通过向量的叉乘公式sinAB = |A×B|/|A|*|B|获取两向量夹角，若arcsin(sinAB)为正则为顺时针旋转，否则为逆时针旋转

之前是使用上面这个方法求旋转角度的，后来发现有个问题，由于每次ACTINON_MOVE触发间隔较短，转过的角度较小所以使用该方法不会出错，但是若在一次事件内旋转超过90或者-90度则会导致旋转错位。所以更改为使用如下方法

使用Math.atan2(y, x), 可以很方便的求出某个点(x,y)与x轴的夹角，于是我们先求出上次两指连线所表示的向量与x轴夹角与当前两指连线所表示的向量与x轴夹角之差，即为当前需要转过的角度

![旋转角度](https://upload-images.jianshu.io/upload_images/2704468-cd3222dcff573417.png?imageMogr2/auto-orient/)

```
private void rotate(MotionEvent event) {
        // 计算当前两指触点所表示的向量
        mCurrentVector.set(event.getX(1) - event.getX(0),
                event.getY(1) - event.getY(0));
        // 获取旋转角度
        float degree = getRotateDegree(mLastVector, mCurrentVector);
        // 更新矩阵的值
        mMatrix.postRotate(degree, mImageRect.centerX(), mImageRect.centerY());
        // 设置当前向量为最后一次事件的向量
        mLastVector.set(mCurrentVector);
    }

    /**
     * 使用Math#atan2(double y, double x)方法求上次触摸事件两指所示向量与x轴的夹角，
     * 再求出本次触摸事件两指所示向量与x轴夹角，最后求出两角之差即为图片需要转过的角度
     *
     * @param lastVector 上次触摸事件两指间连线所表示的向量
     * @param currentVector 本次触摸事件两指间连线所表示的向量
     * @return 两向量夹角，单位“度”，顺时针旋转时为正数，逆时针旋转时返回负数
     */
    private float getRotateDegree(PointF lastVector, PointF currentVector) {
        //上次触摸事件向量与x轴夹角
        double lastRad = Math.atan2(lastVector.y, lastVector.x);
        //当前触摸事件向量与x轴夹角
        double currentRad = Math.atan2(currentVector.y, currentVector.x);
        // 两向量与x轴夹角之差即为需要旋转的角度
        double rad = currentRad - lastRad;
        //“弧度”转“度”
        return (float) Math.toDegrees(rad);
    }
}
```

## 设置回弹及动画

为了控制图片保持在某个区域、某个角度，以及控制图片的缩放大小不能超过我们设置的上限和下限，就需要设置回弹，直接设置超过了某个临界值就不能操作也能达到目的，但是体验不好，会有卡顿的感觉，而使用动画加上回弹可以使操作更加顺滑

这边有个点需要注意的是由于旋转会改变mMatrix的3×3矩阵上两行的值，缩放会改变矩阵和平移相关的两个值，所以在回弹的时候应先检测角度是否需要回弹，再检测缩放，最后检查平移回弹，顺序不能乱。

### 旋转回弹

此处以让角度只能为0、90、180、270四个值为例，介绍一下如何判断是否需要对角度进行回弹，及如何回弹

此处还有个坑，原本我是使用一个float类型的全局变量mDegree保存当前图片已转过的角度，但是回弹的时候会因为计算过程中浮点数的精度损失而导致图片不能完全转回去，和控件之间还是有一点微小的夹角，于是之后改为即时计算当前图片转过的角度的方式就不会再出现这个问题了

```
/**
 * 根据当前图片旋转的角度，判断是否回弹
 */
private void checkRotation() {
    // 获取当前图片已经转过的角度
    float currentDegree = getCurrentRotateDegree();
    float degree = currentDegree;
    // 根据当前图片旋转的角度值所在区间，判断要转到几度
    // 取绝对值可以使(-180, 0]与(0,180]两个区间的角度统一判断
    degree = Math.abs(degree);
    if (degree > 45 && degree <= 135) {
        degree = 90;
    } else if(degree > 135 && degree <= 225) {
        degree = 180;
    } else if(degree > 225 && degree <= 315) {
        degree = 270;
    } else {
        degree = 0;
    }
    // 判断顺时针还是逆时针旋转
    degree = currentDegree < 0 ? -degree : degree;
    // 更新矩阵的值
    mMatrix.postRotate(degree - currentDegree, mImageRect.centerX(), mImageRect.centerY());
}

private float[] xAxis = new float[]{1f, 0f}; // 表示与x轴同方向的向量
/**
 * 获取当前图片旋转角度
 * @return 图片当前的旋转角度
 */
private float getCurrentRotateDegree() {
    // 每次重置初始向量的值为与x轴同向
    xAxis[0] = 1f;
    xAxis[1] = 0f;
    // 初始向量通过矩阵变换后的向量
    mMatrix.mapVectors(xAxis);
    // 变换后向量与x轴夹角即为当前图片已经转过的角度
    double rad = Math.atan2(xAxis[1], xAxis[0]);
    return (float) Math.toDegrees(rad);
}
```

### 缩放回弹

首先需要判断图片当前是否与初始时的时候角度相同（或者转过了180度，将这个位置定义为水平），或者当前的旋转角度为90或者-90度(定义此时为垂直状态)，
根据判断结果决定使用水平时的最小缩放比例或者垂直时的最小缩放比例

接着判断当前图片的缩放比例是否小于最小缩放比例，或者大于最大缩放比例，若超过了则进行回弹

最大缩放比例可通过外部设置，最小缩放比例即为适应控件大小时的缩放比例

```
	/**
     * 检查图片缩放比例是否超过设置的大小
     */
    private void checkScale() {
        // 获取缩放中心
        PointF scaleCenter = getScaleCenter();

        // 默认不进行回弹
        float scaleFactor = 1.0f;

        // 获取图片当前是水平还是垂直
        int imgOrientation = imgOrientation();
        // 超过设置的上限或下限则回弹到设置的最大或最小值
        // 除以当前图片缩放比例mScaleFactor，postScale()方法执行后的图片的缩放比例即为被除数大小
        if (imgOrientation == HORIZONTAL
                && mScaleFactor < mHorizontalMinScaleFactor) {
            scaleFactor = mHorizontalMinScaleFactor / mScaleFactor;
        } else if (imgOrientation == VERTICAL
                && mScaleFactor < mVerticalMinScaleFactor) {
            scaleFactor = mVerticalMinScaleFactor / mScaleFactor;
        }else if(mScaleFactor > mMaxScaleFactor) {
            scaleFactor = mMaxScaleFactor / mScaleFactor;
        }

        // 更新矩阵的值
        mMatrix.postScale(scaleFactor, scaleFactor, scaleCenter.x, scaleCenter.y);
        // 更新图片当前的缩放比例
        mScaleFactor *= scaleFactor;
    }

    private static final int HORIZONTAL = 0;
    private static final int VERTICAL = 1;

    /**
     * 判断图片当前是水平还是垂直
     * @return 水平则返回 {@code HORIZONTAL}，垂直则返回 {@code VERTICAL}
     */
    private int imgOrientation() {
        // 获取图片当前的旋转角度的绝对值
        float degree = Math.abs(getCurrentRotateDegree());
        int orientation = HORIZONTAL;
        // 当前图片旋转角度在[-135, -45)或(45, 135]之间即为垂直状态
        if (degree > 45f && degree <= 135f) {
            orientation = VERTICAL;
        }
        return orientation;
    }
```

### 平移回弹

图片宽或高小于控件时回弹到控件中心，大于控件时则与控件之间不能有空隙，不符合条件则进行回弹

此处需要注意mImageRect所使用的坐标系与View的坐标系不一致

![customImageView和mImageRect使用不同坐标系](https://upload-images.jianshu.io/upload_images/2704468-7b30b9192e0ed2e7.png?imageMogr2/auto-orient/)

```
	/**
     * 将图片移回控件中心
     */
    private void checkBorder() {
        // 由于旋转回弹与缩放回弹会影响图片所在位置，所以此处需要更新ImageRect的值
        refreshImageRect();
        // 默认不移动
        float dx = 0f;
        float dy = 0f;

        // mImageRect中的坐标值为相对View的值
        // 图片宽大于控件时图片与控件之间不能有白边
        if (mImageRect.width() > getWidth()) {
            if (mImageRect.left > 0) {/*判断图片左边界与控件之间是否有空隙*/
                dx = -mImageRect.left;
            } else if(mImageRect.right < getWidth()) {/*判断图片右边界与控件之间是否有空隙*/
                dx = getWidth() - mImageRect.right;
            }
        } else {/*宽小于控件则移动到中心*/
            dx = getWidth() / 2 - mImageRect.centerX();
        }

        // 图片高大于控件时图片与控件之间不能有白边
        if (mImageRect.height() > getHeight()) {
            if (mImageRect.top > 0) {/*判断图片上边界与控件之间是否有空隙*/
                dy = -mImageRect.top;
            } else if(mImageRect.bottom < getHeight()) {/*判断图片下边界与控件之间是否有空隙*/
                dy = getHeight() - mImageRect.bottom;
            }
        } else {/*高小于控件则移动到中心*/
            dy = getHeight() / 2 - mImageRect.centerY();
        }
        mMatrix.postTranslate(dx, dy);
    }
```

## 动画技巧

设置好初始状态的矩阵和最终状态的矩阵，就可以根据动画当前执行的进度对矩阵的值进行更新，简单粗暴

```@Override
public void onAnimationUpdate(ValueAnimator animation) {
    /*mFromMatrixValue为初始矩阵， mToMatrixValue为最终矩阵，mInterpolateMatrixValue为保存动画过程中中间值的矩阵*/
    if (mFromMatrixValue != null
            && mToMatrixValue != null 
            && mInterpolateMatrixValue != null) {
        // 根据动画当前进度设置矩阵的值
        for (int i = 0; i < 9; i++) {
            float animatedValue = (float) animation.getAnimatedValue();
            mInterpolateMatrixValue[i] = mFromMatrixValue[i]
                            + (mToMatrixValue[i] - mFromMatrixValue[i]) * animatedValue;
        }
        mMatrix.setValues(mInterpolateMatrixValue);
        applyMatrix();
    }
}
```

## 辅助方法

```
/**
 * 更新图片所在区域，并将矩阵应用到图片
 */
protected void applyMatrix() {
    refreshImageRect(); /*将矩阵映射到ImageRect*/
    setImageMatrix(mMatrix);
}

/**
 * 图片使用矩阵变换后，刷新图片所对应的mImageRect所指示的区域
 */
private void refreshImageRect() {
    if (getDrawable() != null) {
        mImageRect.set(getDrawable().getBounds());
        mMatrix.mapRect(mImageRect, mImageRect);
    }
}
```

## 总结

自定义可平移、旋转、缩放的ImageView，整个过程不会特别复杂，主要是对手势的识别，计算旋转角度、平移距离、缩放比例三个值，还有对矩阵的操作，过程中要注意矩阵三个操作执行后内部3×3矩阵的变化，及三个操作如何互相影响，需要注意先后顺序，最后将矩阵应用到图片上就可以对图片进行变换了


