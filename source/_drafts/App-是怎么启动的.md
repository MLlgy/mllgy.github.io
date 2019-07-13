---
title: App 是怎么启动的
tags:
---


源码８.0 

#### 在　Launcher 中点击 App 图标

在系统的　LaucherAcitivty 中有以下代码：

```
//LaucherAcitivty#ActivityAdapter
public Intent intentForPosition(int position) {
    if (mActivitiesList == null) {
        return null;
    }
    Intent intent = new Intent(mIntent);
    ListItem item = mActivitiesList.get(position);
    intent.setClassName(item.packageName, item.className);
    if (item.extras != null) {
        intent.putExtras(item.extras);
    }
    return intent;
}

//LaucherActivity#IconResizer
@Override
protected void onListItemClick(ListView l, View v, int position, long id) {
    Intent intent = intentForPosition(position);
    startActivity(intent);
}
```

由于　startActivity　是重载函数，所以经过一系列的调用，最终会走到　startActivityForResult　这个方法，如下：

```
public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
    if (this.mParent == null) {
        ActivityResult ar = this.mInstrumentation.execStartActivity(this, this.mMainThread.getApplicationThread(), this.mToken, this, intent, requestCode, options);
        ...
    } else if (options != null) {
        this.mParent.startActivityFromChild(this, intent, requestCode, options);
    } else {
        this.mParent.startActivityFromChild(this, intent, requestCode);
    }
}
```

我们需要特别注意的是　mMainThread　这个变量，它的数据类型为　ActivityThread,他就是我们平时所说的主线程/UI 线程，它是在　App 启动时创建的，代表了App 应用程序.

















－－－－
ActivityThread#ActivityClientRecord:　Ａｃｔｉｖｉｔｙ　的客户记录,用于记录真实的　Ａｃｔｉｖｉｔｙ　实例；



----
https://www.jianshu.com/p/72059201b10a

http://www.cloudchou.com/android/post-805.html


[统计App时间三方](https://nimbledroid.com/)

[淘宝优化](https://yq.aliyun.com/articles/2696)