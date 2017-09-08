#iOS WKWebView使用归纳 
**TopBuzz  3.9版本尝试接入WKWebView，这里总结下接入过程中遇到的问题，以便后来者查阅。**
##Case 1 -- 访问Sandbox资源权限受限
WKWebView加载资源的接口: 

```
// just like UIWebView
// m1
open func load(_ request: URLRequest) -> WKNavigation?
// m2
open func loadHTMLString(_ string: String, baseURL: URL?) -> WKNavigation? 
// m3
open func load(_ data: Data, mimeType MIMEType: String, characterEncodingName: String, baseURL: URL) -> WKNavigation?

// WKWebView unique
// m4
@available(iOS 9.0, *)
open func loadFileURL(_ URL: URL, allowingReadAccessTo readAccessURL: URL) -> WKNavigation?  

```
以上接口中，直接使用m1~m3进行资源加载时，WebView是不能通过file:///路径直接获取sandbox中的图片等资源文件的，考虑到WKWebView时独立进程，这个设定也是可以接受的。 
使用m4，可以增加允许访问的目录或者文件的URL，WKWebView就可以直接访问制定目录下的资源了。 
##Case 2 -- loadFileURL资源路径受限
**使用loadFileURL接口时，文档html的workspace必须存储在Sandbox中，不可以直接访问MainBundle目录下的workspace。**
##Case 3 -- loadFileURL资源编码后缀受限
**html主文档的编码格式决定了，WKWebView加载资源的编码格式(如果其他资源中没有指定具体的编码信息)，基本标准：编码使用utf-8。** 

```
<head> 
  <meta http-equiv=\"content-type\" content=\"text/html;charset=utf-8\" />
</head>
```
**限制encode后，主文档文件后缀不可以随便指定，必须是.htm或者.html。**
##Case 4 -- WKWebView中zepto tap有点透问题
**使用click代替**
##Case 5 -- WKWebView视图默认可缩放
**<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=0">**