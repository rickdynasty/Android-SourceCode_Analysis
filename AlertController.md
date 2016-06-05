# AlertController 源码分析

本文是基于 [Android 6.0.0]的源码进行分析。

**目录**

- [1. 简介](#1)
- [2. AlertController 与 AlertDialog](#2)
  - [2.1 AlertDialog源码分析](#21)
- [3. AlertController](#3)
  - [3.1 AlertController属性](#31)
  - [3.2 AlertDialogLayout - alert_dialog_holo.xml](#32)
  - [3.3 AlertController - AlertDialog创建过程](#33)
  - [3.4 AlertController - ListView](#34)
  - [3.5 AlertController - Button](#35)
- [4. 结论](#4)
- [5. 参考](#5)

## 1. 简介

AlertController并非一个控件类，很少单独使用，一般是结合AlertDialog、AlertActivity作为他们的内容控制器来使用。这里对AlertController源码进行分析就是为了更好的在我们代码里面使用AlertDialog、AlertActivity。AlertDialog是Android开发者使用非常频繁的控件之一，下面以AlertDialog开头来分析下AlertController，具体使用的demo请参考[Android-burgeon-project](https://github.com/rickdynasty/Android-burgeon-project)(这个工程依赖v7包，可以使用[appcompat_v7](https://github.com/rickdynasty/appcompat_v7)) **class:**AlertDialogSamples.java有大部分AlertDialog使用场景的案例。

## 2. AlertController 与 AlertDialog
先来看几个AlertDialog的界面：

![普通文本对话框](https://raw.githubusercontent.com/rickdynasty/Android-SourceCode_Analysis/master/res/alertDilaog_message.png) ![列表对话框](https://raw.githubusercontent.com/rickdynasty/Android-SourceCode_Analysis/master/res/alertDilaog_list.png) ![带进度条对话框](https://raw.githubusercontent.com/rickdynasty/Android-SourceCode_Analysis/master/res/alertDilaog_progress.png)

...

上面界面显示的内容就是由AlertController来控制的，截图来源[Android-burgeon-project](https://github.com/rickdynasty/Android-burgeon-project)，里面有很多案例；开发者也可以自定义显示视图，就像上面的**带进度条对话框**里面的进度条视图。

AlertDialog创建案例:

```
java

			new AlertDialog.Builder(AlertDialogSamples.this).setIconAttribute(android.R.attr.alertDialogIcon).setTitle(R.string.alert_dialog_two_buttons_msg)
					.setMessage(R.string.alert_dialog_two_buttons2_msg).setPositiveButton(R.string.alert_dialog_ok, new DialogInterface.OnClickListener() {
						public void onClick(DialogInterface dialog, int whichButton) {

							/* User clicked OK so do some stuff */
						}
					}).setNeutralButton(R.string.alert_dialog_something, new DialogInterface.OnClickListener() {
						public void onClick(DialogInterface dialog, int whichButton) {

							/* User clicked Something so do some stuff */
						}
					}).setNegativeButton(R.string.alert_dialog_cancel, new DialogInterface.OnClickListener() {
						public void onClick(DialogInterface dialog, int whichButton) {

							/* User clicked Cancel so do some stuff */
						}
					}).create().show();

```

### 2.1 AlertDialog源码分析
上面的AlertDialog创建案例看不到一点AlertController的身影，这个...就需要来看一下AlertDialog的源码，看看内部是怎样包装AlertController的：

```
java

// AlertDialog.java
package android.app;

public class AlertDialog extends Dialog implements DialogInterface {
    private AlertController mAlert;      

    public static class Builder {
        private final AlertController.AlertParams P;

	....
```
AlertDialog作为Android典型的创建者模式案例，也**只是对AlertController进行了一次很简单的包装**，内部类Builder也是对AlertController.AlertParams进行了一次简单的包装，然后通过这一层包装向外提供相应的api接口，AlertDialog内部结构图：
![AlertDialog内部结构](https://raw.githubusercontent.com/rickdynasty/Android-SourceCode_Analysis/4fefd6c6e69f0b6530055080ceb07467ccc43afd/res/AlertDialog%E5%86%85%E9%83%A8%E7%BB%93%E6%9E%84.png)

>从上面的结构图可以很清晰的看到AlertDialog对Dialog扩展了一个AlertController mAlert；成员，而提供的一系列的公开 api接口都被直接或者间接的转向了这个成员mAlert对应的行为，内部类Builder也是一样的：转向了内部成员AlertController.AlertParams P，`这里主要分析AlertController的源码就不展开细讲AlertDialog的每一个api`，这里面提供的api也很简单就的将行为转向内部成员的行为，部分会附加做一些相应处理，比如涉及到主题或者父类相应的接口调用。
>
>通过上面AlertDialog效果图和AlertDialog的类结构图基本也可以想到AlertController提供了那些能力，^_^不急下面会一步步进行分析。

## 3. AlertController

### 3.1 AlertController属性
```
java

// AlertController.java
package com.android.internal.app;

public class AlertController {

	private final Context mContext; // 上下文
	private final DialogInterface mDialogInterface; // 用于回调
	private final Window mWindow; // 构造的时候传进来，用于查找子view

	private CharSequence mTitle; // 标题文本
	private CharSequence mMessage; // 信息文本
	private ListView mListView; // 列表View

	// 接受使用者调用setView设置进来的View
	private View mView;
	private int mViewLayoutResId;
	private int mViewSpacingLeft;
	private int mViewSpacingTop;
	private int mViewSpacingRight;
	private int mViewSpacingBottom;
	private boolean mViewSpacingSpecified = false;

	//Positive按钮
	private Button mButtonPositive;
	private CharSequence mButtonPositiveText;
	private Message mButtonPositiveMessage;

	//Negative按钮
	private Button mButtonNegative;
	private CharSequence mButtonNegativeText;
	private Message mButtonNegativeMessage;

	//Neutral按钮
	private Button mButtonNeutral;
	private CharSequence mButtonNeutralText;
	private Message mButtonNeutralMessage;

	private ScrollView mScrollView;

	//Icon
	private int mIconId = 0;
	private Drawable mIcon;

	//Icon的承载View
	private ImageView mIconView;
	//标题View
	private TextView mTitleView;
	//MessageView
	private TextView mMessageView;
	//用户自定义的标题View
	private View mCustomTitleView;

	private boolean mForceInverseBackground;

	//用于列表的Adapter
	private ListAdapter mAdapter;

	private int mCheckedItem = -1;

	//AlertDialog布局文件id
	private int mAlertDialogLayout;
	//其他相关Layout id
	private int mButtonPanelSideLayout;
	private int mListLayout;
	private int mMultiChoiceItemLayout;
	private int mSingleChoiceItemLayout;
	private int mListItemLayout;

	private int mButtonPanelLayoutHint = AlertDialog.LAYOUT_HINT_NONE;

	private Handler mHandler;
	...
```

AlertController有一个很重要的内部类，上面也提到过:AlertParams.通过名字比较容易知道是AlertController的属性管理。先看一下AlertParams的属性：

```
java

	public static class AlertParams {
		public final Context mContext;
		public final LayoutInflater mInflater;

		public int mIconId = 0;
		public Drawable mIcon;
		public int mIconAttrId = 0;
		public CharSequence mTitle;
		public View mCustomTitleView;
		public CharSequence mMessage;
		public CharSequence mPositiveButtonText;
		public DialogInterface.OnClickListener mPositiveButtonListener;
		public CharSequence mNegativeButtonText;
		public DialogInterface.OnClickListener mNegativeButtonListener;
		public CharSequence mNeutralButtonText;
		public DialogInterface.OnClickListener mNeutralButtonListener;
		public boolean mCancelable;
		public DialogInterface.OnCancelListener mOnCancelListener;
		public DialogInterface.OnDismissListener mOnDismissListener;
		public DialogInterface.OnKeyListener mOnKeyListener;
		public CharSequence[] mItems;
		public ListAdapter mAdapter;
		public DialogInterface.OnClickListener mOnClickListener;
		public int mViewLayoutResId;
		public View mView;
		public int mViewSpacingLeft;
		public int mViewSpacingTop;
		public int mViewSpacingRight;
		public int mViewSpacingBottom;
		public boolean mViewSpacingSpecified = false;
		public boolean[] mCheckedItems;
		public boolean mIsMultiChoice;
		public boolean mIsSingleChoice;
		public int mCheckedItem = -1;
		public DialogInterface.OnMultiChoiceClickListener mOnCheckboxClickListener;
		public Cursor mCursor;
		public String mLabelColumn;
		public String mIsCheckedColumn;
		public boolean mForceInverseBackground;
		public AdapterView.OnItemSelectedListener mOnItemSelectedListener;
		public OnPrepareListViewListener mOnPrepareListViewListener;
		public boolean mRecycleOnMeasure = true;
```

### 3.2 AlertDialogLayout - alert_dialog_holo.xml

从上面的属性成员来看AlertParams是对AlertController的显示&回调的属性进行细分管理，这里暂不详细讲解每一个成员，下面的使用场景会讲解，先来讲解一个很重要的布局文件：在AlertController属性里面的很多Layout里面有一个非常重要的Layout：**mAlertDialogLayout**，这个布局文件直接决定了界面的显示风格。这个布局文件是通过主题进行配置的：alert_dialog_holo.xml,这个配置主题是：android:Theme.Holo，细心的玩家就知道通过ADT创建出来的project默认theme都是继承自**android:Theme.Holo.Light**.

先来看一下这个布局文件：

```xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/parentPanel"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginStart="8dip"
    android:layout_marginEnd="8dip"
    android:orientation="vertical">

    <!-- 顶部面板：显示标题、icon以及Divider分割块 -->
    <LinearLayout android:id="@+id/topPanel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">
        <View android:id="@+id/titleDividerTop"
            android:layout_width="match_parent"
            android:layout_height="2dip"
            android:visibility="gone"
            android:background="@android:color/holo_blue_light" />
        <LinearLayout android:id="@+id/title_template"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center_vertical|start"
            android:minHeight="@dimen/alert_dialog_title_height"
            android:layout_marginStart="16dip"
            android:layout_marginEnd="16dip">
            <ImageView android:id="@+id/icon"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:paddingEnd="8dip"
                android:src="@null" />
            <com.android.internal.widget.DialogTitle android:id="@+id/alertTitle"
                style="?android:attr/windowTitleStyle"
                android:singleLine="true"
                android:ellipsize="end"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:textAlignment="viewStart" />
        </LinearLayout>
        <View android:id="@+id/titleDivider"
            android:layout_width="match_parent"
            android:layout_height="2dip"
            android:visibility="gone"
            android:background="@android:color/holo_blue_light" />
        <!-- If the client uses a customTitle, it will be added here. -->
    </LinearLayout>

    <!-- 内容显示面板：文本显示 -->
    <LinearLayout android:id="@+id/contentPanel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:orientation="vertical"
        android:minHeight="64dp">
        <ScrollView android:id="@+id/scrollView"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:clipToPadding="false">
            <TextView android:id="@+id/message"
                style="?android:attr/textAppearanceMedium"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:paddingStart="16dip"
                android:paddingEnd="16dip"
                android:paddingTop="8dip"
                android:paddingBottom="8dip"/>
        </ScrollView>
    </LinearLayout>

    <!-- 用户自定面板，用于接收用户设定自己的view -->
    <FrameLayout android:id="@+id/customPanel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:minHeight="64dp">
        <FrameLayout android:id="@+android:id/custom"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
    </FrameLayout>

    <!-- 按钮面板，默认提供了三个按钮分别是：ButtonNegative、ButtonNeutral和ButtonPositive -->
    <LinearLayout android:id="@+id/buttonPanel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:minHeight="@dimen/alert_dialog_button_bar_height"
        android:orientation="vertical"
        android:divider="?android:attr/dividerHorizontal"
        android:showDividers="beginning"
        android:dividerPadding="0dip">
        <LinearLayout
            style="?android:attr/buttonBarStyle"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layoutDirection="locale"
            android:measureWithLargestChild="true">
            <Button android:id="@+id/button2"
                android:layout_width="wrap_content"
                android:layout_gravity="start"
                android:layout_weight="1"
                android:maxLines="2"
                style="?android:attr/buttonBarButtonStyle"
                android:textSize="14sp"
                android:minHeight="@dimen/alert_dialog_button_bar_height"
                android:layout_height="wrap_content" />
            <Button android:id="@+id/button3"
                android:layout_width="wrap_content"
                android:layout_gravity="center_horizontal"
                android:layout_weight="1"
                android:maxLines="2"
                style="?android:attr/buttonBarButtonStyle"
                android:textSize="14sp"
                android:minHeight="@dimen/alert_dialog_button_bar_height"
                android:layout_height="wrap_content" />
            <Button android:id="@+id/button1"
                android:layout_width="wrap_content"
                android:layout_gravity="end"
                android:layout_weight="1"
                android:maxLines="2"
                android:minHeight="@dimen/alert_dialog_button_bar_height"
                style="?android:attr/buttonBarButtonStyle"
                android:textSize="14sp"
                android:layout_height="wrap_content" />
        </LinearLayout>
     </LinearLayout>
</LinearLayout>
```
alert_dialog_holo.xml自上而下一共分了四个面板：topPanel、contentPanel、customPanel、buttonPanel。每一个面板持有的内容都是AlertController在维护的对象,比如：图标、标题、显示的文本内容、自定义视图以及底部的按钮等。


### 3.3 AlertController - AlertDialog创建过程

下就结合开篇举的AlertDialog创建案例，逐步讲解一下AlertController~

AlertDialog创建案例：

```
java

AlertDialog.Builder(AlertDialogSamples.this)
			.setIconAttribute(android.R.attr.alertDialogIcon)
			.setTitle(R.string.alert_dialog_two_buttons_msg)
			.setMessage(R.string.alert_dialog_two_buttons2_msg)
			.setPositiveButton(...)
			.setNeutralButton(...)
			.setNegativeButton(...).create().show();

```



- Step1：AlertDialog.Builder构造

```
java

        public Builder(Context context, int theme) {
			//初始化属性AlertController.AlertParams P
            P = new AlertController.AlertParams(new ContextThemeWrapper(
                    context, resolveDialogTheme(context, theme)));
            mTheme = theme;
        }

```

- Step2：AlertController.AlertParams构造

```
java

		public AlertParams(Context context) {
			mContext = context;
			mCancelable = true;
			mInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
		}
```

- Step3：设置对话框AlertDialog.Builder.P的相关属性setIconAttribute、setTitle、setMessage、set***Button

```
java

        public Builder setIconAttribute(@AttrRes int attrId) {
            TypedValue out = new TypedValue();
            P.mContext.getTheme().resolveAttribute(attrId, out, true);
            P.mIconId = out.resourceId;
            return this;
        }

        public Builder setTitle(@StringRes int titleId) {
            P.mTitle = P.mContext.getText(titleId);
            return this;
        }

        public Builder setTitle(CharSequence title) {
            P.mTitle = title;
            return this;
        }

        public Builder setMessage(@StringRes int messageId) {
            P.mMessage = P.mContext.getText(messageId);
            return this;
        }

        public Builder setMessage(CharSequence message) {
            P.mMessage = message;
            return this;
        }

        public Builder setPositiveButton(@StringRes int textId, final OnClickListener listener) {
            P.mPositiveButtonText = P.mContext.getText(textId);
            P.mPositiveButtonListener = listener;
            return this;
        }

        public Builder setPositiveButton(CharSequence text, final OnClickListener listener) {
            P.mPositiveButtonText = text;
            P.mPositiveButtonListener = listener;
            return this;
        }

        public Builder setNegativeButton(@StringRes int textId, final OnClickListener listener) {
            P.mNegativeButtonText = P.mContext.getText(textId);
            P.mNegativeButtonListener = listener;
            return this;
        }

        public Builder setNegativeButton(CharSequence text, final OnClickListener listener) {
            P.mNegativeButtonText = text;
            P.mNegativeButtonListener = listener;
            return this;
        }

        public Builder setNeutralButton(@StringRes int textId, final OnClickListener listener) {
            P.mNeutralButtonText = P.mContext.getText(textId);
            P.mNeutralButtonListener = listener;
            return this;
        }

        public Builder setNeutralButton(CharSequence text, final OnClickListener listener) {
            P.mNeutralButtonText = text;
            P.mNeutralButtonListener = listener;
            return this;
        }
```
上面的api只是对属性的简单赋值，并将自身返回。

- Step4：创建 AlertDialog.Builder.create()

创建是最核心的移步，先上代码：

```
java

        public AlertDialog create() {
            final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
            P.apply(dialog.mAlert);
            dialog.setCancelable(P.mCancelable);
            if (P.mCancelable) {
                dialog.setCanceledOnTouchOutside(true);
            }
            dialog.setOnCancelListener(P.mOnCancelListener);
            dialog.setOnDismissListener(P.mOnDismissListener);
            if (P.mOnKeyListener != null) {
                dialog.setOnKeyListener(P.mOnKeyListener);
            }
            return dialog;
        }

```

- Step5：AlertDialog构造

```
java

    AlertDialog(Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        super(context, createContextThemeWrapper ? resolveDialogTheme(context, themeResId) : 0,
                createContextThemeWrapper);

        mWindow.alwaysReadCloseOnTouchAttr();
        mAlert = new AlertController(getContext(), this, getWindow());
    }

```

- Step6：AlertController构造

```
java

	public AlertController(Context context, DialogInterface di, Window window) {
		//初始化属性
		mContext = context;
		mDialogInterface = di;
		mWindow = window;
		mHandler = new ButtonHandler(di);

		final TypedArray a = context.obtainStyledAttributes(null, R.styleable.AlertDialog, R.attr.alertDialogStyle, 0);

		//AlertDialogLayout是从配置的主题哪里直接获取的，如果没有获取到就使用默认的：R.layout.alert_dialog
		mAlertDialogLayout = a.getResourceId(R.styleable.AlertDialog_layout, R.layout.alert_dialog);
		//下面Layout也都是从配置的主题里面获取
		mButtonPanelSideLayout = a.getResourceId(R.styleable.AlertDialog_buttonPanelSideLayout, 0);
		mListLayout = a.getResourceId(R.styleable.AlertDialog_listLayout, R.layout.select_dialog);

		mMultiChoiceItemLayout = a.getResourceId(R.styleable.AlertDialog_multiChoiceItemLayout,
				R.layout.select_dialog_multichoice);
		mSingleChoiceItemLayout = a.getResourceId(R.styleable.AlertDialog_singleChoiceItemLayout,
				R.layout.select_dialog_singlechoice);
		mListItemLayout = a.getResourceId(R.styleable.AlertDialog_listItemLayout, R.layout.select_dialog_item);

		a.recycle();
	}

```

在AlertController构造里面主要完成了对mWindow、mDialogInterface、mHandle的初始化；另外一个重要的工作就是完成了Layout根据应用的主题进行赋值。

- Step7：调用**apply**完成AlertParams到AlertController的转换


```
java

		public void apply(AlertController dialog) {
			if (mCustomTitleView != null) {
				dialog.setCustomTitle(mCustomTitleView);
			} else {
				if (mTitle != null) {
					dialog.setTitle(mTitle);
				}
				if (mIcon != null) {
					dialog.setIcon(mIcon);
				}
				if (mIconId != 0) {
					dialog.setIcon(mIconId);
				}
				if (mIconAttrId != 0) {
					dialog.setIcon(dialog.getIconAttributeResId(mIconAttrId));
				}
			}
			if (mMessage != null) {
				dialog.setMessage(mMessage);
			}
			if (mPositiveButtonText != null) {
				dialog.setButton(DialogInterface.BUTTON_POSITIVE, mPositiveButtonText, mPositiveButtonListener, null);
			}
			if (mNegativeButtonText != null) {
				dialog.setButton(DialogInterface.BUTTON_NEGATIVE, mNegativeButtonText, mNegativeButtonListener, null);
			}
			if (mNeutralButtonText != null) {
				dialog.setButton(DialogInterface.BUTTON_NEUTRAL, mNeutralButtonText, mNeutralButtonListener, null);
			}
			if (mForceInverseBackground) {
				dialog.setInverseBackgroundForced(true);
			}
			// For a list, the client can either supply an array of items or an
			// adapter or a cursor
			if ((mItems != null) || (mCursor != null) || (mAdapter != null)) {
				createListView(dialog);
			}
			if (mView != null) {
				if (mViewSpacingSpecified) {
					dialog.setView(mView, mViewSpacingLeft, mViewSpacingTop, mViewSpacingRight, mViewSpacingBottom);
				} else {
					dialog.setView(mView);
				}
			} else if (mViewLayoutResId != 0) {
				dialog.setView(mViewLayoutResId);
			}

			/*
			 * dialog.setCancelable(mCancelable);
			 * dialog.setOnCancelListener(mOnCancelListener); if (mOnKeyListener
			 * != null) { dialog.setOnKeyListener(mOnKeyListener); }
			 */
		}

```

apply函数里面仅仅是将AlertParams里面的数据通过AlertController的api调用设置到了AlertController里面。


- Step8：显示对话框 show()**[AlertDialog.Builder.show()]**

```
java

        public AlertDialog show() {
            AlertDialog dialog = create();
            dialog.show();
            return dialog;
        }

```
这篇就不讲Dialog的显示了，感兴趣的请移步**Dialog源码解析**。
在AlertDialog里面对父类的onCreate进行了复写，这个是在AlertDialog的create()首次调用的时候回调用到。

```
java

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mAlert.installContent();
    }

```
在OnCreate里面执行了一个很重要的步骤：mAlert.installContent();完成对AlertController的显示内容准备工作。

- Step9：准备显示内容 AlertController.installContent()

>整个过程也是通过这一步完成显示内容view的Inflater和view的排版调整，之前的都是数据的准备。

```
java

	public void installContent() {
		/* We use a custom title so never request a window title */
		mWindow.requestFeature(Window.FEATURE_NO_TITLE);
		int contentView = selectContentView();
		//将mAlertDialogLayout设置到mWindow
		mWindow.setContentView(contentView);
		//设置显示的界面
		setupView();
		//设置显示的DecorView
		setupDecor();
	}

	在setupView里面完成了对标题、显示文本、按钮的构建：
	private void setupView() {
		//对contentPanel的mMessageView内容的构建
		setupContent(contentPanel);
		//对buttonPanel的按钮s进行构建
		setupButtons(buttonPanel);
		//对topPanel的标题进行构建
		setupTitle(topPanel);
		...

```


到这整个流程基本走完，另外还有一个主要的功能行为就是用户自定义，这个可以拿系统的ProgressDialog【extends AlertDialog】来做参考，
在ProgressDialog的onCreate方法：

```
java

    @Override
    protected void onCreate(Bundle savedInstanceState) {
			...
            View view = inflater.inflate(a.getResourceId(
                    com.android.internal.R.styleable.AlertDialog_horizontalProgressLayout,
                    R.layout.alert_dialog_progress), null);
            mProgress = (ProgressBar) view.findViewById(R.id.progress);
            mProgressNumber = (TextView) view.findViewById(R.id.progress_number);
            mProgressPercent = (TextView) view.findViewById(R.id.progress_percent);
            setView(view);
			...

```

来看一下setView这个方法【AlertDialog&AlertDialog的内部类Builder都有相应的api】：

```
java

	// AlertDialog.java
    public void setView(View view) {
        mAlert.setView(view);
    }
    
    public void setView(View view, int viewSpacingLeft, int viewSpacingTop, int viewSpacingRight,
            int viewSpacingBottom) {
        mAlert.setView(view, viewSpacingLeft, viewSpacingTop, viewSpacingRight, viewSpacingBottom);
    }

	// AlertDialog.java - AlertDialog.Builder
        public Builder setView(int layoutResId) {
            P.mView = null;
            P.mViewLayoutResId = layoutResId;
            P.mViewSpacingSpecified = false;
            return this;
        }

        public Builder setView(View view) {
            P.mView = view;
            P.mViewLayoutResId = 0;
            P.mViewSpacingSpecified = false;
            return this;
        }
        
        /**
         * @hide
         */
        public Builder setView(View view, int viewSpacingLeft, int viewSpacingTop,
                int viewSpacingRight, int viewSpacingBottom) {
            P.mView = view;
            P.mViewLayoutResId = 0;
            P.mViewSpacingSpecified = true;
            P.mViewSpacingLeft = viewSpacingLeft;
            P.mViewSpacingTop = viewSpacingTop;
            P.mViewSpacingRight = viewSpacingRight;
            P.mViewSpacingBottom = viewSpacingBottom;
            return this;
        }
```

### 3.4 AlertController - ListView
在AlertController内部对ListView进行了简单封装 - 内部类RecycleListView：

```
java

    public static class RecycleListView extends ListView {
        boolean mRecycleOnMeasure = true;

        public RecycleListView(Context context) {
            super(context);
        }

        public RecycleListView(Context context, AttributeSet attrs) {
            super(context, attrs);
        }

        public RecycleListView(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
        }

        public RecycleListView(
                Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
            super(context, attrs, defStyleAttr, defStyleRes);
        }

        @Override
        protected boolean recycleOnMeasure() {
            return mRecycleOnMeasure;
        }
    }

```
上面的RecycleListView就是添加了一个属性mRecycleOnMeasure。但系统AlertController对内部列表的使用进行了包装扩展支持，就开篇的第二屏效果图就是多选列表，**code:**

```
java

//Android-burgeon-project AlertDialogSamples.java

setMultiChoiceItems(R.array.select_dialog_items3, new boolean[] { false, true, false, true, false, false, false }, new DialogInterface.OnMultiChoiceClickListener() {
						public void onClick(DialogInterface dialog, int whichButton, boolean isChecked) {

						}
					})

```

单选or重选 属于CheckedTextView的内容，这里就不分析了。

### 3.4 AlertController - Button
在AlertController属性内部有一个成员变量：

private Handler mHandler; //在构造的时候初始化为内部类ButtonHandler对象


```
java

    private static final class ButtonHandler extends Handler {
        // Button clicks have Message.what as the BUTTON{1,2,3} constant
        private static final int MSG_DISMISS_DIALOG = 1;

        private WeakReference<DialogInterface> mDialog;

        public ButtonHandler(DialogInterface dialog) {
            mDialog = new WeakReference<DialogInterface>(dialog);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {

                case DialogInterface.BUTTON_POSITIVE:
                case DialogInterface.BUTTON_NEGATIVE:
                case DialogInterface.BUTTON_NEUTRAL:
                    ((DialogInterface.OnClickListener) msg.obj).onClick(mDialog.get(), msg.what);
                    break;

                case MSG_DISMISS_DIALOG:
                    ((DialogInterface) msg.obj).dismiss();
            }
        }
    }

```

在AlertController内部持有了三个按钮ButtonNegative、ButtonNeutral和ButtonPositive。点击监听：


```
java

    private final View.OnClickListener mButtonHandler = new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            final Message m;
            if (v == mButtonPositive && mButtonPositiveMessage != null) {
                m = Message.obtain(mButtonPositiveMessage);
            } else if (v == mButtonNegative && mButtonNegativeMessage != null) {
                m = Message.obtain(mButtonNegativeMessage);
            } else if (v == mButtonNeutral && mButtonNeutralMessage != null) {
                m = Message.obtain(mButtonNeutralMessage);
            } else {
                m = null;
            }

            if (m != null) {
                m.sendToTarget();
            }

            // Post a message so we dismiss after the above handlers are executed
            mHandler.obtainMessage(ButtonHandler.MSG_DISMISS_DIALOG, mDialogInterface)
                    .sendToTarget();
        }
    };

```

这样做的目的：告诉系统点击按钮后除了做点击相应的内容外，对话框需要dismiss了。


## 4. 结论
AlertController并非一个控件类，很少单独使用，一般是结合AlertDialog作为他的内容控制器来使用。AlertDialog的相关使用请参考Android-burgeon-project工程。

对AlertController的内部实现能使我们能更好的使用AlertDialog以及派生。


## 5. 参考

[Android-burgeon-project](https://github.com/rickdynasty/Android-burgeon-project)

[Android 6.0.0](https://github.com/xdtianyu/android-6.0.0_r1)