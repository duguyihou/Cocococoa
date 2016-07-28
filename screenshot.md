```objc
#import <QuartzCore/QuartzCore.h>


if ([[UIScreen mainScreen] respondsToSelector:@selector(scale)])
    UIGraphicsBeginImageContextWithOptions(self.window.bounds.size, NO, [UIScreen mainScreen].scale);
else
    UIGraphicsBeginImageContext(self.window.bounds.size);

[self.window.layer renderInContext:UIGraphicsGetCurrentContext()];
UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
NSData * imgData = UIImagePNGRepresentation(image);
if(imgData)
    [imgData writeToFile:@"screenshot.png" atomically:YES];
else
    NSLog(@"error while taking screenshot");
```

```swift
func captureScreen() -> UIImage
{

    UIGraphicsBeginImageContextWithOptions(self.view.bounds.size, false, 0);

    self.view.drawViewHierarchyInRect(view.bounds, afterScreenUpdates: true)

    let image: UIImage = UIGraphicsGetImageFromCurrentImageContext()

    UIGraphicsEndImageContext()

    return image
}
```
