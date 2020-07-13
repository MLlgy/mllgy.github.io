---
title: RN 入手
tags:
---
RN打包：https://www.jianshu.com/p/5dab81191631


集成到现有原生应用：https://reactnative.cn/docs/integration-with-existing-apps/


RN集成到现有Android原生应用：https://www.jianshu.com/p/09b1351d497c

NPM 使用介绍：https://www.runoob.com/nodejs/nodejs-npm.html


[React native拆包之 原生加载多bundle（iOS&Android）](https://blog.csdn.net/tyro_smallnew/article/details/83660345)


RN jsbundle???
---

Codorvar



----



[Cordova插件开发（1）-Android插件开发详解](https://blog.csdn.net/fxp850899969/article/details/70195569)



[](https://blog.csdn.net/liugang921118/article/details/82345435)



[](https://blog.csdn.net/lifeshow/article/details/51028948?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)


实际上，各平台涉及到本地能力的调用，以插件形式被封装了。（每个插件的实现实际上还是Native模式）。



[](https://blog.csdn.net/weixin_37730482/article/details/73920722?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)




[RN与原生交互（二）——数据传递](https://juejin.im/post/5b20ceb16fb9a01e4f47cd49)(关于交互的数据类型的对应关系 )


[RN系列：Android原生与RN如何交互通信](https://blog.csdn.net/sinat_17775997/article/details/106418224)(关于Android 跳转指定 RN 页面的逻辑)




----

[浅析React Native 原理](https://www.jianshu.com/p/038975d7f22d) Android 加载 JsBundle 的基本逻辑。


---

[react-native热更新之code-push](https://www.cnblogs.com/wood-life/p/10691765.html)



---

[React Native 代码阅读（一）：启动流程（Android）](https://maxiee.github.io/post/ReactNativeCode1md/)

----


### Android 跳转 RN 页面的传值问题

Android 提交 RN 页面的数据

{"page":"walletHome","appId":"main","type":"t1","query":{},"body":{}}

env：uat？release
index：walletHome


Bundle[{env=release, index=walletHome, props={}}]


可以看到打开的页面的路由为 walletHome，通过 props 传递的相关参数。

最终通过 mReactRootView.startReactApplication(mReactInstanceManager, "KFC_RN", bundle ); 将数据通过 Bundle 对象传递给 RN，至于中间如何映射，后续看源码。


## RN 路由

[react-navigation使用详解](https://www.jianshu.com/p/5c070a302192)：这篇博客是极好的，通过它自己可以了解 react-navigation 的相关用法以及知识点。


### one

```
 class MyHomeScreen extends React.Component {
  static navigationOptions = {
    title: 'Home',
  }

  render() {
    return (
      <Button
        onPress={() => this.props.navigation.navigate('Profile', {name: 'Lucy'})}
        title="Go to Lucy's profile"
      />
    );
  }
}

const ModalStack = StackNavigator({
  Home: {
    screen: MyHomeScreen,
  },
  Profile: {
    path: 'people/:name',
    screen: MyProfileScreen,
  },
});
```

使用 `this.props.navigation.navigate('Profile', {name: 'Lucy'})` 进行跳转指定的页面，并且可以配置参数。

### two

StackNavigator(Routeconfigs,StackNavigatorConfig)


创建路由的参数的意义

* Routeconfigs 路由配置


route的配置对象是route name到route config的映射(译者:这才是重点),配置对象告诉 navigator 什么来代表 route.

* StackNavigatorConfig

路由的配置、选项，设置路由的相关属性，用来控制器外观显示等


----


[React navigation的NavigationActions的一些属性使用](https://www.jianshu.com/p/b6387ea89b4d)