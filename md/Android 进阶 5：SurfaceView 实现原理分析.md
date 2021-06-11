> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903968217235469)

> 第一次接触 SurfaceView，找了很多资料才理解 SurfaceView 概念，总结查资料的结果。

### SurfaceView 的概念

第一次接触 SurfaceView，找了很多资料才理解 SurfaceView 概念，总结查资料的结果。Android 中有一种特殊的视图，称为 SurefaceView，与平时时候的 TextView、Button 的区别：

1.  它拥有独立的特殊的绘制表面，即 它不与其宿主窗口共享一个绘制表面
2.  SurefaceView 的 UI 可以在一个独立的线程中进行绘制
3.  因为不会占用主线程资源，一方面可以实现复杂而高效的 UI，二是不会导致用户输入得不到及时响应。

**综合这些特点，SurfaceView 一般用来实现动态的或者比较复杂的图像还有动画的显示。**

### 为什么需要 SurfaceView

普通空间，TextView,Button 等，都是讲自己的 UI 绘制在宿主窗口的绘制表面 Surface 之上，意味着他们的 UI 在应用程序的主线程中绘制的。但是主线程除了绘制 UI 之外，还要及时响应用户输入，手势等，否则，系统会认为应用程序没响应，ANR。

因此，对一些游戏画面，或者摄像头，视频播放等，UI 都比较复杂，要求能够进行高效的绘制，因此，他们的 UI 不适合在主线程中绘制。这时候就必须要给那些需要复杂而高效的 UI 视图生成一个独立的绘制表面 Surface, 并且使用独立的线程来绘制这些视图 UI。

### 理解 SurfaceView 的形成

每个窗口在 SurfaceFlinger 服务中都对应有一个 layer，用来描述它的绘制表面 surface。**对于那些具有 SurfaceView 的窗口来说，每个 SurfaceFlinger 服务中还对应一个独立的 Layer 或者 LayerBuffer**，用来单独描述它的绘制表面，以区别它的宿主窗口的绘制表面。

无论是 LayerBuffer，还是 Layer，它们都是以 LayerBase 为基类的，也就是说，SurfaceFlinger 服务把所有的 LayerBuffer 和 Layer 都抽象为 LayerBase，因此就可以用统一的流程来绘制和合成它们的 UI。由于 LayerBuffer 的绘制和合成与 Layer 的绘制和合成是类似的

为了接下来可以方便地描述 SurfaceView 的实现原理分析，我们假设在一个 Activity 窗口的视图结构中，除了有一个 DecorView 顶层视图之外，还有两个 TextView 控件，以及一个 SurfaceView 视图，这样该 Activity 窗口在 SurfaceFlinger 服务中就对应有两个 Layer 或者一个 Layer 的一个 LayerBuffer，如图 1 所示  

![](https://user-gold-cdn.xitu.io/2019/10/16/16dd4db144b4ab24?imageView2/0/w/1280/h/960/ignore-error/1)图 1 SurfaceView 及其宿主 Activity 窗口的绘图表面示意图

在图 1 中，Activity 窗口的顶层视图 DecorView 及其两个 TextView 控件的 UI 都是绘制在 SurfaceFlinger 服务中的同一个 Layer 上面的，而 SurfaceView 的 UI 是绘制在 SurfaceFlinger 服务中的另外一个 Layer 或者 LayerBuffer 上的。

注意，用来描述 SurfaceView 的 Layer 或者 LayerBuffer 的 Z 轴位置是小于用来其宿主 Activity 窗口的 Layer 的 Z 轴位置的，但是前者会在后者的上面挖一个 “洞” 出来，以便它的 UI 可以对用户可见。**实际上，SurfaceView 在其宿主 Activity 窗口上所挖的 “洞” 只不过是在其宿主 Activity 窗口上设置了一块透明区域。**

从总体上描述了 SurfaceView 的大致实现原理之后，接下来我们就详细分析它的具体实现过程，包括它的绘图表面的创建过程、在宿主窗口上面进行挖洞的过程，以及绘制过程。参考链接: https://blog.csdn.net/luoshengyang/article/details/8661317/

#### SurfaceView 的绘制过程

SurfaceView 虽然具有独立的绘图表面，不过它仍然是宿主窗口的视图结构中的一个结点，因此，它仍然是可以参与到宿主窗口的绘制流程中去的。从前面 [Android 应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](https://blog.csdn.net/luoshengyang/article/details/8661317/)一文可以知道，窗口在绘制的过程中，每一个子视图的成员函数 draw 或者 dispatchDraw 都会被调用到，以便它们可以绘制自己的 UI。  
SurfaceView 类的成员函数 draw 和 dispatchDraw 的实现如下所示：

```
    @Override
    public void draw(Canvas canvas) {
        if (mDrawFinished && !isAboveParent()) {
            // draw() is not called when SKIP_DRAW is set
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == 0) {
                // punch a whole in the view-hierarchy below us
                canvas.drawColor(0, PorterDuff.Mode.CLEAR);
            }
        }
        super.draw(canvas);
    }

    @Override
    protected void dispatchDraw(Canvas canvas) {
        if (mDrawFinished && !isAboveParent()) {
            // draw() is not called when SKIP_DRAW is set
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                // punch a whole in the view-hierarchy below us
                canvas.drawColor(0, PorterDuff.Mode.CLEAR);
            }
        }
        super.dispatchDraw(canvas);
    }
复制代码

```

SurfaceView 类的成员函数 draw 和 dispatchDraw 的参数 canvas 所描述的都是建立在宿主窗口的绘图表面上的画布，因此，在这块画布上绘制的任何 UI 都是出现在宿主窗口的绘图表面上的.

本来 SurfaceView 类的成员函数 draw 是用来将自己的 UI 绘制在宿主窗口的绘图表面上的，但是这里我们可以看到，如果当前正在处理的 SurfaceView 不是用作宿主窗口面板的时候，即其成员变量 mWindowType 的值不等于 WindowManager.LayoutParams.TYPE_APPLICATION_PANEL 的时候，**SurfaceView 类的成员函数 draw 只是简单地将它所占据的区域绘制为黑色。**

本来 SurfaceView 类的成员函数 dispatchDraw 是用来绘制 SurfaceView 的子视图的，但是这里我们同样看到，如果当前正在处理的 SurfaceView 不是用作宿主窗口面板的时候 ，那么 SurfaceView 类的成员函数 dispatchDraw 只是简单地将它所占据的区域绘制为黑色，同时，它还会通过调用另外一个成员函数 updateWindow 更新自己的 UI，实际上就是请求 WindowManagerService 服务对自己的 UI 进行布局，以及创建绘图表面

从 SurfaceView 类的成员函数 draw 和 dispatchDraw 的实现就可以看出，SurfaceView 在其宿主窗口的绘图表面上面所做的操作就是将自己所占据的区域绘为黑色，除此之外，就没有其它更多的操作了，这是因为 SurfaceView 的 UI 是要展现在它自己的绘图表面上面的。接下来我们就分析如何在 SurfaceView 的绘图表面上面进行 UI 绘制。

从前面 [Android 应用程序窗口（Activity）的测量（Measure）、布局（Layout）和绘制（Draw）过程分析](https://blog.csdn.net/luoshengyang/article/details/8661317/)一文可以知道，如果要在一个绘图表面进行 UI 绘制，那么就顺序执行以下的操作：

```
    (1). 在绘图表面的基础上建立一块画布，即获得一个Canvas对象。
    (2). 利用Canvas类提供的绘图接口在前面获得的画布上绘制任意的UI。
    (3). 将已经填充好了UI数据的画布缓冲区提交给SurfaceFlinger服务，以便SurfaceFlinger服务可以将它合成到屏幕上去。
复制代码

```

### 分析源码

分析 Surface，SurfaceHolder，SurfaceView 三个类

**Surface**:

处理被屏幕排序的原生的 buffer，Android 中的 Surface 就是一个用来画图形（graphics）或图像（image）的地方，对于 View 及其子类，都是画在 Surface 上，各 Surface 对象通过 Surfaceflinger 合成到 frameBuffer，每个 Surface 都是双缓冲（实际上就是两个线程，一个渲染线程，一个 UI 更新线程），它有一个 backBuffer 和一个 frontBuffer，Surface 中创建了 Canvas 对象，用来管理 Surface 绘图操作，Canvas 对应 Bitmap，存储 Surface 中的内容。

**SurfaceView**:

SurfaceView 是 View 的子类，且实现了 Parcelable 接口且实现了 Parcelable 接口，其中**内嵌了一个专门用于绘制的 Surface，SurfaceView 可以控制这个 Surface 的格式和尺寸，以及 Surface 的绘制位置**。可以理解为 Surface 就是管理数据的地方，SurfaceView 就是展示数据的地方。

**SurfaceHolder**:

一个管理 SurfaceHolder 的容器。SurfaceHolder 是一个接口，可理解为一个 Surface 的监听器。 通过回调方法 addCallback（SurfaceHolder.Callback callback ) 监听 Surface 的创建 通过获取 Surface 中的 Canvas 对象，并锁定之。所得到的 Canvas 对象 通过当修改 Surface 中的数据完成后，释放同步锁，并提交改变 Surface 的状态及图像，将新的图像数据进行展示。-

而最后综合：SurfaceView 中调用 getHolder 方法，可以获得当前 SurfaceView 中的 Surface 对应的 SurfaceHolder，SurfaceHolder 开始对 Surface 进行管理操作。这里其实按 MVC 模式理解的话，可以更好理解。M:Surface(图像数据)，V:SurfaceView(图像展示)，C:SurfaceHolder(图像数据管理)。

### Surface 源码

```
/**
 * Handle onto a raw buffer that is being managed by the screen compositor.
 */
public class Surface implements Parcelable {  
    // code......
}
复制代码

```

首先来看 Surface 这个类，它实现了 Parcelable 接口进行序列化（这里主要用来在进程间传递 surface 对象），用来处理屏幕显示缓冲区的数据，源码中对它的注释为： Handle onto a raw buffer that is being managed by the screen compositor. **Surface 是原始图像缓冲区（raw buffer）的一个句柄**，而原始图像缓冲区是由屏幕图像合成器（screen compositor）管理的。

*   由屏幕显示内容合成器 (screen compositor) 所管理的原生缓冲器的句柄（类似句柄） - 名词解释：句柄，英文：HANDLE，数据对象进入内存之后获取到内存地址，但是所在的内存地址并不是固定的，需要用句柄来存储内容所在的内存地址。从数据类型上来看它只是一个 32 位 (或 64 位) 的无符号整数。 - Surface 充当句柄的角色，用来获取源生缓冲区以及其中的内容 - 源生缓冲区（raw buffer）用来保存当前窗口的像素数据 - 于是可知 Surface 就是 Android 中用来绘图的的地方，具体来说应该是 Surface 中的 Canvas Surface 中定义了画布相关的 Canvas 对象

```
private final Canvas mCanvas = new CompatibleCanvas();  
复制代码

```

Java 中，绘图通常在一个 Canvas 对象上进行的，Surface 中也包含了一个 Canvas 对象，这里的 CompatibleCanvas 是 Surface.java 中的一个内部类，其中包含一个矩阵对象 Matrix（变量名 mOrigMatrix）。矩阵 Matrix 就是一块内存区域，针对 View 的各种绘画操作都保存在此内存中。  
Surface 内部有一个 CompatibleCanvas 的内部类，这个内部类的作用是为了能够兼容 Android 各个分辨率的屏幕，根据不同屏幕的分辨率处理不同的图像数据。

```
private final class CompatibleCanvas extends Canvas {  
        // A temp matrix to remember what an application obtained via {@link getMatrix}
        private Matrix mOrigMatrix = null;

        @Override
        public void setMatrix(Matrix matrix) {
            if (mCompatibleMatrix == null || mOrigMatrix == null || mOrigMatrix.equals(matrix)) {
                // don't scale the matrix if it's not compatibility mode, or
                // the matrix was obtained from getMatrix.
                super.setMatrix(matrix);
            } else {
                Matrix m = new Matrix(mCompatibleMatrix);
                m.preConcat(matrix);
                super.setMatrix(m);
            }
        }

        @SuppressWarnings("deprecation")
        @Override
        public void getMatrix(Matrix m) {
            super.getMatrix(m);
            if (mOrigMatrix == null) {
                mOrigMatrix = new Matrix();
            }
            mOrigMatrix.set(m);
        }
    }
复制代码

```

1.Surface 的两个重要方法：  
Surface 中的很多方法都是原生方法，lockCanvas 和 unlockCanvasAndPost 也是原生的，这里不是指 SurfaceHolder 中的 lockCanvas 和 unlockCanvasAndPost，SurfaceHolder 只是做了封装。

*   lockCanvas(…) + Gets a Canvas for drawing into this surface. 获取进行绘画的 Canvas 对象 + After drawing into the provided Canvas, the caller must invoke unlockCanvasAndPost to post the new contents to the surface. 绘制完一帧的数据之后需要调用 unlockCanvasAndPost 方法把画布解锁，然后把画好的图像 Post 到当前屏幕上去显示 + 当一个 Canvas 在被绘制的时候，它是出于被锁定的状态，就是说必须等待正在绘制的这一帧绘制完成之后并解锁画布之后才能进行别的操作 + 实际锁住 Canvas 的过程是在 jni 层完成的
*   unlockCanvasAndPost(…) + Posts the new contents of the Canvas to the surface and releases the Canvas. 将新绘制的图像内容传给 surface 之后这个 Canvas 对象会被释放掉（实际释放的过程是在 jni 层完成的）

urface 的 lockCanvas 和 unlockCanvasAndPost 两个方法最终都是调用 jni 层的方法来处理，有兴趣可以看下相关的源码：

```
/frameworks/native/libs/gui/Surface.cpp /frameworks/base/core/jni/android_view_Surface.cpp
复制代码

```

### SurfaceHolder 源码

SurfaceHolder 实际上是一个接口，它充当的是 Controller 的角色。

```
package android.view;

import android.graphics.Canvas;
import android.graphics.Rect;

/**
 * Abstract interface to someone holding a display surface.  Allows you to
 * control the surface size and format, edit the pixels in the surface, and
 * monitor changes to the surface.  This interface is typically available
 * through the {@link SurfaceView} class.
 *
 * <p>When using this interface from a thread other than the one running
 * its {@link SurfaceView}, you will want to carefully read the
 * methods
 * {@link #lockCanvas} and {@link Callback#surfaceCreated Callback.surfaceCreated()}.
 */
public interface SurfaceHolder {

    ...otherCodes

    public interface Callback {

        public void surfaceCreated(SurfaceHolder holder);

        public void surfaceChanged(SurfaceHolder holder, int format, int width,
                int height);

        public void surfaceDestroyed(SurfaceHolder holder);
    }

    /**
     * Additional callbacks that can be received for {@link Callback}.
     */
    public interface Callback2 extends Callback {

        void surfaceRedrawNeeded(SurfaceHolder holder);

        default void surfaceRedrawNeededAsync(SurfaceHolder holder, Runnable drawingFinished) {
            surfaceRedrawNeeded(holder);
            drawingFinished.run();
        }
    }

    public void addCallback(Callback callback);

    public void removeCallback(Callback callback);

    public boolean isCreating();

    @Deprecated
    public void setType(int type);

    public void setFixedSize(int width, int height);

    public void setSizeFromLayout();

    public void setFormat(int format);

    public void setKeepScreenOn(boolean screenOn);

    public Canvas lockCanvas();

    public Canvas lockCanvas(Rect dirty);

    default Canvas lockHardwareCanvas() {
        throw new IllegalStateException("This SurfaceHolder doesn't support lockHardwareCanvas");
    }

    public void unlockCanvasAndPost(Canvas canvas);

    public Rect getSurfaceFrame();

    public Surface getSurface();
}
复制代码

```

1.  关键接口 Callback

callback 是 SurfaceHolder 内部的一个接口，例子中就实现了这个接口来控制绘制动画的线程。  
接口中有以下三个方法

*   public void surfaceCreated(SurfaceHolder holder); + Surface 第一次被创建时被调用，例如 SurfaceView 从不可见状态到可见状态时
    *   在这个方法被调用到 surfaceDestroyed 方法被调用之前的这段时间，Surface 对象是可以被操作的，拿 SurfaceView 来说就是如果 SurfaceView 只要是在界面上可见的情况下，就可以对它进行绘图和绘制动画
    *   这里还有一点需要注意，Surface 在一个线程中处理需要渲染的图像数据，如果你已经在另一个线程里面处理了数据渲染，就不需要在这里开启线程对 Surface 进行绘制了
*   public void surfaceChanged(SurfaceHolder holder, int format, int width, int height);
    *   Surface 大小和格式改变时会被调用，例如横竖屏切换时如果需要对 Sufface 的图像和动画进行处理，就需要在这里实现
    *   这个方法在 surfaceCreated 之后至少会被调用一次
*   public void surfaceDestroyed(SurfaceHolder holder);
    *   Surface 被销毁时被调用，例如 SurfaceView 从可见到不可见状态时
    *   在这个方法被调用过之后，就不能够再对 Surface 对象进行任何操作，所以需要保证绘图的线程在这个方法调用之后不再对 Surface 进行操作，否则会报错

### SurfaceView 源码

SurfaceView，就是用来显示 Surface 数据的 View，通过 SurfaceView 来看到 Surface 的数据。

```
public class SurfaceView extends View implements ViewRootImpl.WindowStoppedCallback {

    //OtherCodes

   final ArrayList<SurfaceHolder.Callback> mCallbacks
            = new ArrayList<SurfaceHolder.Callback>();//重要的回调集合

    final int[] mLocation = new int[2];

    final ReentrantLock mSurfaceLock = new ReentrantLock();
    // Current surface in use
    final Surface mSurface = new Surface();       

    //OtherCodes

     /**
     * Return the SurfaceHolder providing access and control over this
     * SurfaceView's underlying surface.
     *
     * @return SurfaceHolder The holder of the surface.
     */
     //获取当前持有的holder
    public SurfaceHolder getHolder() {
        return mSurfaceHolder;
    }

    //surfaceView持有mSurfaceHolder的final类。对上面的surface进行管理

    private final SurfaceHolder mSurfaceHolder = new SurfaceHolder() {
        private static final String LOG_TAG = "SurfaceHolder";

        @Override
        public boolean isCreating() {
            return mIsCreating;
        }

         //把holder.callback添加到上面回调集合
        @Override
        public void addCallback(Callback callback) {
            synchronized (mCallbacks) {
                // This is a linear search, but in practice we'll
                // have only a couple callbacks, so it doesn't matter.
                if (mCallbacks.contains(callback) == false) {
                    mCallbacks.add(callback);
                }
            }
        }

        @Override
        public void removeCallback(Callback callback) {
            synchronized (mCallbacks) {
                mCallbacks.remove(callback);
            }
        }

        @Override
        public void setFixedSize(int width, int height) {
            if (mRequestedWidth != width || mRequestedHeight != height) {
                mRequestedWidth = width;
                mRequestedHeight = height;
                requestLayout();
            }
        }

        @Override
        public void setSizeFromLayout() {
            if (mRequestedWidth != -1 || mRequestedHeight != -1) {
                mRequestedWidth = mRequestedHeight = -1;
                requestLayout();
            }
        }

        @Override
        public void setFormat(int format) {
            // for backward compatibility reason, OPAQUE always
            // means 565 for SurfaceView
            if (format == PixelFormat.OPAQUE)
                format = PixelFormat.RGB_565;

            mRequestedFormat = format;
            if (mSurfaceControl != null) {
                updateSurface();
            }
        }

        /**
         * @deprecated setType is now ignored.
         */
        @Override
        @Deprecated
        public void setType(int type) { }

        @Override
        public void setKeepScreenOn(boolean screenOn) {
            runOnUiThread(() -> SurfaceView.this.setKeepScreenOn(screenOn));
        }

        /**
         * Gets a {@link Canvas} for drawing into the SurfaceView's Surface
         *
         * After drawing into the provided {@link Canvas}, the caller must
         * invoke {@link #unlockCanvasAndPost} to post the new contents to the surface.
         *
         * The caller must redraw the entire surface.
         * @return A canvas for drawing into the surface.
         */

         //封装的锁Canvas
        @Override
        public Canvas lockCanvas() {
            return internalLockCanvas(null, false);
        }

        /**
         * Gets a {@link Canvas} for drawing into the SurfaceView's Surface
         *
         * After drawing into the provided {@link Canvas}, the caller must
         * invoke {@link #unlockCanvasAndPost} to post the new contents to the surface.
         *
         * @param inOutDirty A rectangle that represents the dirty region that the caller wants
         * to redraw.  This function may choose to expand the dirty rectangle if for example
         * the surface has been resized or if the previous contents of the surface were
         * not available.  The caller must redraw the entire dirty region as represented
         * by the contents of the inOutDirty rectangle upon return from this function.
         * The caller may also pass <code>null</code> instead, in the case where the
         * entire surface should be redrawn.
         * @return A canvas for drawing into the surface.
         */
        @Override
        public Canvas lockCanvas(Rect inOutDirty) {
            return internalLockCanvas(inOutDirty, false);
        }

        @Override
        public Canvas lockHardwareCanvas() {
            return internalLockCanvas(null, true);
        }

         //内部锁画布
        private Canvas internalLockCanvas(Rect dirty, boolean hardware) {
            mSurfaceLock.lock();

            if (DEBUG) Log.i(TAG, System.identityHashCode(this) + " " + "Locking canvas... stopped="
                    + mDrawingStopped + ", surfaceControl=" + mSurfaceControl);

            Canvas c = null;
            if (!mDrawingStopped && mSurfaceControl != null) {
                try {
                    //最终是锁住持有的surface的canvas，上面说的surface的lockcanvas是jni方法
                    if (hardware) {

                        c = mSurface.lockHardwareCanvas();
                    } else {
                        c = mSurface.lockCanvas(dirty);
                    }
                } catch (Exception e) {
                    Log.e(LOG_TAG, "Exception locking surface", e);
                }
            }

            if (DEBUG) Log.i(TAG, System.identityHashCode(this) + " " + "Returned canvas: " + c);
            if (c != null) {
                mLastLockTime = SystemClock.uptimeMillis();
                return c;
            }

            // If the Surface is not ready to be drawn, then return null,
            // but throttle calls to this function so it isn't called more
            // than every 100ms.
            long now = SystemClock.uptimeMillis();
            long nextTime = mLastLockTime + 100;
            if (nextTime > now) {
                try {
                    Thread.sleep(nextTime-now);
                } catch (InterruptedException e) {
                }
                now = SystemClock.uptimeMillis();
            }
            mLastLockTime = now;
            mSurfaceLock.unlock();

            return null;
        }

        /**
         * Posts the new contents of the {@link Canvas} to the surface and
         * releases the {@link Canvas}.
         *
         * @param canvas The canvas previously obtained from {@link #lockCanvas}.
         */

         //封装的解锁canvas，实际也是调用Surafce的解锁canvas
        @Override
        public void unlockCanvasAndPost(Canvas canvas) {
            mSurface.unlockCanvasAndPost(canvas);
            mSurfaceLock.unlock();
        }
        //获取当前持有的 surface
        @Override
        public Surface getSurface() {
            return mSurface;
        }

        @Override
        public Rect getSurfaceFrame() {
            return mSurfaceFrame;
        }
    };

}
    ...otherCodes
复制代码

```

### SurfaceView 的使用

surfaceview 提供了两个线程：UI 线程和渲染线程。

1.  所有 SurfaceView 和 SurfaceHolder.Callback 的方法都应该在 UI 线程里调用，一般来说就是应用程序的主线程。渲染线程访问的各种变量应该做同步处理。
2.  由于 surface 可能被销毁，它只在 SurfaceHolder.Callback.surfaceCreated() 和 SurfaceHolder.Callback.surfaceDestroyed() 之间有效，所以要确保渲染线程访问的是合法有效的 surface。
3.    
    SurfaceView 是个重要的绘图容器，它可以在主线程外的线程中向屏幕绘图，这样可以避免画图任务繁重的时候造成主线程阻塞，从而提高了程序的反应速度。在游戏开发中多用到 SurfaceView，游戏中的背景、人物、动画等等尽量在画布 canvas 中画出。  
    可以把 Surface 理解为显存的一个映射，写入到 Surface 的内容可以直接复制到显存从而显示出来，这会使得显示速度非常快），Surface 被销毁之前必须结束。  
    应用过程：

```
1. class MyView extends SurfaceView implements SurfaceHolder.Callback 
2. SurfaceView.getHolder()获得SurfaceHolder对象 
3. SurfaceHolder.addCallback(callback)添加回调函数 
4. SurfaceHolder.lockCanvas()获得Canvas对象并锁定画布 
5. Canvas绘画 
6. SurfaceHolder.unlockCanvasAndPost(Canvas canvas)结束锁定画图，并提交改变，将图形显示。
复制代码

```

其中 4,5,6 都应该在绘图线程中执行，1,2,3 同步变量并且在主线程中执行。  
SurfaceHolder 可以看成是一个 surface 控制器，用来操纵 surface。处理它的 Canvas 上画的效果和动画，控制表面，大小，像素等。

### SurfaceView 的例子

```
public class MySurfaceView extends SurfaceView implements Runnable, SurfaceHolder.Callback {  
    private SurfaceHolder mHolder; // 用于控制SurfaceView
    private Thread t; // 声明一条线程
    private volatile boolean flag; // 线程运行的标识，用于控制线程
    private Canvas mCanvas; // 声明一张画布
    private Paint p; // 声明一支画笔
    float m_circle_r = 10;

    public MySurfaceView(Context context) {
        super(context);

        mHolder = getHolder(); // 获得SurfaceHolder对象
        mHolder.addCallback(this); // 为SurfaceView添加状态监听
        p = new Paint(); // 创建一个画笔对象
        p.setColor(Color.WHITE); // 设置画笔的颜色为白色
        setFocusable(true); // 设置焦点
    }

    /**
     * 当SurfaceView创建的时候，调用此函数
     */
    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        t = new Thread(this); // 创建一个线程对象
        flag = true; // 把线程运行的标识设置成true
        t.start(); // 启动线程
    }

    /**
     * 当SurfaceView的视图发生改变的时候，调用此函数
     */
    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width,
            int height) {
    }

    /**
     * 当SurfaceView销毁的时候，调用此函数
     */
    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        flag = false; // 把线程运行的标识设置成false
        mHolder.removeCallback(this);
    }

    /**
     * 当屏幕被触摸时调用
     */
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        return true;
    }

    /**
     * 当用户按键时调用
     */
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_DPAD_UP) {
        }
        return super.onKeyDown(keyCode, event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        surfaceDestroyed(mHolder);
        return super.onKeyDown(keyCode, event);
    }

    @Override
    public void run() {
        while (flag) {
            try {
                synchronized (mHolder) {
                    Thread.sleep(100); // 让线程休息100毫秒
                    Draw(); // 调用自定义画画方法
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if (mCanvas != null) {
                    // mHolder.unlockCanvasAndPost(mCanvas);//结束锁定画图，并提交改变。

                }
            }
        }
    }

    /**
     * 自定义一个方法，在画布上画一个圆
     */
    protected void Draw() {
        mCanvas = mHolder.lockCanvas(); // 获得画布对象，开始对画布画画
        if (mCanvas != null) {
            Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
            paint.setColor(Color.BLUE);
            paint.setStrokeWidth(10);
            paint.setStyle(Style.FILL);
            if (m_circle_r >= (getWidth() / 10)) {
                m_circle_r = 0;
            } else {
                m_circle_r++;
            }
            Bitmap pic = ((BitmapDrawable) getResources().getDrawable(
                    R.drawable.qq)).getBitmap();
            mCanvas.drawBitmap(pic, 0, 0, paint);
            for (int i = 0; i < 5; i++)
                for (int j = 0; j < 8; j++)
                    mCanvas.drawCircle(
                            (getWidth() / 5) * i + (getWidth() / 10),
                            (getHeight() / 8) * j + (getHeight() / 16),
                            m_circle_r, paint);
            mHolder.unlockCanvasAndPost(mCanvas); // 完成画画，把画布显示在屏幕上
        }
    }
}
复制代码

```

### 参考链接

1.  https://tech.youzan.com/surfaceview-sourcecode/
2.  https://blog.csdn.net/luoshengyang/article/details/8303098
3.  https://www.zhihu.com/question/30922650
4.  https://blog.csdn.net/jam96/article/details/53180093
5.  https://www.jianshu.com/p/15060fc9ef18