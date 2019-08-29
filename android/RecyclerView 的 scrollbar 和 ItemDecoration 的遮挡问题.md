# `RecyclerView` 的 `scrollbar` 和 `ItemDecoration` 的遮挡问题

## 前言

`RecyclerView` 是自带 `scrollbar` 的, 可自定义设置它的展示与方向还有属性「`scrollbarStyle`」。

`RecyclerView` 的 `ItemDecoration` 很方便，可以为每个 `item` 之间添加分割线， 那么分割线的绘制是怎么绘制的呢？与 `item view` 的绘制顺序是什么样的呢？

以下内容分为三部分：

1. `scrollbar` 的属性 `scrollbarStyle`
2.  `ItemDecoration` 自定义分割线的注意事项和**绘制顺序**
3.  两者之间可能产生的问题

### 1. `scrollbar` 的属性 `scrollbarStyle`

在 `RecyclerView` 里面 `scrollbar` 的属性 是支持直接在 `xml` 中设置属性的 `scrollbarStyle`

如下代码：

```
<com.android.base.widget.ZRecyclerView
            android:id="@+id/recycler"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:scrollbarStyle="insideOverlay"
            android:scrollbars="vertical"/>
```

> 注：`ZRecyclerView` 简单继承自 `RecyclerView`

#### 1.1 `android:scrollbarStyle` 取值

`android:scrollbarStyle` 有四种：
1. `insideOverlay`  默认值
	表示在 `padding` 区域内并且覆盖在 `view`上  
	**这里的 `view` 都是指 `RecyclerView` **
	
2. `insideInset`
	表示在 `padding` 区域内并且插入在 `view` 后面
	
3. `outsideOverlay`
	表示在 `padding` 区域外并且覆盖在 `view` 上
	
4. `outsideInset`
	表示在 `padding` 区域外并且插入在 `view` 后面

假设设置的 `RecyclerView` 属性为上面代码所示，且不为它设置 `padding`

当 `android:scrollbarStyle="insideInset|outsideInset"` 时，

利用 `Layout inspector`的到的布局显示结果图：

![layout-inspect](https://upload-images.jianshu.io/upload_images/1211741-276c3f9d2e0ec588.png?imageMogr2/auto-orient/)

会发现，`RecyclerView` 会额外造成  `RecyclerView` 多了一个 `paddingRight = 11`

	> 注： 11 为像素值，本质是 `scrollbar` 的宽度，`4 dp`

当 `android:scrollbarStyle="insideOverlay|outsideOverlay"` 时，

利用 `Layout inspector`的到的布局显示结果图：

![](https://upload-images.jianshu.io/upload_images/1211741-ed7c597f3ea99054.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/390/format)


会发现 `RecyclerView` 并没有多余的 `padding`。

#### 1.2 **源码分析**

首先 `android:scrollbarStyle` 对应的 `java` 方法是 `View.setScrollBarStyle()`, 在该方法中，对 `mViewFlags` 进行了赋值。

在 `View` 的源码中，`setPadding(xxx)` 的实现中，最后一行会调用 `internalSetPadding(left, top, right, bottom)`

在 `internalSetPadding(xxx)`方法中，	会根据 `mViewFlags` 对 进行判断，会对 `mPaddingRight` 进行 `+ offset` 添加偏移值「`getVerticalScrollbarWidth()`」 

![代码示例](https://upload-images.jianshu.io/upload_images/1211741-0befd79f82364d6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/703)

#### 结论：除非有必要，且已知的情况下，请不要使用 `android:scrollbarStyle="insideInset|outsideInset"`,  默认的属性为 `insideOverlay` 可以满足我们的需要。

**当修改 `android:scrollbarStyle` 时，会对 `RecyclerView` 里面的子 `item` 的宽有影响「宽度减少」，布局上产生影响。**


### 2. `ItemDecoration` 自定义分割线的注意事项和绘制顺序

自定义分割线时，需要继承 `RecyclerView.ItemDecoration`  并且实现三个方法: 

1. `onDraw(xxx)`
	利用 `canvas` 可以画出你想要的分割线样式
	
	```
	canvas.drawRect(left.toFloat(), top.toFloat(), right.toFloat(), (top + mDividerHeight).toFloat(), mPaint)
	```
2. `onDrawOver(xxx)`
	利用 `canvas` 可以画出你想要的分割线样式
3. `getItemOffsets(xxx)`
	
	这里是设置 `item view`绘制区域的偏移值
	
在 `onDraw(xxx)` 和 `onDrawOver(xxx)` 里面都可以让我们去画出分割线，那么这两个方法的区别是什么呢？从名字上来看，`onDrawOver(xxx)` 绘制的时机应该比 `onDraw(xxx)` 要晚。

#### 2.1 绘制顺序

那么具体的实现呢？源码：
在 `RecyclerView` 的  `draw(xxx)` 方法里的代码片段：

![代码片段](https://upload-images.jianshu.io/upload_images/1211741-14227f7f7f0b875f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/977/format/)


在 `draw()` 里面首先调用了 `super.draw(xxx)` 「完成绘制 `RecyclerView` 和它里面的子 `view`」

具体逻辑如下，不再详细的分析源码：

```sequence
RecyclerView.draw() -> super.draw(xxx) : 先调用 
super.draw(xxx) -> RecyclerView.onDraw() : 会调用「第一步」
RecyclerView.onDraw() -> ItemDecoration.onDraw(): 在 `RecyclerView`.onDraw() 里会调用「第二步」 
super.draw(xxx) -> 子 view 的 onDraw(): 通过调用 dispatchDraw() 「第三步」
RecyclerView.draw() -> ItemDecoration.onDrawOver(): 再调用 onDrawOver「第四步」
```

#### 2.2 **总结一下绘制顺序为：**

1. 先绘制 `RecyclerView` 自身；
2. 再调用 `ItemDecoration.onDraw()`；
3. 再调用了 `RecyclerView` 里面的子 `view`；
4. 调用了 `ItemDecoration.onDrawOver()`.

所以，如果我们自定义 `ItemDecoration` 是在 `onDraw()` 里面画的分割线，那么会早与 `item view` 的绘制；

所以，如果我们自定义 `ItemDecoration` 是在 `onDrawOver()` 里面画的分割线，那么会晚与 `item view` 的绘制；

#### 2.3 覆盖问题

既然绘制有先后，那么就会存在被覆盖的问题。

当对 `getItemOffsets(xxx)`  方法不做任何操作时，
1. 当在 `ItemDecoration.onDraw()` 方法里画分割线时，画出来的效果，会被 `item view` 覆盖, 即有可能看不出分割线「与没添加分割线一样」

2. 当在 `ItemDecoration.onDrawOver()` 方法里画分割线时，画出来的效果，会遮挡 `item view` 部分区域
	假设，是在卡片下方画分割线，那么画出来的效果是：分割线遮挡住 `item view` 的底部位置。

上述两个问题，并不是我们实际想要的效果，我们想要的分割线效果是不影响 `item view` 的展示。

**所以， 特别重要的是，我们需要重写 `getItemOffsets(xxx)` 这个方法，添加我们想要的分割线的 `offset`**

#### 2.4 `getItemOffsets(xxx)` 的重写

官方源码，示例如下：

```
    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent,
            RecyclerView.State state) {
        if (mDivider == null) {
            outRect.set(0, 0, 0, 0);
            return;
        }
        if (mOrientation == VERTICAL) {
            outRect.set(0, 0, 0, mDivider.getIntrinsicHeight());
        } else {
            outRect.set(0, 0, mDivider.getIntrinsicWidth(), 0);
        }
    }
```

当需要在竖直方向上依次画分割线时，添加的偏移值是 `mDivider.getIntrinsicHeight()` 就是我们想要的分割线的高度。

因为我们需要在 `getItemOffsets(xxx)` 方法中，添加我们想要的分割线的宽度给 `outRect` 的 `offset`.

### 3. 两者之间可能产生的问题

`RecyclerView.scrollbar` 和 `RecyclerView.ItemDecoration` 之间会产生什么问题呢？

1. 在列表滑动的过程中，分割线会覆盖在  `scrollbar` 的上面

	如果分割线的样式「颜色」和 `scrollbar` 的差别很大，那么会产生的视觉效果是：当滑动到两个卡片的交界处「分割线的地方」，「分割线」分割开了 `scrollbar`， 十分的丑。

2. `RecyclerView.ItemDecoration` 分割线并未完全画满屏幕的宽度「即使是 `match_parent`」


#### 3.1  在列表滑动的过程中，分割线会覆盖在  `scrollbar` 的上面

如图：

![分割线错误效果](https://upload-images.jianshu.io/upload_images/1211741-42dcfa67b1b1dd9a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format)

可猜测问题出在：`ItemDecoration` 绘制的时机晚与 `scrollbar` 绘制的时机，导致分割线覆盖在了 `scrollbar` 上面。

那么 `scrollbar` 的绘制时机是在哪里呢？源码中，`View` 的 `onDraw()` 里部分代码如下：

```
if (!verticalEdges && !horizontalEdges) {
    // Step 3, draw the content
    if (!dirtyOpaque) onDraw(canvas);
    // Step 4, draw the children
    dispatchDraw(canvas);
    drawAutofilledHighlight(canvas);
    // Overlay is part of the content and draws beneath Foreground
    if (mOverlay != null && !mOverlay.isEmpty()) {
        mOverlay.getOverlayView().dispatchDraw(canvas);
    }
    // Step 6, draw decorations (foreground, scrollbars)
    onDrawForeground(canvas);
    // Step 7, draw the default focus highlight
    drawDefaultFocusHighlight(canvas);
	...
}

```

在 `step 6` 中，调用了 `onDrawForeground(xxx)`, 而在这个方法中，调用了 

```
// 绘制 `scrollbar` 的位置
onDrawScrollIndicators(canvas);
onDrawScrollBars(canvas);
```

这是绘制 `scrollbar` 的位置。

**那么，我们就知道了：`scrollbar` 的绘制晚与 `item view` 的 `onDraw`, 早与 `ItemDecoration` 的 `onDrawOver()`. **

根据上面我们的分析，

出现该问题的原因是：自定义的 `ItemDecoration` 分割线绘制是在 `onDrawOver()`这个里面绘制的。

**正确的解决办法：** 把绘制分割线时机放在 `onDraw()`这个时机，就可以解决该问题。

**错误的解决办法：** 设置 `RecyclerView` 的  `android:scrollbarStyle="insideInset|outsideInset"`。这样会导致 `3.2` 的问题 , 

#### 3.2 `RecyclerView.ItemDecoration` 分割线并未完全画满屏幕的宽度「即使是 `match_parent`」

从上面，我们也知道了，当设置 `RecyclerView` 的  `android:scrollbarStyle="insideInset|outsideInset"`时，就会额外为 `RecyclerView` 添加一个 `paddingRight`， 导致分割线未绘制全屏。

**解决办法：** 不要使用 `android:scrollbarStyle="insideInset|outsideInset"`


### 总结

以上内容，其实都是对 `RecyclerView` 里面的一些属性的研究，有些内容很细节，
往往不是那么引人注意，但真的可能会造成很困扰的问题，`Android` 里面的一些源码设计里面，还是蛮有逻辑在的。

上述的问题，本质上还是 `view` 的绘制引起的，所以界面遇到遮档问题时，不妨想一想绘制顺序。


> 水平有限，文中有些内容可能存在错误，如有，大胆指出，哪个程序员还没翻过车 ～_～

### 参考链接

1. [有关 `ItemDecoration` 的绘制顺序](https://www.2cto.com/kf/201608/543301.html)
	https://www.2cto.com/kf/201608/543301.html
2. `RecyclerView.java` 源码
3. [RecyclerView之ItemDecoration](https://juejin.im/post/59099fe844d904006942a983) 
	https://juejin.im/post/59099fe844d904006942a983











