# SuperChartView

#### 1、添加依赖和配置

* 根目录build.gradle文件添加如下配置：

```Java
allprojects {
    repositories {
       	maven { url 'https://jitpack.io' }
    }
}
```

* APP目录build.gradle文件添加如下配置：

```Java
dependencies {
    'com.github.Victory-Over:SuperChartView:v1.0.0'
}
```

#### 2、效果展示
![点我查看效果图](https://github.com/Victory-Over/Resource/blob/master/file_chartview.gif)

#### 3、核心代码

###### 1.ScrollChartView 可滚动的自定义图表
每次滚动完成 计算滚动的位置，使indicate居中并回调当前位置的position 供外部使用
```Java
    /**
     * 调整indicate，使其居中。
     */
    private void adjustIndicate() {
        if (!mOverScroller.isFinished()) {
            mOverScroller.abortAnimation();
        }

        int position = computeSelectedPosition();
        int scrollX = getScrollByPosition(position);
        scrollX -= getScrollX();
        this.position = position;

        if (scrollX != 0) {
            mOverScroller.startScroll(getScrollX(), getScrollY(), scrollX, 0);
            invalidateView();
        }

        //滚动完毕回调
        onScaleChanged(position);
    }
```

根据传入的position 计算出每个indicate的位置，用于画图 
例如position = 5 * indicate总宽度(indicate宽度+indicatePadding间隔*2) = 80 
则该下标的位置为 left = 400 , right = left+indicate。
```Java
    /**
     * 计算indicate的位置
     */
    private void computeIndicateLoc(Rect outRect, int position) {
        if (outRect == null) {
            return;
        }

        int height = getHeight();
        int indicate = getIndicateWidth();

        int left = (indicate * position);
        int right = left + indicate;
        int top = getPaddingTop();
        int bottom = height - getPaddingBottom();

        if (isAlignTop()) {
            bottom -= mIndicateBottomPadding;
        } else {
            top += mIndicateBottomPadding;
        }

        outRect.set(left, top, right, bottom);
    }
  ```
  
上面两个方法是调整indicate 并计算出他的位置，得到这些参数后，就可以开始画图了
  
```Java
    /**
     * 绘制网格线
     */
    private void drawGridLine(Canvas canvas) {
        for (int i = 0; i < mList.size(); i++) {
            computeIndicateLoc(mIndicateLoc, i);
            int left = mIndicateLoc.left + mIndicatePadding;
            int right = mIndicateLoc.right - mIndicatePadding;
            int bottom = getHeight() - mShadowMarginHeight;
            canvas.drawRect(left, 0, right, bottom, mGridPaint);
        }
    }
```

this.position == position 判断当前的position与将要绘制的position是否一致，是则改变其颜色
并判断SDK版本是否大于21(支持画圆角的矩形)
```Java
    /**
     * 绘制指示标
     */
    private void drawIndicate(Canvas canvas, int position) {
        computeIndicateLoc(mIndicateLoc, position);
        int left = mIndicateLoc.left + mIndicatePadding;
        int right = mIndicateLoc.right - mIndicatePadding;
        int bottom = mIndicateLoc.bottom;
        int top = bottom - mIndicateHeight;
        if (this.position == position) {
            mIndicatePaint.setColor(mSelectedColor);
        } else {
            mIndicatePaint.setColor(mIndicateColor);
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            canvas.drawRoundRect(left, top, right, bottom, 5, 5, mIndicatePaint);
        } else {
            canvas.drawRect(left, top, right, bottom, mIndicatePaint);
        }
    }
```
 
同上，如果position一致则改变其大小和颜色
```Java
    /**
     * 绘制文字
     */
    private void drawText(Canvas canvas, int position, String text) {
        computeIndicateLoc(mIndicateLoc, position);

        if (this.position == position) {
            mTextPaint.setTextSize(mTextSelectedSize);
            mTextPaint.setColor(mSelectedColor);
        } else {
            mTextPaint.setTextSize(mTextSize);
            mTextPaint.setColor(mTextColor);
        }

        int x = (mIndicateLoc.left + mIndicateLoc.right) / 2;
        int y = mIndicateLoc.bottom + mIndicateBottomPadding - mTextBottomPadding;

        if (!isAlignTop()) {
            y = mIndicateLoc.top;
            mTextPaint.getTextBounds(text, 0, text.length(), mIndicateLoc);
            //增加一些偏移
            y += mIndicateLoc.top / 2;
        }

        canvas.drawText(text, x, y, mTextPaint);
    }
 ```
 
 ```Java
   /**
     * 绘制折线图
     */
    private void drawLine(Canvas canvas) {
        Path path = new Path();
        path.moveTo(mList.get(0).x, mList.get(0).y);
        for (int i = 1; i < mList.size(); i++) {
            path.lineTo(mList.get(i).x, mList.get(i).y);
        }
        canvas.drawPath(path, mLinePaint);
    }


    /**
     * 绘制曲线图
     */
    private void drawScrollLine(Canvas canvas) {
        Point pStart;
        Point pEnd;
        Path path = new Path();
        for (int i = 0; i < mList.size() - 1; i++) {
            pStart = mList.get(i);
            pEnd = mList.get(i + 1);
            Point point3 = new Point();
            Point point4 = new Point();
            float wd = (pStart.x + pEnd.x) / 2;
            point3.x = wd;
            point3.y = pStart.y;
            point4.x = wd;
            point4.y = pEnd.y;
            path.moveTo(pStart.x, pStart.y);
            path.cubicTo(point3.x, point3.y, point4.x, point4.y, pEnd.x, pEnd.y);
            canvas.drawPath(path, mLinePaint);
        }
    }

    /**
     * 绘制阴影
     */
    private void drawShadow(Canvas canvas) {
        if (mLineType == LineType.ARC) {
            Point pStart;
            Point pEnd;
            Path path = new Path();
            for (int i = 0; i < mList.size() - 1; i++) {
                pStart = mList.get(i);
                pEnd = mList.get(i + 1);
                Point point3 = new Point();
                Point point4 = new Point();
                float wd = (pStart.x + pEnd.x) / 2;
                point3.x = wd;
                point3.y = pStart.y;
                point4.x = wd;
                point4.y = pEnd.y;
                path.moveTo(pStart.x, pStart.y);
                path.cubicTo(point3.x, point3.y, point4.x, point4.y, pEnd.x, pEnd.y);
                //减去文字和指示标的高度
                path.lineTo(pEnd.x, getHeight() - mShadowMarginHeight);
                path.lineTo(pStart.x, getHeight() - mShadowMarginHeight);
            }
            path.close();
            canvas.drawPath(path, mShadowPaint);
        } else {
            Path path = new Path();
            path.moveTo(mList.get(0).x, mList.get(0).y);
            for (int i = 1; i < mList.size(); i++) {
                path.lineTo(mList.get(i).x, mList.get(i).y);
            }
            //链接最后两个点
            int index = mList.size() - 1;
            path.lineTo(mList.get(index).x, getHeight() - mShadowMarginHeight);
            path.lineTo(mList.get(0).x, getHeight() - mShadowMarginHeight);
            path.close();
            canvas.drawPath(path, mShadowPaint);
        }
    }
```
