---
title: Gradle 依赖冲突拾遗
date: 2020-07-02 14:06:43
tags:
---
gd
|Bugly 异常日志|可能原因|
|--|--
|[619611 java.lang.IllegalArgumentException](https://bugly.qq.com/v2/crash-reporting/crashes/e19cf1d736/619611?pid=1&crashDataType=unSystemExit)|Dialog 宿主 Activity 被销毁
|[618417 android.view.WindowManager$BadTokenException](https://bugly.qq.com/v2/crash-reporting/crashes/e19cf1d736/618417?pid=1&crashDataType=unSystemExit)| 同上
[#620304 java.lang.IllegalArgumentException](https://bugly.qq.com/v2/crash-reporting/crashes/e19cf1d736/620304?pid=1&crashDataType=unSystemExit)|Dialog 宿主被销毁后，执行 dismiss
|PH||
[#245835 java.lang.NullPointerException](https://bugly.qq.com/v2/crash-reporting/crashes/c732bd3d03/245835?pid=1&crashDataType=unSystemExit)|空指针异常


肯德基相关数据


## 肯德基的前世今生

## 里程碑事件

## 肯德基的自己的严格

KFC APP 中主要业务


面向客户群体

收入增长
店铺增加数量

互联网下的肯德基： 阴阳师，书


财报：http://www.cninfo.com.cn/new/disclosure/stock?stockCode=832722&orgId=gfbj0832722


早餐


* 门店数量

http://www.kfc.com.cn/kfccda/about.html


http://news.xixik.com/zt/kfc-shop/

书




[](商业模式）肯德基商业运营模式 .doc)

商业模式）肯德基商业运营模式 .doc https://doc.mbalib.com/view/bfc4f780b1b0ab37597a8c8095e0ecc8.html

官网财报 https://ir.yumchina.com/news-releases/news-release-details/yum-china-reports-first-quarter-2020-results

https://zhuanlan.zhihu.com/p/149572851

**** [https://www.weizhuannet.com/p-11528608.html](https://www.weizhuannet.com/p-11528608.html)


[商业模式的定义——做产品到底是做什么](http://www.woshipm.com/pmd/1536950.html)

[商业模式丨肯德基赚钱的四重境界！](https://zhuanlan.zhihu.com/p/38235526)


[百胜中国：快餐业王者归来（附下载）](http://www.199it.com/archives/647288.html)
---


{"data":{"success":1,"challenge":"e10d625aeb03ed84de49b47546b7f7cc","userid":"PH_USER","gt":"9c7a47f9e5fa600e149487d116cbdee9","gtServerStatus":1},"errCode":0}

{"geetest_challenge":"efdd111aa0f77787c18637116fb64466i8","geetest_validate":"26579bc5c228be8fc3b575e88319f9fe","geetest_seccode":"26579bc5c228be8fc3b575e88319f9fe|jordan"}


@ReactMethod
    public void openGT(final ReadableMap readableMap, final Promise promise) {
        boolean isCompletely=false;
        try {
            if(ServiceConfig.isOpenLog) Log.i("applog", "------openGT,"+readableMap);
            JSONObject jsonObject = JSONTools.toJSONObject(readableMap);
            final int type = jsonObject.getInt("type");
            String challenge = jsonObject.getString("challenge");
            int success = jsonObject.getInt("success");
            String gt = jsonObject.getString("gt");
            
            if(success == 1 && !TextUtils.isEmpty(gt) && ){
                
            }
            
            
            try {
                getCurrentActivity().runOnUiThread(new Runnable() {

                    @Override
                    public void run() {

                        GTManager.getInstance().initGT(getCurrentActivity());
                        GTManager.getInstance().startCustomVerify(getCurrentActivity(),type, new GTManager.IGTDialogResult() {
                            @Override
                            public void success(Map<String, String> validateParams,JSONObject json1,JSONObject json2) {
                                if(ServiceConfig.isOpenLog)Log.i("applog", "------success,");
                                JSONObject backJson=new JSONObject();
                                try {
                                    backJson.put("captcha",json1);
                                    backJson.put("result",json2);
                                }catch (Exception e){
                                    e.printStackTrace();
                                }
                                promise.resolve(JSONTools.getWritableMap(backJson));
                            }

                            @Override
                            public void fail() {
                                if(ServiceConfig.isOpenLog)Log.i("applog", "------fail,");
                                JSONObject backJson=new JSONObject();
                                promise.resolve(JSONTools.getWritableMap(backJson));
                            }
                        });
                    }
                });
            } catch (Exception e) {
                e.printStackTrace();
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
