---
layout: post
title: Android热修复技术——QQ空间补丁方案解析(2)
keywords: android
categories: [Android]
tag: [Android]
icon: code
---
接下来的几篇博客我会用一个真实的demo来介绍如何实现热修复。具体的内容包括：

- 如何打包补丁包
- 如何将通过ClassLoader加载补丁包

### 1. 创建Demo
demo很简单，创建一个只有一个Activity的demo：
```
package com.biyan.demo
public class MainActivity extends Activity {

    private Calculator mCal;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mCal = new Calculator();
    }
    public void click(View view) {
        Toast.makeText(this, String.valueOf(mCal.calculate()),Toast.LENGTH_SHORT).show();
    }
}
```

```
Public class Caculoator {
	public float calculate() {
		return 1 / 0;
	}
}
```
demo的代码很简单，运行会出什么bug也很清楚了，在此就不演示了。

### 2.创建补丁包
首先修复Calculator的bug。
```
package com.biyan.demo
Public class Caculoator {
	public float calculate() {
		return 1 / 1;
	}
}
```
重新编译项目，在build目录下找到Calculator.class文件,将其拷出来，准备打包。放置在于Calculator包名相同的路径下。
![这里写图片描述](http://img.blog.csdn.net/20170219013256214?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXNpYUxJWUFaSE9V/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
将其打成jar包：
```
jar -cvf patch.jar com
```

然后再将对应的jar包打成dex包：
```
dx --dex --output=patch_dex.jar patch.jar
```
dx是讲jar包打成dex包的工具，安装在path-android-sdk/build-tools/version(如24.0.0)/dx。
patch_dex.jar就是补丁包，接下来将其安装在sdCard中，接下来应用从sdCard上加载该补丁包。

### 3. 加载补丁
根据上一篇博客的介绍，加载补丁的思路如下：

- 在Application的onCreate()方法中获取应用本身的`BaseDexClassLoader`,然后通过反射得到对应的dexElements
- 创建一个新的DexClassLoader实例，然后加载sdCard上的补丁包，然后通过同样的方法得到对应的dexElements
- 将两个dexElements合并，然后再利用反射将合并后的dexElements赋值给应用本身的`BaseDexClassLoader`

接下来看下具体代码：
```
package com.hotpatch.demo;

import android.app.Application;
import android.os.Environment;
import android.util.Log;
import java.io.File;
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import dalvik.system.DexClassLoader;

/**
 * Created by hp on 2016/4/6.
 */
public class HotPatchApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        // 获取补丁，如果存在就执行注入操作
        String dexPath = Environment.getExternalStorageDirectory().getAbsolutePath().concat("/patch_dex.jar");
        File file = new File(dexPath);
        if (file.exists()) {
            inject(dexPath);
        } else {
            Log.e("BugFixApplication", dexPath + "不存在");
        }
    }

    /**
     * 要注入的dex的路径
     *
     * @param path
     */
    private void inject(String path) {
        try {
            // 获取classes的dexElements
            Class<?> cl = Class.forName("dalvik.system.BaseDexClassLoader");
            Object pathList = getField(cl, "pathList", getClassLoader());
            Object baseElements = getField(pathList.getClass(), "dexElements", pathList);

            // 获取patch_dex的dexElements（需要先加载dex）
            String dexopt = getDir("dexopt", 0).getAbsolutePath();
            DexClassLoader dexClassLoader = new DexClassLoader(path, dexopt, dexopt, getClassLoader());
            Object obj = getField(cl, "pathList", dexClassLoader);
            Object dexElements = getField(obj.getClass(), "dexElements", obj);

            // 合并两个Elements
            Object combineElements = combineArray(dexElements, baseElements);

            // 将合并后的Element数组重新赋值给app的classLoader
            setField(pathList.getClass(), "dexElements", pathList, combineElements);

            //======== 以下是测试是否成功注入 =================
            Object object = getField(pathList.getClass(), "dexElements", pathList);
            int length = Array.getLength(object);
            Log.e("BugFixApplication", "length = " + length);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }

    /**
     * 通过反射获取对象的属性值
     */
    private Object getField(Class<?> cl, String fieldName, Object object) throws NoSuchFieldException, IllegalAccessException {
        Field field = cl.getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(object);
    }

    /**
     * 通过反射设置对象的属性值
     */
    private void setField(Class<?> cl, String fieldName, Object object, Object value) throws NoSuchFieldException, IllegalAccessException {
        Field field = cl.getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(object, value);
    }

    /**
     * 通过反射合并两个数组
     */
    private Object combineArray(Object firstArr, Object secondArr) {
        int firstLength = Array.getLength(firstArr);
        int secondLength = Array.getLength(secondArr);
        int length = firstLength + secondLength;

        Class<?> componentType = firstArr.getClass().getComponentType();
        Object newArr = Array.newInstance(componentType, length);
        for (int i = 0; i < length; i++) {
            if (i < firstLength) {
                Array.set(newArr, i, Array.get(firstArr, i));
            } else {
                Array.set(newArr, i, Array.get(secondArr, i - firstLength));
            }
        }
        return newArr;
    }

}
```
核心代码就这么多，接下来运行一下程序。程序还是Crash了。。。
![DingTalk20170220205018](https://yqfile.alicdn.com/ea0643c460ef9cbbb1bab3fb1a8a3a20ecc09eff.png)
原因是类预校验问题引起的：
- 在apk安装的时候系统会将dex文件优化成odex文件，在优化的过程中会涉及一个预校验的过程
- 如果一个类的static方法，private方法，override方法以及构造函数中引用了其他类，而且这些类都属于同一个dex文件，此时该类就会被打上`CLASS_ISPREVERIFIED`
- 如果在运行时被打上`CLASS_ISPREVERIFIED`的类引用了其他dex的类，就会报错
- 所以`MainActivity`的`onCreate()`方法中引用另一个dex的类就会出现上文中的问题
- 正常的分包方案会保证相关类被打入同一个dex文件
- 想要使得patch可以被正常加载，就必须保证类不会被打上`CLASS_ISPREVERIFIED`标记。而要实现这个目的就必须要在分完包后的class中植入对其他dex文件中类的引用
- 要在已经编译完成后的类中植入对其他类的引用，就需要操作字节码，惯用的方案是插桩。常见的工具有javaassist，asm等。

其实QQ空间补丁方案的关键就在于字节码的注入而不是dex的注入。下一篇博客将会介绍字节码注入的相关细节。


