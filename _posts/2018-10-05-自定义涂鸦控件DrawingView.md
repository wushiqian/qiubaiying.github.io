---
layout:     post
title:      自定义涂鸦控件DrawingView
subtitle:   android上的一个涂鸦控件
date:       2018-10-05
author:     wushiqian
header-img: img/post-bg-jay.jpg
catalog: true
tags:
    - Android
---

# 自定义涂鸦控件DrawingView

android上的一个涂鸦控件。可以设置画笔的粗细，颜色，撤销上一笔涂鸦，提供保存图片的接口。

## 基本功能

可以设置画笔的粗细，颜色，撤销上一笔涂鸦，清空涂鸦，提供保存图片的接口。

## 具体实现

### 核心代码

```
@Override
    public boolean onTouchEvent(MotionEvent event) {
        // 多个模式，检测当前模式可否绘画
        if (!mDrawMode) {
            return false;
        }
        float x;
        float y;
        if (mProportion != 0) {
            x = (event.getX()) / mProportion;
            y = event.getY() / mProportion;
        } else {
            x = event.getX();
            y = event.getY();
        }
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                // This happens when we undo a path
                if (mLastDrawPath != null) {
                    mPaint.setColor(mPaintBarPenColor);
                    mPaint.setStrokeWidth(mPaintBarPenSize);
                }
                mPath = new Path();
                mPath.reset();
                mPath.moveTo(x, y);
                mX = x;
                mY = y;
                mCanvas.drawPath(mPath, mPaint);
                break;
            case MotionEvent.ACTION_MOVE:
                float dx = Math.abs(x - mX);
                float dy = Math.abs(y - mY);
                if (dx >= TOUCH_TOLERANCE || dy >= TOUCH_TOLERANCE) {
                    mPath.quadTo(mX, mY, (x + mX) / 2, (y + mY) / 2);
                    mX = x;
                    mY = y;
                }
                mCanvas.drawPath(mPath, mPaint);
                break;
            case MotionEvent.ACTION_UP:
                mPath.lineTo(mX, mY);
                mCanvas.drawPath(mPath, mPaint);
                mLastDrawPath = new DrawPath(mPath, mPaint.getColor(), mPaint.getStrokeWidth());
                savePath.add(mLastDrawPath);
                mPath = null;
                break;
            default:
                break;
        }
        invalidate();
        return true;
    }
```

### 控件适应图片
因为这个我们需要这个控件居中显示，而且canvas必须和加载的图片一样大（否则可以涂鸦的范围和图片大小不一样）所以在绘制这个控件的时候要测量图片大小。

重写onMeasure()方法

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSize = getMeasureWidth(widthMeasureSpec);
        int heightSize = getMeasureHeight(heightMeasureSpec);
        if (mBitmap != null) {
            if ((mBitmap.getHeight() > heightSize) && (mBitmap.getHeight() > mBitmap.getWidth())) {
                widthSize = heightSize * mBitmap.getWidth() / mBitmap.getHeight();
            } else if ((mBitmap.getWidth() > widthSize) && (mBitmap.getWidth() > mBitmap.getHeight())) {
                heightSize = widthSize * mBitmap.getHeight() / mBitmap.getWidth();
            } else {
                heightSize = mBitmap.getHeight();
                widthSize = mBitmap.getWidth();
            }
        }
        setMeasuredDimension(widthSize, heightSize);  //必须调用此方法，否则会抛出异常
    }
```

### 撤销功能

创建一个列表，在每次手指抬起时（MotionEvent.ACTION_UP）记录下paint和path，需要“撤销”的时候先清空画布，然后重新加载图片，之后移除列表中的最后一笔，最后把列表中记录的paint和path重绘一次。

记录画笔和路径，注意如果你是直接保存mPaint和mPath的话，每次手指下落的时候都要新建这两个对象，不然会导致路径列表里所有路径都是一样的，因为他们保存的对象最终指向同样的内容。

这里我做了一点小改变。不保存mPaint，只保存了mPaint的两个属性，这样就不用每次new Paint()了。

```
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
            	// 手指下落 新建Path对象
                mPath = new Path();
                ...
                break;
            case MotionEvent.ACTION_MOVE:
                ...
                mCanvas.drawPath(mPath, mPaint);
                break;
            case MotionEvent.ACTION_UP:
                mPath.lineTo(mX, mY);
                mCanvas.drawPath(mPath, mPaint);
            	// 保存 path和paint的两个属性
                savePath.add(new DrawPath(mPath, mPaint.getColor(), mPaint.getStrokeWidth()));
                mPath = null;
                break;
            default:
                break;
        }
```        
```
    /**
     * 撤销上一步
     */
    public void undo() {
        Log.d(TAG, "undo: recall last path");
        if (savePath != null && savePath.size() > 0) {
            // 清空画布
            mCanvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
            loadImage(mOriginBitmap);

            savePath.removeLast();

            // 将路径保存列表中的路径重绘在画布上 遍历绘制
            for (DrawPath dp : savePath) {
                mPaint.setColor(dp.getPaintColor());
                mPaint.setStrokeWidth(dp.getPaintWidth());
                mCanvas.drawPath(dp.path, mPaint);
            }
            invalidate();
        }
    }
```    

### 清空画布
```
    /**
     * 清空画布
     */
    public void clear() {
        Log.d(TAG, "clear the path");
        if (savePath != null && savePath.size() > 0) {
            // 清空画布
            mCanvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
            loadImage(mOriginBitmap);
            invalidate();
        }
    }
```
## 提供的接口

```
    mDrawingView = (DrawingView) findViewById(R.id.img_screenshot);
    mDrawingView.initializePen();// 初始化画笔
    mDrawingView.setPenSize(10);// 设置画笔大小
    mDrawingView.setPenColor(getColor(R.color.red));// 设置画笔颜色
    mDrawingView.loadImage(bitmap);// 加载图片
    mDrawingView.saveImage(sdcardPath, "DrawImg", Bitmap.CompressFormat.PNG, 100);//保存图片
    mDrawingView.undo();// 撤销上一步
    mDrawingView.getImageBitmap();// 返回控件上的bitmap，可用于保存文件
```    