## 远程推送

当服务端远程向APNS推送至一台离线的设备时，苹果服务器Qos组件会自动保留一份最新的通知， 等设备上线后，Qos将把推送发送到目标设备上

远程推送的基本过程

1. 客户端的app需要将用户的UDID和app的bundleID发送给apns服务器,进行注册, apns将加密后的device Token返回给app
2. app获得device Token后,上传到公司服务器
3. 当需要推送通知时,公司服务器会将推送内容和device Token一起发给apns服务器
4. apns再将推送内容送到客户端上

创建证书的流程：

1. 打开钥匙串，生成CertificateSigningRequest.certSigningRequest文件
2. 将CertificateSigningRequest.certSigningRequest上传进developer，导出.cer文件
3. 利用CSR导出P12文件
4. 需要准备下设备token值（无空格）
5. 使用OpenSSL合成服务器所使用的推送证书

本地app代码参考 1.注册远程通知

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions//中注册远程通知
{
[[UIApplication sharedApplication] registerForRemoteNotificationTypes:(UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound)];
}
```

1. 实现几个代理方法：

  ```objc
  //获取deviceToken令牌  
  -(void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken  
  {  
  //获取设备的deviceToken唯一编号  
  NSLog(@"deviceToken=%@",deviceToken);  
  NSString *realDeviceToken=[NSString stringWithFormat:@"%@",deviceToken];  
  //去除<>  
  realDeviceToken = [realDeviceToken stringByReplacingOccurrencesOfString:@"<" withString:@""];  
  realDeviceToken = [realDeviceToken stringByReplacingOccurrencesOfString:@">" withString:@""];  
  NSLog(@"realDeviceToken=%@",realDeviceToken);  
  [[NSUserDefaults standardUserDefaults] setValue:realDeviceToken forKey:@"DeviceToken"];  //要发送给服务器
  }  

  //获取令牌出错  
  -(void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error  
  {  
  //注册远程通知设备出错  
  NSLog(@"RegisterForRemoteNotification error=%@",error);  
  }  
  //在应用在前台时受到消息调用  
  -(void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo  
  {  
  //打印推送的消息  
  NSLog(@"%@",[[userInfo objectForKey:@"aps"] objectForKey:@"alert"]):  
  }
  ```

  一般我们是使用开发版本的Provisioning做推送测试,如果没有问题,再使用发布版本证书的时候一般也应该是没有问题的。 为了以防万一,我们可以在越狱的手机上安装我们的使用发布版证书的ipa文件(最好使用debug版本, 并打印出获取到的deviceToken),安装成功后在;XCode->Window->Organizer-找到对应的设 备查看console找到打印的deviceToken。

在后台的推送程序中使用发布版制作的证书并使用该deviceToken做推送服务. 使用开发和发布证书获取到的deviceToken是不一样的。
