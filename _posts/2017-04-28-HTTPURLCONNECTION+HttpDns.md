---
layout: post
title: HttpClient/HttpURLConnection + HttpDns最佳实践
keywords: android
categories: [Android]
tag: [Android]
icon: code
---
在Android端如果OkHttp作为网络请求框架，由于其提供了自定义DNS服务接口，可以很优雅地结合HttpDns，相关实现可参考:[HttpDns+OkHttp最佳实践](https://help.aliyun.com/document_detail/52008.html?spm=5176.product30100.6.585.USrgj7)。
如果您使用`HttpClient`或`HttpURLConnection`发起网络请求，尽管无法直接自定义Dns服务，但是由于`HttpClient`和`HttpURLConnection`也通过`InetAddress`进行域名解析，通过修改`InetAddress`的DNS缓存，同样可以比通用方案更为优雅地使用HttpDns。

[InetAddress](https://developer.android.com/reference/java/net/InetAddress.html)在虚拟机层面提供了域名解析能力，通过调用`InetAddress.getByName(String host)`即可获取域名对应的IP。调用`InetAddress.getByName(String host)`时，`InetAddress`会首先检查本地是否保存有对应域名的ip缓存，如果有且未过期则直接返回；如果没有则调用系统DNS服务（Android的DNS也是采用NetBSD-derived resolver library来实现）获取相应域名的IP，并在写入本地缓存后返回该IP。

核心代码位于`java.net.InetAddress.lookupHostByName(String host, int netId)`

```java
public class InetAddress implements Serializable {
  ...
      /**
     * Resolves a hostname to its IP addresses using a cache.
     *
     * @param host the hostname to resolve.
     * @param netId the network to perform resolution upon.
     * @return the IP addresses of the host.
     */
    private static InetAddress[] lookupHostByName(String host, int netId)
            throws UnknownHostException {
        BlockGuard.getThreadPolicy().onNetwork();
        // Do we have a result cached?
        Object cachedResult = addressCache.get(host, netId);
        if (cachedResult != null) {
            if (cachedResult instanceof InetAddress[]) {
                // A cached positive result.
                return (InetAddress[]) cachedResult;
            } else {
                // A cached negative result.
                throw new UnknownHostException((String) cachedResult);
            }
        }
        try {
            StructAddrinfo hints = new StructAddrinfo();
            hints.ai_flags = AI_ADDRCONFIG;
            hints.ai_family = AF_UNSPEC;
            // If we don't specify a socket type, every address will appear twice, once
            // for SOCK_STREAM and one for SOCK_DGRAM. Since we do not return the family
            // anyway, just pick one.
            hints.ai_socktype = SOCK_STREAM;
            InetAddress[] addresses = Libcore.os.android_getaddrinfo(host, hints, netId);
            // TODO: should getaddrinfo set the hostname of the InetAddresses it returns?
            for (InetAddress address : addresses) {
                address.hostName = host;
            }
            addressCache.put(host, netId, addresses);
            return addresses;
        } catch (GaiException gaiException) {
          ...
        }
    }
}
```

其中`addressCache`为`InetAddress`的本地缓存：

```java
private static final AddressCache addressCache = new AddressCache();
```



结合`InetAddress`的解析策略，我们可以通过如下方法实现自定义DNS服务：

- 通过HttpDns SDK获取目标域名的ip
- 利用反射的方式获取到`InetAddress.addressCache`对象
- 利用反射方式调用`addressCache.put()`方法，域名和ip的对应关系写入`InetAddress`缓存

具体实现可参考以下代码：

```java
public class CustomDns {

    public static void writeSystemDnsCache(String hostName, String ip) {
        try {
            Class inetAddressClass = InetAddress.class;
            Field field = inetAddressClass.getDeclaredField("addressCache");
            field.setAccessible(true);
            Object object = field.get(inetAddressClass);
            Class cacheClass = object.getClass();
            Method putMethod;
            if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                //put方法在api21及以上为put(String host, int netId, InetAddress[] address)
                putMethod = cacheClass.getDeclaredMethod("put", String.class, int.class, InetAddress[].class);
            } else {
                //put方法在api20及以下为put(String host, InetAddress[] address)
                putMethod = cacheClass.getDeclaredMethod("put", String.class, InetAddress[].class);
            }
            putMethod.setAccessible(true);
            String[] ipStr = ip.split("\\.");
            byte[] ipBuf = new byte[4];
            for(int i = 0; i < 4; i++) {
                ipBuf[i] = (byte) (Integer.parseInt(ipStr[i]) & 0xff);
            }
            if(Build.VERSION.SDK_INT  >= Build.VERSION_CODES.LOLLIPOP) {
                putMethod.invoke(object, hostName, 0, new InetAddress[] {InetAddress.getByAddress(ipBuf)});
            } else {
                putMethod.invoke(object, hostName, new InetAddress[] {InetAddress.getByAddress(ipBuf)});
            }

        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```

和通用方案相比，使用该方法具有下列优势：

- 实现简单
- 通用性强，该方案在HTTPS,SNI以及设置Cookie等场景均适用。规避了证书校验，域名检查等环节
- 全局生效，`InetAddress.addressCache`为全局单例，该方案对所有使用`InetAddress`作为域名解析服务的请求全部生效

<font color="red">另外使用该方案请务必注意以下几点：</font>

- `AddressCache`的默认TTL为2S，且默认最多可以保存16条缓存记录:

  >```java
  >class AddressCache {
  >	...
  >    /**
  >     * When the cache contains more entries than this, we start dropping the oldest ones.
  >     * This should be a power of two to avoid wasted space in our custom map.
  >     */
  >    private static final int MAX_ENTRIES = 16;
  >
  >    // The TTL for the Java-level cache is short, just 2s.
  >    private static final long TTL_NANOS = 2 * 1000000000L;
  >    }
  >}
  >```
  > Android虚拟机下反射规则与JVM存在差异，无法直接修改final变量的值。所以使用该方法请务必注意IP过期时间及缓存数量。另外针对该问题可尝试另一种解决方案：重写AddressCache类，并通过ClassLoader优先加载，覆盖系统类。

- `AddressCache.put`方法在 API 21进行了改动，增加了`netId`参数，为保证兼容性需要针对不同版本区别处理。具体方案参考上文代码

- 该方式可以解决HTTPS，SNI以及设置cookie等场景，但不适用于WebView场景。Android Webview使用`Chromium`或`Webkit`作为内核(Android 4.4开始，Webview内核由Chromium替代Webkit)。上述两者均绕开InetAddress而直接使用系统DNS服务，所以该方案对此场景无效。