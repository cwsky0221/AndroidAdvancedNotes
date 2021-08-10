#### 创建过程：

1.Activity 的 setContentView 方法
首先会进入到代理AppCompatDelegate中
```
@Override
public void setContentView(@LayoutRes int layoutResID) {
    initViewTreeOwners();
    //此处的getDelegate（）返回AppCompatDelegate
    getDelegate().setContentView(layoutResID);
}
```
AppCompatDelegate setContentView:
```
@Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        //解析xml
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mAppCompatWindowCallback.getWrapped().onContentChanged();
    }
```
AppCompatDelegate中的createSubDecor()中初始化contentview，即下图的contentParent然后赋值给window

```
 private void ensureSubDecor() {
        if (!mSubDecorInstalled) {
            //初始化content view，然后赋值给window
            mSubDecor = createSubDecor();           
        }
    }

    private ViewGroup createSubDecor() {
        ensureWindow();

        // Now set the Window's content view with the decor
        mWindow.setContentView(subDecor);
    }

    private void ensureWindow() {
        
        if (mWindow == null && mHost instanceof Activity) {
            attachToWindow(((Activity) mHost).getWindow());
        }
    }
```
这里有个知识点：
```
每个activity都对应一个窗口window，这个窗口是PhoneWindow的实例，PhoneWindow对应的布局是DecirView，是一个FrameLayout，DecorView内部又分为两部分，一部分是ActionBar，另一部分是ContentParent，即activity在setContentView对应的布局。
```

![](https://upload-images.jianshu.io/upload_images/7010367-4969918bb6d51d75.png?imageMogr2/auto-orient/strip|imageView2/2/w/530/format/webp)

window是在哪里创建的？Activity的attach里面：
```
final void attach(Context context, ActivityThread aThread,
Instrumentation instr, IBinder token, int ident,
Application application, Intent intent, ActivityInfo info,
CharSequence title, Activity parent, String id,
NonConfigurationInstances lastNonConfigurationInstances,
Configuration config, String referrer, IVoiceInteractor voiceInteractor,
Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
    mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
    mWindow.setUiOptions(info.uiOptions);
    }
}
```

PhoneWindow的DecorView在哪创建的：
```
private void installDecor() {

}
if (mDecor == null) {
    mDecor = generateDecor(-1);
} else {
    mDecor.setWindow(this);
}

protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}

```
回到AppCompatDelegateImpl看看资源文件解析过程：
```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                  + Integer.toHexString(resource) + ")");
        }
        //主要流程在这里
        View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
        if (view != null) {
            return view;
        }
        XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```
最总调用到这里：

```
private @Nullable
    View tryInflatePrecompiled(@LayoutRes int resource, Resources res, @Nullable ViewGroup root,
        boolean attachToRoot) {
            if (view != null && root != null) {
                // We were able to use the precompiled inflater, but now we need to do some work to
                // attach the view to the root correctly.
                XmlResourceParser parser = res.getLayout(resource);
                try {
                    AttributeSet attrs = Xml.asAttributeSet(parser);
                    advanceToRootNode(parser);
                    ViewGroup.LayoutParams params = root.generateLayoutParams(attrs);

                    if (attachToRoot) {
                        //加载到root，这个root就是contentview
                        root.addView(view, params);
                    } else {
                        view.setLayoutParams(params);
                    }
                } finally {
                    parser.close();
                }
            }
        }
```

最终都会通过ViewGroup的addView方法进行绘制展示

```
public void addView(View child, int index, LayoutParams params) {
        if (DBG) {
            System.out.println(this + " addView");
        }

        if (child == null) {
            throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
        }

        // addViewInner() will call child.requestLayout() when setting the new LayoutParams
        // therefore, we call requestLayout() on ourselves before, so that the child's request
        // will be blocked at our level
        //此处开始绘制
        requestLayout();
        invalidate(true);
        addViewInner(child, index, params, false);
    }
```

```
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        // 真正进行遍历的地方，也就是之后的View绘制流程的入口
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}

```

