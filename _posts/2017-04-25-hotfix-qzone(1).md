---
layout: post
title: Android热修复技术——QQ空间补丁方案解析(1)
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
传统的app开发模式下，线上出现bug，必须通过发布新版本，用户手动更新后才能修复线上bug。随着app的业务越来越复杂，代码量爆发式增长，出现bug的机率也随之上升。如果单纯靠发版修复线上bug，其较长的新版覆盖期无疑会对业务造成巨大的伤害，更不要说大型app开发通常涉及多个团队协作，发版排期必须多方协调。
那么是否存在一种方案可以在不发版的前提下修复线上bug？有！而且不只一种，业界各家大厂都针对这一问题拿出了自家的解决方案，较为著名的有腾讯的Tinker和阿里的Andfix以及QQ空间补丁。网上对上述方案有很多介绍性文章，不过大多不全面，中间略过很多细节。笔者在学习的过程中也遇到很多麻烦。所以笔者将通过接下来几篇博客对上述两种方案进行介绍，力求不放过每一个细节。首先来看下QQ空间补丁方案。



### 1. Dex分包机制
大家都知道，我们开发的代码在被编译成class文件后会被打包成一个dex文件。但是dex文件有一个限制，由于方法id是一个short类型，所以导致了一个dex文件最多只能存放65536个方法。随着现今App的开发日益复杂，导致方法数早已超过了这个上限。为了解决这个问题，Google提出了multidex方案，即一个apk文件可以包含多个dex文件。
不过值得注意的是，除了第一个dex文件以外，其他的dex文件都是以资源的形式被加载的，换句话说就是在`Application.onCreate()`方法中被注入到系统的`ClassLoader`中的。这也就为热修复提供了一种可能：将修复后的代码达成补丁包，然后发送到客户端，客户端在启动的时候到指定路径下加载对应dex文件即可。
根据Android虚拟机的类加载机制，同一个类只会被加载一次，所以要让修复后的类替换原有的类就必须让补丁包的类被优先加载。接下来看下Android虚拟机的类加载机制。

### 2. 类加载机制
Android的类加载机制和jvm加载机制类似，都是通过ClassLoader来完成，只是具体的类不同而已：
![1](https://yqfile.alicdn.com/e96e937c3d66731552574aeb21b2a44b77b3a9a1.jpeg)
Android系统通过`PathClassLoader`来加载系统类和主dex中的类。而`DexClassLoader`则用于加载其他dex文件中的类。上述两个类都是继承自`BaseDexClassLoader`，具体的加载方法是`findClass`:
```
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

    /**
     * Constructs an instance.
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     * should be written; may be {@code null}
     * @param libraryPath the list of directories containing native
     * libraries, delimited by {@code File.pathSeparator}; may be
     * {@code null}
     * @param parent the parent class loader
     */
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
}
```
从代码中可以看到加载类的工作转移到了`pathList`中，`pathList`是一个`DexPathList`类型，从变量名和类型名就可以看出这是一个维护Dex的容器：
```
/*package*/ final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String JAR_SUFFIX = ".jar";
    private static final String ZIP_SUFFIX = ".zip";
    private static final String APK_SUFFIX = ".apk";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements;

    /**
     * Finds the named class in one of the dex files pointed at by
     * this instance. This will find the one in the earliest listed
     * path element. If the class is found but has not yet been
     * defined, then this method will define it in the defining
     * context that this instance was constructed with.
     *
     * @param name of class to find
     * @param suppressed exceptions encountered whilst finding the class
     * @return the named class or {@code null} if the class is not
     * found in any of the dex files
     */
    public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
}
```
`DexPathList`的`findClass`也很简单，`dexElements`是维护dex文件的数组，每一个item对应一个dex文件。`DexPathList`遍历`dexElements`，从每一个dex文件中查找目标类，在找到后即返回并停止遍历。所以要想达到热修复的目的就必须让补丁dex在`dexElements`中的位置先于原有dex：
![2](https://yqfile.alicdn.com/8f5f99712021a0fe6f0555b003a9958312787d45.jpeg)![3](https://yqfile.alicdn.com/4b87db83383b91bfb559f551fe8471bffca12cd5.jpeg)
这就是QQ空间补丁方案的基本思路，接下来的博文笔者将以一个实际的例子详述QQ空间补丁热修复的过程