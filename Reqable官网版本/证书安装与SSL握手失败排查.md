# Reqable 证书安装与 SSL 握手失败排查

> HTTPS 流量要经过 Reqable 的中间人代理才看得到明文，前提是手机信任它生成的 CA 根证书。本篇讲该装用户证书还是系统证书、Chrome 与 Firefox 为什么要单独处理，以及证书装好后个别应用仍然解析不出内容的几种原因。
> **相关文档**：[抓不到流量怎么排查.md](抓不到流量怎么排查.md) · [独立模式与协同模式怎么选.md](独立模式与协同模式怎么选.md) · [下载与安装教程.md](下载与安装教程.md)

---

> [!IMPORTANT]
> **Reqable 安装文件资源（夸克网盘）**：[https://pan.quark.cn/s/ea9427b6532d](https://pan.quark.cn/s/ea9427b6532d)

---

## 一、不装证书会怎样

HTTPS 是加密的。Reqable 用自己生成的一张 CA 根证书，替真实服务器重签一份证书给手机上的应用看；应用（或系统）必须信任这张根证书，握手才做得下去。证书没装或没被信任，抓到的只会是一片 `SSL 握手失败`，看不到任何内容。

所以判据很简单：**只看 HTTP 明文可以不装证书；要看 HTTPS 内容就必须装。**

## 二、装哪一张证书

这一步最容易搞错，先对上自己的用法：

| 你的用法 | 要装到手机上的证书 |
| --- | --- |
| 用电脑端 Reqable 分析手机流量（协同模式） | 电脑端生成的根证书 |
| 手机端独立模式直接抓包 | 手机端生成的根证书 |
| 协同模式扫码连电脑 | 不用自己准备，连上后电脑端证书会自动同步过来 |

Reqable 给每台设备生成的根证书都不一样。想让几台设备共用同一张，可以在桌面端导出 `.p12` 格式的证书，再到其他设备上导入。

## 三、Android 的两种证书目录

| 目录 | 增删是否要 Root | 覆盖面 |
| --- | --- | --- |
| 用户证书 | 不需要 | 只对信任用户证书的应用生效；Android 7.0 起系统默认不信任 |
| 系统证书 | 需要 Root | 系统级信任，覆盖面更大 |

**用户证书**的安装路径（不同品牌叫法略有差异）：

```
设置 → 安全 → 加密与凭据 → 安装证书 → CA 证书
```

选中下载或导出的证书文件，按提示验证锁屏密码或指纹。

**系统证书**支持在 Reqable 里一键安装，前提是设备已经 Root（官方标注覆盖 Android 5.0 – 15）。要用这个功能，电脑上得先装好 ADB——Reqable 靠 ADB 去检查已连接设备的证书安装状态。ADB 装好后配置好 `ANDROID_HOME` 和 `PATH` 环境变量，重启 Reqable 才会生效。

一个已知的显示问题：非 Root 设备上 Reqable 读不到用户证书的安装状态，证书页会一直显示「未知证书安装状态」。这是正常的，不代表没装上。

## 四、证书装好了，个别 App 还是解析不了

按可能性从高到低：

1. **应用自己不信任用户证书**。Android 7.0 起，应用默认不信任用户目录的 CA。官方给的两种让应用信任的方式，是写给你自己开发的 App 用的：
   - 方式一，在 `build.gradle` 里加依赖，Debug 包会自动带上网络安全配置：

     ```gradle
     dependencies {
         debugImplementation("com.reqable.android:user-certificate-trust:1.0.0")
     }
     ```

     连不上 Maven 中央仓库时，用方式二手动配置。

   - 方式二，新建 `res/xml/network_security_config.xml`：

     ```xml
     <?xml version="1.0" encoding="utf-8"?>
     <network-security-config>
       <base-config cleartextTrafficPermitted="true">
         <trust-anchors>
           <certificates src="system" />
           <certificates src="user" />
         </trust-anchors>
       </base-config>
     </network-security-config>
     ```

     再在 `AndroidManifest.xml` 的 `<application>` 上挂 `android:networkSecurityConfig="@xml/network_security_config"`。官方提醒：这段配置要在发行版本里去掉。另外，这种方式只对 Android Native 应用有效，基于 Flutter 的应用不管用。

2. **应用内置了自己的 CA Store**，不读系统证书库。这种应用装了系统证书也照样不认。Firefox 是这一类里最常见的例子，好在它留了开关（见下一节）；别的应用这么做，就只能由应用方给方案。

3. **固定证书（SSL Pinning）**：应用只认服务器那张证书，Reqable 签发的中间人证书过不了校验。

4. **双向验证**：服务器要求客户端也上传证书，默认情况下 Reqable 不带客户端证书上去，连接会被直接拒掉。

第 3、4 两类是应用或服务端一侧的安全设置，Reqable 这边没有通用办法，只能由应用开发者或服务端调整——官方 FAQ 里也是这么写的。

## 五、Chrome 和 Firefox 要单独处理

### Chrome

Chrome 对自签 CA 的信任策略一直在变，较新的版本会忽略装在**系统证书目录**里的自签 CA。所以：

| Chrome 版本 | 证书装到哪 |
| --- | --- |
| 高版本 | 用户证书目录 |
| 低版本 | 系统证书目录 |

拿不准就把两个目录都装一遍。

### Firefox

Firefox 用自带的 CA Store，装到系统里也不生效，得先在它自己的调试菜单里放开：

1. Firefox 设置 → 关于 Firefox → 连点顶部 Logo 5 下，启用调试菜单。
2. Firefox 设置 → Secret Settings → 打开 `Use third party CA certificates`。
3. 再把 CA 证书装到**用户证书目录**。

## 六、怎么确认证书到底生效了

- **应用内自查**：侧边栏进证书管理页。没装好会显示红色的「证书未安装」提示，下面附对应平台的安装指引；协同模式下手机端用的就是电脑端同步过来的那张根证书（见第二节）。
- **实际验证**：打开调试开关，用手机浏览器访问一个网页。网页打不开、调试列表里也一条记录都没有，可能是证书没生效，也可能是端口的问题——两者的区分办法在 [抓不到流量怎么排查.md](抓不到流量怎么排查.md) 的三种情况表里。
