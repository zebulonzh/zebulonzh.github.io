---
layout: post
title: 'cocos3.0+ListView等出现绿色显示错误'
date: 2015-07-14 19:49
categories: Cocos
excerpt:
---

* 问题
      使用了ListView,ScrollView,PageView等控件后（尤其是在cocostudio中使用），安卓设备上会出现绿色的区域，而不是这些控件
* 解决
      只需要修改AppActivity.java，将TestCpp里面的代码直接拷贝进去覆盖，即可解决上述问题。
主要是这个函数:


``` cpp
public Cocos2dxGLSurfaceView onCreateView() {  
       Cocos2dxGLSurfaceView glSurfaceView = new Cocos2dxGLSurfaceView(this);  
       // TestCpp should create stencil buffer  
       glSurfaceView.setEGLConfigChooser(5, 6, 5, 0, 16, 8);  
       return glSurfaceView;  
}
```