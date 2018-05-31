#分享渠道
#WhatsApp
*无需集成SDK*
##App检查Scheme
**whatsapp://**

##Method 1 - Scheme

```
let shareTextPrefix = "whatsapp://msg/text/"
let encodedShareText = text.addingPercentEncoding(withAllowedCharacters: .illegalCharacters)
UIApplication.shared.openURL(URL(string: shareTextPrefix + encodedShareTextTmp))

```

##Method 2 - Share Extension

接入系统分享入口，UIActivityViewController  
* String 纯文本；   
* URL 分享Remote链接，可被自动抓取；可和String组合使用  
* UIImage 分享图片，只能单独使用；存在String或者Remote URL将无效；   
* URL Local链接，图片或者Video；存在String或者Remote URL将无效;    


##Method 3 - UIDocumentInteractionController

```
UIImage *iconImage = [UIImage imageNamed:@"icon"];
NSString *savePath  = [NSHomeDirectory() stringByAppendingPathComponent:@"Documents/icon.wai"];
[UIImageJPEGRepresentation(iconImage, 1.0) writeToFile:savePath atomically:YES];
_documentInteractionController = [UIDocumentInteractionController interactionControllerWithURL:[NSURL fileURLWithPath:savePath]];
_documentInteractionController.UTI = @"net.whatsapp.image";
_documentInteractionController.delegate = self;
[_documentInteractionController presentOpenInMenuFromRect:CGRectZero inView:self.view animated: YES];
```

#Line
*无需集成SDK*

##App检查Scheme
**line://**

##分享规则
```
line://msg/<CONTENT TYPE>/<CONTENT KEY>

CONTENT TYPE:
1. text
2. image

```
##分享文字
```
let shareTextPrefix = "line://msg/text/"
let encodedShareText = text.addingPercentEncoding(withAllowedCharacters: .illegalCharacters)
```

* Line会自动抓取文案中的链接对应的卡片信息；

##分享图片(未公开API)

```
Specify value in percent encoded (utf-8) text.
By principle, only the page title and page URL may be specified.
* When sending from iPhone apps, please attach the image to the Pasteboard and set a PasteboardName in the following format: line://msg/image/(PasteboardName)
* When sending from Android devices, please specify a local image path that can be accessed by LINE in the following format: line://msg/image/(localfilepath)
* Specifying information irrelevant to the page is prohibited under the Guidelines.
```

Code  
```
- (BOOL)sharePicture:(NSString *)pictureUrl
{
    UIPasteboard *pasteboard = [UIPasteboard generalPasteboard];
    NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:pictureUrl]];
    UIImage *image = [UIImage imageWithData:data];
    [pasteboard setData:UIImageJPEGRepresentation(image, 0.9) forPasteboardType:@"public.jpeg"];
    NSString *contentType =@"image";
    NSString *contentKey = [pasteboard.name stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
    
    NSString *urlString = [NSString stringWithFormat:@"line://msg/%@/%@",contentType, contentKey];
    NSURL *url = [NSURL URLWithString:urlString];
    
    if ([[UIApplication sharedApplication] canOpenURL:url]) {
        return [[UIApplication sharedApplication] openURL:url];
    }else{
        return NO;
    }
}
```

