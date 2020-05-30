---
title: KFC 知识点
tags:
---


### 1. 获得相应的签名信息

```
/** 通过包管理器获得指定包名包含签名的包信息 **/
packageInfo = manager.getPackageInfo(pkgname, PackageManager.GET_SIGNATURES);
/******* 通过返回的包信息获得签名数组 *******/
signatures = packageInfo.signatures;
/******* 循环遍历签名数组拼接应用签名 *******/
for (Signature signature : signatures) {
    builder.append(signature.toCharsString());
}
```
MessageDigest 类为应用程序提供信息摘要算法的功能



### 剪切板

 
相关代码

写入数据：

```
ClipboardManager cm=(ClipboardManager)getSystemService(Context.CLIPBOARD_SERVICE); 
cm.setPrimaryClip(ClipData.newPlainText("data", "Jack")); // 或分2步写 ClipData cd = ClipData.newPlain("label","Jack");cm.setPrimaryClip(cd);
Intent intent=new Intent(MainActivity.this,otherActivity.class); 
startActivity(intent); 
```

读取数据：

```
ClipboardManager cm=(ClipboardManager)getSystemService(Context.CLIPBOARD_SERVICE); 
ClipData cd=cm.getPrimaryClip(); 
String msg=cd.getItemAt(0).getText().toString(); 
```

[Android使用剪切板传递数据](https://www.jb51.net/article/183697.htm)