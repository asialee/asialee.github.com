---
layout: post
keywords: iOS
title: Objective-c下具有下载功能的NSURLCache类CustomURLCache
categories: [iOS]
tags: [iOS,NSURLCache]
group: archive
icon: code
---


最近在做iOS APP开发的过程中遇到了这么一个问题：我开发的是一个阅读类的App，正文界面是通过UIWebViewController来实现的，现在要实现文章离线阅读功能。即将当前web页面所有的资源请求结果都下载都本地。在网上找了很多方法，发现都不是很好。后来还是决定从cache入手：iOS自带的NSURLCache并不支持将cache下载到自定义的目录底下，所以只有复写NSURLCache这个类来实现这些功能。在网上找到一个他人写的CustomURLCache的类，但是发现使用的过程中程序会崩，于是一步一步调试代码，定位到该类的错误，对代码进行了修改，修复了这个bug。而随着iOS8的推出，发现设置自定义的URLCache必须要在AppDelegate.m的
`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`
函数中使用才行，否则无效。无奈，只好进一步对代码进行修改。终于可以适配ios8了。今天将这部分代码分享出来，希望对大家有用。代码我已经托管到[GitHub](https://github.com/asialee/CustomURLCache)上，大家可以自行下载。

同时为了兼容系统NSURLCache的功能，CustomURLCache有两种模式，NORMAL\_MODE和DOWNLOAD\_MODE，NORMAL_MODE下的CustomCache和NSURLCache的功能是一样的；而DOWNLOAD_MODE下的CustomURLCache则可以实现包含自定义下载目录，设置过期时间的子功能的下载功能。


接下来用例子教大家如何使用CustomURLCache这个类。假设我们在做一个阅读类的App，正文页面用UIWebViewController来展示。现在需要实现一个离线下载的功能，即将当前网页的所有访问到的资源（html文件，图片，JavaScript，CSS等等）全部都保存到本地。用CustomURLCache就可以很容易地实现这个功能。

#在AppDelegate.m中将CustomURLCache设置为全局URLCache#

上文已经讲到了，在iOS8中，设置全局URLCache必须在AppDelegate.m中的`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`实现，所以第一步就是将我们CustomURLCache在该函数中设置为全局URLCache：

{% highlight Objective-c%}
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions{
    ………
    //Init custom cache
    
    CustomURLCache *_mCache = [[CustomURLCache alloc] initWithMemoryCapacity:20 * 1024 * 1024
                                                diskCapacity:200 * 1024 * 1024
                                                    diskPath:nil
                                                   cacheTime:0
                                                subDirectory:nil];
    [NSURLCache setSharedURLCache:_mCache];

    return YES;
}

{% endhighlight%}
这段代码将我们的CustomURLCache设置为了全局的URLCache，其中的参数有如下意义：

- diskCapacity:就是用户设置的这个cache的大小
- diskPath:cache的下载路径，如果设置为nil的话，那么该类将会自动将当前应用的缓存目录作为下载路径
- cacheTime:cache的过期时间，如果设置为0的话，那么cache将永远不会过期
- subDirectory:这是位于diskPath下的子目录，意为当前的cache会存在哪个子目录中。不过在初始化的时候只用将其设置为nil即可

值得说明的时候，在初始化之后其实没有任何变化，因为这个时候的默认模式是NORMAL\_MODE，即跟NSURLCache是一样的。所以感觉不出什么不一样。不要着急，很快我们就需要使用到DOWNLOAD\_MODE了。


#将CustomURLCache设置为DOWNLOAD\_MODE#
好了，假设我们在第一个页面选中了一篇文章，然后点击计入了正文页面，正文页面的布局很简单，就是一个`UIWebView`，我们需要用其显示一个网页，并且将所有需要请求的资源都下载到本地自定义的目录中。这个时候就需要用到DOWNLOAD\_MODE了。实现很简单，在正文界面的UIViewController的`- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil`或者`init`函数中将CustomURLCache设置为DOWNLOAD\_MODE：
{% highlight Objective-c %}
- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        
	//Other Init code…

	
        _mCache = (CustomURLCache *)[NSURLCache sharedURLCache];
        [_mCache changeToDownloadMode:_subDir];
        [NSURLCache setSharedURLCache:_mCache];
    }
    return self;
}
{% endhighlight %}
现在我们就成功地将CustomURLCache设置为了下载模式，在执行`[_mCache changeToDownloadMode:_subDir];`时需要指定下载子目录，就位于diskPath下的下载子目录。接下来所有的web请求资源都会下载到我们的自定义目录了。


#将CustomURLCache恢复到NORMAL\_MODE#
好了，现在web页面也展现出来了，所有的web请求资源都下载到本地了，该退出了。回退到第一个页面，并且不再需要DOWNLOAD\_MODE，我们要将其恢复为NORMAL\_MODE，那么在当前UIViewController下的dealloc函数或者其他用户自定义的退出函数中加入如下代码(在这里我使用的是自己的退出函数)：
{% highlight Objective-c %}
- (void)backToMainpage:(id)sender {
    if (不再需要文章缓存) {
        NSLog(@"删除文章缓存");
        [_mCache removeCustomRequestDictionary ];  
    }
    else
    {
	//do nothing
     }
    //不再使用自定义缓存，而是换到系统的缓存
    [_mCache changeToNormalMode];
    [self.navigationController popViewControllerAnimated:YES];
}
{% endhighlight %}
好了，如果你没有删除文章缓存，那么当我们再次就如正文页面的时候，你会发现页面很快就加在完成了。原因很简单，当你再次进入该UIViewController的时候，程序又会执行`[_mCache changeToDownloadMode:_subDir];`这时，CustomURLCache在接到URL请求的时候会先到_subDir路径下去找是否有缓存的资源，如果找到了，并且还不过期，那么就会自动加在本地的资源，如果没有找到或者过期了，那么会重新通过网络请求。所以你知道为什么这么快就完成了，因为所有的资源都是在本地的！

好了，简单地使用就介绍到这里了。希望对大家有帮助。同时，如果大家喜欢的话就请在GitHub上fork或者star以下吧，万分感谢！









