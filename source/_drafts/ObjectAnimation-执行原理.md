---
title: ObjectAnimation 执行原理
tags:
---
```
ObjectAnimator topFlipAnimator = ObjectAnimator.ofFloat(fillbView,"topFlip",-45);
topFlipAnimator.setDuration(2000);
topFlipAnimator.start();
```

#### ofFloat 做了什么？

```
public static ObjectAnimator ofFloat(Object target, String propertyName, float... values) {
    ObjectAnimator anim = new ObjectAnimator(target, propertyName);
    anim.setFloatValues(values);
    return anim;
}
```

```
public void setFloatValues(float... values) {
    if (mValues == null || mValues.length == 0) {
        if (mProperty != null) {
            // 根据 values 创建 PropertyValuesHolder
            setValues(PropertyValuesHolder.ofFloat(mProperty, values));
        } else {
            setValues(PropertyValuesHolder.ofFloat(mPropertyName, values));
        }
    } else {
        super.setFloatValues(values);
    }
}
```

topFlipAnimator.start();

```
start() -> setCurrentPlayTime(0)  -> setCurrentFraction(fraction)  ->  animateValue(currentIterationFraction); -> initAnimation()、animateValue()

void initAnimation() {
    if (!mInitialized) {
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].init();
        }
        mInitialized = true;
    }
}

void init() {
    // 如果 mEvaluator 为 null 使用默认的 Evaluator，否则使用设置的 Evaluator
    if (mEvaluator == null) {
        mEvaluator = (mValueType == Integer.class) ? sIntEvaluator :
                (mValueType == Float.class) ? sFloatEvaluator :
                null;
    }
    if (mEvaluator != null) {
        mKeyframes.setEvaluator(mEvaluator);
    }
}
```

---
ObjectAnimation#animateValue
```
void animateValue(float fraction) {
    final Object target = getTarget();
    if (mTarget != null && target == null) {
        // We lost the target reference, cancel and clean up. Note: we allow null tar
        /// target has never been set.
        cancel();
        return;
    }
    super.animateValue(fraction);
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].setAnimatedValue(target);
    }
}
```
PropertyValuesHolder#setAnimatedValue

```
void setAnimatedValue(Object target) {
    if (mProperty != null) {
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = getAnimatedValue();
            mSetter.invoke(target, mTmpValueArray);// 调用了 View 的 setter 进行绘制
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```

ValueAnimation#animateValue

```
animateValue(float fraction){
    // 使用插值器，根据时间进度计算得出动画的进行度
    fraction = mInterpolator.getInterpolation(fraction);
    mCurrentFraction = fraction;
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].calculateValue(fraction);
    }
}
```

--> calculateValue(float fraction)
```
@Override
void calculateValue(float fraction) {
    mFloatAnimatedValue = mFloatKeyframes.getFloatValue(fraction);
}
```