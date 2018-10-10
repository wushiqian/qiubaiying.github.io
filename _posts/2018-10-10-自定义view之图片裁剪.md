---
layout:     post
title:      自定义view之图片裁剪
subtitle:   android自定义控件
date:       2018-10-10
author:     wushiqian
header-img: img/post-bg-coffee.jpg
catalog: true
tags:
    - Android
---

# 自定义view之图片裁剪

参考了github上一个封装好的裁剪库cropper，不过因为github这个裁剪库比较大，而且提供的功能好多都用不到，在研究完其实现的基本思路之后，修改了一下，满足了项目裁剪的要求。

<img src="https://ws2.sinaimg.cn/large/006tNbRwgy1fw31a4sq38j30g80wawvz.jpg" width = "400" height = "800" alt="title" align=center />

## 实现原理

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fw3180q3xoj30bn082t8y.jpg)

如上图所示绿色背景假设为待裁剪的图片，红色边框为裁剪框。A，B,C,D四个黑点是手指相对于裁剪框可能出现的位置。那么A,B,C,D四个黑点周围的虚圆圈是什么意思呢？圆圈的意思就是说以手指为圆心，圆圈周边如果与裁剪框上下左右四条边任一一条边相交，则表明手指此时可以对裁剪框进行操作，比如缩放，拉伸，平移等等。

手指位于A 位置：表明手指是对裁剪框的四个角某一个角进行拉伸缩放操作。此时随着手指的移动，裁剪框的四条边的长度都会发生变化：要么同时缩小，要么同时放大。

手指位于C位置：表明手指操作的是裁剪框的上下左右四条边的某一条边，此时随着手指的左右移动，比如手指在如图所示C的位置，那么手指向左移动的时候，上下两条边会缩短；向右移动的时候，上下两条边被拉长。

手指位于B位置：说明此时手指处于裁剪框的内部，此时随着手指的移动，裁剪框也会跟着移动。也即是此时是对裁剪框进行拖动操作。

当然当手指位于D位置的时候：都远离裁剪框了，当然do nothing了.

## 具体实现

一个图片对象(ImageView),一个裁剪框对象（CropWindow),五个手指按下的位置对象（为同一类的五个实例）。

而构成裁剪框的四个要素如下图四个边（LEFT，TOP，BOTTOM，RIGHT）所示： 

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fw31b2gl01j30b80aot9e.jpg)

正如上图所示一个裁剪框有四条边，在cropper的实现中讲这四条边用一个对象来表示（枚举类）：

```
public enum Edge {
    //对应着上图中的LEFT，TOP，RIGHT，BOTTOM四条边
    LEFT,
    TOP,
    RIGHT,
    BOTTOM;

    //上下左右边界的的坐标值，比如LEFT，RIGHT两条边的值是对应的边距离图片最左边的距离
    private float mCoordinate;

    //初始化裁剪框时 
    public void initCoordinate(float coordinate) {
        mCoordinate = coordinate;
    }
  ｝
```

而裁剪框的基本功能就是随着手指的移动而动态改变四条边对应的坐标mCoordinate！！！

根据第二幅图的情况我们知道手指操控的裁剪框的方位： 
四个角（对应图二中的A位置）：（LEFT，TOP），（TOP，RIGHT），（RIGNT，BOTTOM），（BOTTOM，LEFT）  
四条边（对应图二中国的C位置）：LEFT，RIGHT，BOTTOM，TOP 
中间位置Center

```
/**
 * 表示手指选中的裁剪框的哪一个边：有如下几种情况：
 * 手指选中一条边的情况：LEFT,TOP,RIGHT,BOTTOM
 * 手指选中两条边的情况：此时手指位于裁剪框的四个角度的某一个：LEFT and TOP, TOP and RIGHT, RIGHT and BOTTOM, BOTTOM and RIGHT
 * 手指在裁剪框的中间区域，此时移动手指进行的是平移操作
 */
public enum CropWindowEdgeSelector {

    //////////////对应图2 A点///////////////////////////

    //左上角：此时是控制裁剪框最上边和最左边的两条边
    TOP_LEFT(new CropWindowScaleHelper(Edge.TOP, Edge.LEFT)),

    //右上角：此时是控制裁剪框最上边和最右边的两条边
    TOP_RIGHT(new CropWindowScaleHelper(Edge.TOP, Edge.RIGHT)),

    //左下角：此时是控制裁剪框最下边和最左边的两条边
    BOTTOM_LEFT(new CropWindowScaleHelper(Edge.BOTTOM, Edge.LEFT)),

    //右下角：此时是控制裁剪框最下边和最右边的两条边
    BOTTOM_RIGHT(new CropWindowScaleHelper(Edge.BOTTOM, Edge.RIGHT)),

     //////////////对应图2 C点///////////////////////////

    //仅控制裁剪框左边线
    LEFT(new CropWindowScaleHelper(null, Edge.LEFT)),

    //仅控制裁剪框右边线
    TOP(new CropWindowScaleHelper(Edge.TOP, null)),

    //仅控制裁剪框上边线
    RIGHT(new CropWindowScaleHelper(null, Edge.RIGHT)),

    //仅控制裁剪框下边线
    BOTTOM(new CropWindowScaleHelper(Edge.BOTTOM, null)),

   //////////////对应图2 B点///////////////////////////

    //中间位置
    CENTER(new CropWindowMoveHelper());
   ｝
```
注意上面这部分代码在ACTION_DOWN事件种会根据手指的位置来返回具体的对象，比如手指在裁剪框中间（B位置）就返回上面的CENTER对象。

### 裁剪框的具体实现：

初始化裁剪框的大小，或者说什么时候初始化？

纵观View的绘制流程，在View 的onLayout方法方法里面比较合适，首先获取ImageView的范围大小，然后用ImageView的返回大小来初始化裁剪框的大小：

```
 protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

        super.onLayout(changed, left, top, right, bottom);
        //获取图片的范围RectF 
        mBitmapRect = getBitmapRect();
        //初始化裁剪框的大小
        initCropWindow(mBitmapRect);
    }
```

让我们看看裁剪框的初始化都做了些什么：

```
 /**
     * 初始化裁剪框
     *
     * @param bitmapRect
     */
    private void initCropWindow(@NonNull RectF bitmapRect) {

        //裁剪框距离图片左右的padding值
        final float horizontalPadding = 0.01f * bitmapRect.width();
        final float verticalPadding = 0.01f * bitmapRect.height();

        //初始化裁剪框上下左右四条边
        Edge.LEFT.initCoordinate(bitmapRect.left + horizontalPadding);
        Edge.TOP.initCoordinate(bitmapRect.top + verticalPadding);
        Edge.RIGHT.initCoordinate(bitmapRect.right - horizontalPadding);
        Edge.BOTTOM.initCoordinate(bitmapRect.bottom - verticalPadding);
    }
```

绘制裁剪框：根据图一我们的裁剪框有三部分内容：九宫格引导线、裁剪边框、和四个角 
具体的绘制当然是在onDraw方法里面：

```
@Override
    protected void onDraw(Canvas canvas) {

        super.onDraw(canvas);
        //绘制九宫格引导线
        drawGuidelines(canvas);
        //绘制裁剪边框
        drawBorder(canvas);
        //绘制裁剪边框的四个角
        drawCorners(canvas);
    }

```

以绘制裁剪框为例看看怎么实现的drawBorder为例

```
 private void drawBorder(@NonNull Canvas canvas) {

        canvas.drawRect(Edge.LEFT.getCoordinate(),
                Edge.TOP.getCoordinate(),
                Edge.RIGHT.getCoordinate(),
                Edge.BOTTOM.getCoordinate(),
                mBorderPaint);
    }

```

直接调用drawRect绘制一个矩形，当然矩形的四个坐标值时根据上面说的Edge枚举对象来获取的。

分析到此为止，裁剪框已经可以出现在界面中了，就差手指移动来拖动或者缩放裁剪框了，因为裁剪框时随着手指的移动而改变大小的，所以这又牵扯到类事件处理：

### 触摸事件处理

#### ACTION_DOWN

在ACTION_DOWN事件中怎么确认手指所在的位置是图二中的哪个部位。 
根据上面第二幅图的分析，我们需要确认手指按下的时候位于A，B，C，D对应的具体哪一种位置，这就需要我们在ACTION_DOWN事件中来判断，具体的判断也很简单，首先获取裁剪框的坐标位置，然后取手指按下的位置与裁剪框位置相匹配：

```
 @Override
    public boolean onTouchEvent(MotionEvent event) {

        switch (event.getAction()) {

            case MotionEvent.ACTION_DOWN:
               //处理action逻辑
                onActionDown(event.getX(), event.getY());
                return true;

            //省略部分代码
        }
    }

```

具体的onActionDown方法：

```
     /**
     * 判断手指是否的位置是否在有效的缩放区域：缩放区域的半径为targetRadius
     * 缩放区域使指：裁剪框的四个角度或者四条边，当手指位置处在某个角
     * 或者某条边的时候，则随着手指的移动对裁剪框进行缩放操作。
     * 如果手指位于裁剪框的内部，则裁剪框随着手指的移动而只进行移动操作。
     * 否则可以判定手指距离裁剪框较远而什么都不做
     */
    public static CropWindowEdgeSelector getPressedHandle(float x,
                                                          float y,
                                                          float left,
                                                          float top,
                                                          float right,
                                                          float bottom,
                                                          float targetRadius) {

        CropWindowEdgeSelector nearestCropWindowEdgeSelector = null;

        //判断手指距离裁剪框哪一个角最近

        //最近距离默认正无穷大
        float nearestDistance = Float.POSITIVE_INFINITY;
        ////判断手指是否在图二的A位置：四个角之一/////////////////

        //计算手指距离左上角的距离
        final float distanceToTopLeft = calculateDistance(x, y, left, top);
        if (distanceToTopLeft < nearestDistance) {
            nearestDistance = distanceToTopLeft;
            nearestCropWindowEdgeSelector = CropWindowEdgeSelector.TOP_LEFT;
        }


        //计算手指距离右上角的距离
        final float distanceToTopRight = calculateDistance(x, y, right, top);
        if (distanceToTopRight < nearestDistance) {
            nearestDistance = distanceToTopRight;
            nearestCropWindowEdgeSelector = CropWindowEdgeSelector.TOP_RIGHT;
        }

        //计算手指距离左下角的距离
        final float distanceToBottomLeft = calculateDistance(x, y, left, bottom);
        if (distanceToBottomLeft < nearestDistance) {
            nearestDistance = distanceToBottomLeft;
            nearestCropWindowEdgeSelector = CropWindowEdgeSelector.BOTTOM_LEFT;
        }

        //计算手指距离右下角的距离
        final float distanceToBottomRight = calculateDistance(x, y, right, bottom);
        if (distanceToBottomRight < nearestDistance) {
            nearestDistance = distanceToBottomRight;
            nearestCropWindowEdgeSelector = CropWindowEdgeSelector.BOTTOM_RIGHT;
        }

        //如果手指选中了一个最近的角，并且在缩放范围内则返回这个角
        if (nearestDistance <= targetRadius) {
            return nearestCropWindowEdgeSelector;
        }


  ///判断手指是否在图二的C位置：四个边的某条边/////////////////

        if (CatchEdgeUtil.isInHorizontalTargetZone(x, y, left, right, top, targetRadius)) {
            return CropWindowEdgeSelector.TOP;//说明手指在裁剪框top区域
        } else if (CatchEdgeUtil.isInHorizontalTargetZone(x, y, left, right, bottom, targetRadius)) {
            return CropWindowEdgeSelector.BOTTOM;//说明手指在裁剪框bottom区域
        } else if (CatchEdgeUtil.isInVerticalTargetZone(x, y, left, top, bottom, targetRadius)) {
            return CropWindowEdgeSelector.LEFT;//说明手指在裁剪框left区域
        } else if (CatchEdgeUtil.isInVerticalTargetZone(x, y, right, top, bottom, targetRadius)) {
            return CropWindowEdgeSelector.RIGHT;//说明手指在裁剪框right区域
        }


   ////判断手指是否在图二的B位置：裁剪框的中间/////////////////
        if (isWithinBounds(x, y, left, top, right, bottom)) {
            return CropWindowEdgeSelector.CENTER;
        }

  ////手指位于裁剪框的D位置，此时移动手指什么都不做/////////////
        return null;
    }

```

ACTION_DOWN事件之后我们就知道此时手指相对于裁剪框的方位，返回一个CropWindowEdgeSelector，在本文中为了方便说明假设手指位于B位置，也就是返回了CENTER对象，那么此时是时候处理ACTION_MOVE事件来随着手指的移动对裁剪框进行缩放处理了

#### ACTION_MOVE

ACTION_MOVE 事件

该事件的逻辑也很简单，就是根据手指的x,y位置来动态修改Edge.LEFT,Edege.RIGHT,Edge.BOTTOM,Edge.TOP的值，注意修改之后别忘了调用invalidate方法进行重绘操作。

```
private void onActionMove(float x, float y) {

        if (mPressedCropWindowEdgeSelector == null) {
            return;
        }

        x += mTouchOffset.x;
        y += mTouchOffset.y;


   //调用updateCropWindow方法    
     mPressedCropWindowEdgeSelector.updateCropWindow(x, y, mBitmapRect);
        //别忘了重绘
        invalidate();
    }

```

在对ACTION_DOWN的讲解种我们假设手指位于B位置，即mPressedCropWindowEdgeSelector的具体对象是CENTER这个枚举类，当手指位于B位置的时候是对裁剪框进行的拖动操作，不涉及到缩放，就让我们看看这个对象的updateCropWindow方法都做了写什么：

```
@Override
    void updateCropWindow(float x,
                          float y,
                          @NonNull RectF imageRect) {

        /*获取裁剪框的四个坐标位置*/
        float left = Edge.LEFT.getCoordinate();
        float top = Edge.TOP.getCoordinate();
        float right = Edge.RIGHT.getCoordinate();
        float bottom = Edge.BOTTOM.getCoordinate();

        /*获取裁剪框的中心位置*/
        final float currentCenterX = (left + right) / 2;
        final float currentCenterY = (top + bottom) / 2;

        /*判断手指移动的距离*/
        final float offsetX = x - currentCenterX;
        final float offsetY = y - currentCenterY;

        /*更新裁剪框四条边的坐标*/
        Edge.LEFT.offset(offsetX);
        Edge.TOP.offset(offsetY);
        Edge.RIGHT.offset(offsetX);
        Edge.BOTTOM.offset(offsetY);

        /*/////////////裁剪框越界处理////////////////*/

        /*左边越界*/
        if (Edge.LEFT.isOutsideMargin(imageRect)) {
            /*获取此时x越界时的坐标位置*/
            float currentCoordinate = Edge.LEFT.getCoordinate();

            /*重新指定左边的值为初始值*/
            Edge.LEFT.initCoordinate(imageRect.left);

            /*越界的距离*/
            float offset = Edge.LEFT.getCoordinate() - currentCoordinate;

            /*修正最右边的偏移量*/
            Edge.RIGHT.offset(offset);

       } else if (Edge.RIGHT.isOutsideMargin(imageRect))     {
        /*右边越界处理逻辑与左边越界雷同*/

        }


        if (Edge.TOP.isOutsideMargin(imageRect)) {
            /*上边越界处理逻辑与左边越界雷同*/

        } else if (Edge.BOTTOM.isOutsideMargin(imageRect)) {

        /*下边越界处理逻辑与左边越界雷同*/

        }
    }

```
上面的代码就是常规的处理拖动的逻辑：不断修改裁剪框的坐标值并做临界越界处理。

### 裁剪

当裁剪过后怎么拿到裁剪的图片呢？看看getCroppedImage方法：

```
public Bitmap getCroppedImage() {


        final Drawable drawable = getDrawable();
        if (drawable == null || !(drawable instanceof BitmapDrawable)) {
            return null;
        }

        final float[] matrixValues = new float[9];
        getImageMatrix().getValues(matrixValues);

        final float scaleX = matrixValues[Matrix.MSCALE_X];
        final float scaleY = matrixValues[Matrix.MSCALE_Y];
        final float transX = matrixValues[Matrix.MTRANS_X];
        final float transY = matrixValues[Matrix.MTRANS_Y];

        float bitmapLeft = (transX < 0) ? Math.abs(transX) : 0;
        float bitmapTop = (transY < 0) ? Math.abs(transY) : 0;

        //获取原图片
        final Bitmap originalBitmap = ((BitmapDrawable) drawable).getBitmap();

         //获取裁剪框x,y的坐标位置
        final float cropX = (bitmapLeft + Edge.LEFT.getCoordinate()) / scaleX;
        final float cropY = (bitmapTop + Edge.TOP.getCoordinate()) / scaleY;

        //计算裁剪框的宽高
        final float cropWidth = Math.min(Edge.getWidth() / scaleX, originalBitmap.getWidth() - cropX);
        final float cropHeight = Math.min(Edge.getHeight() / scaleY, originalBitmap.getHeight() - cropY);

        //生成裁剪框的bitmap
        return Bitmap.createBitmap(originalBitmap,
                (int) cropX,
                (int) cropY,
                (int) cropWidth,
                (int) cropHeight);

    }
    
```

通过上面的方法我们拿到了裁剪的bitmap，也就是createBitmap方法的合理使用而已。然后就可以进行自己的操作了，比如上传图片，或者保存bitmap到本地等等。

### 使用方法

当然此自定义裁剪View使用起来也很简单：

```
       cropImageView = (CropImageView) findViewById(R.id.cropImageView);
        //设置要裁剪图片
        cropImageView.setImageResource(R.drawable.timg);

        findViewById(R.id.cropOk).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //获取裁剪的图片
                Bitmap cropBitMap = cropImageView.getCroppedImage();
                cropImageView.setImageBitmap(cropBitMap);
            }
        });

```
## 总结

学习从代码设计角度来说[cropper](https://github.com/edmodo/cropper)库是很好的教材可以让我们去体会。