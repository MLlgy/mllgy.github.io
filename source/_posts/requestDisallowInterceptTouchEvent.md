---
title: requestDisallowInterceptTouchEvent
date: 2019-03-14 11:54:33
tags: [View,事件分发机制]
---
### 

调用 `requestDisallowInterceptTouchEvent` 方法会置位 FLAG_DISALLOW_INTERCEPTER 标志位，ViewGroup 将无法拦截除了 ACTION_DOWN 以外的事件,至于 ViewGroup 为什么还是会拦截 ACTION_DOWN 事件，是因为 ViewGruop 在 ACTION_DOWN 事件时会重新置位 FLAG_DISALLOW_INTERCEPTER 标志。