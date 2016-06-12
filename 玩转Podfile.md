# 玩转Podfile

## 什么是Podfile
官方只有一句话说明什么是Podfile：The Podfile is a specification that describes the dependencies of the targets of one or more Xcode projects.

大概意思是：Podfile文件是一种规则描述，用于描述一或多个Xcode工程的targets之间的依赖。

Podfile可以很简单：
```
target 'MyApp'
pod 'AFNetworking', '~> 1.0'
```
也可以很复杂：
```
platform :ios, '9.0'
inhibit_all_warnings!

target 'MyApp' do
  pod 'ObjectiveSugar', '~> 0.5'

  target "MyAppTests" do
    inherit! :search_paths
    pod 'OCMock', '~> 2.0.1'
  end
end

post_install do |installer|
  installer.pods_project.pod_targets.each do |target|
    puts "#{target.name}"
  end
end
```

## Podfile全局配置
目前根据官方文档说明，Podfile全局配置只有一个命令：
`install!`
官方说明它的作用是：Specifies the installation method to be used when CocoaPods installs this Podfile.（大概意思是：指定CocoaPods安装Podfile时所使用的安装方法）
例如：
```
install! 'cocoapods',
         :deterministic_uuids => false,
         :integrate_targets => false
 ```

 目前支持的key有：
 ```
:clean
:deduplicate_targets
:deterministic_uuids
:integrate_targets
:lock_pod_sources
```
这个没有见过任何工程里边有人使用过，相信99%的人儿都是使用默认的全局配置。对于这几个key，官方也没有明确说明其功能！

在我们日常开发中，我们可能永远不需要使用到此配置命令，因此大家不用太关注它！

## Dependencies（依赖）
CocoaPods就是用于管理第三方依赖的。我们通过Podfile文件配置来指定工程中的每个target之间与第三方之间的依赖。

有以下三个命令来管理依赖：
* pod 指定特定依赖。比如指定依赖AFNetwroking
* podspec 提供简单的API来创建podspec
* target 在我们的工程中，通过target指定所依赖的范围

## Pod命令
此命令用于指定工程的依赖。我们通过Pod命令指定所依赖的第三方及第三方库的版本范围。
## **永远使用最新版本**
```
pod 'HYBMasonryAutoCellHeight'
```
当我们永远使用远程仓库中的最新版本时，我们只需要指定仓库名即可。当有新的版本发布时，执行pod update命令，会更新至最新的版本。

因为版本之间可能会存在很大的差异，因此我们不应该采用这种方式，而是指定版本范围或者指定特定版本。
## 使用固定版本
```
pod 'HYBLoopScrollView', '2.0'
```
当我们不希望版本更新，而是固定使用指定的版本时，我们应该这么写法。当远程有新的版本发布时，pod是不会去更新新版本的。由于版本变化可能较大，因此有时候我们希望这么做的。
## 指定版本范围
```
pod 'HYBUnicodeReadable', '~>1.1.0'
```
当我们不要求固定版本号，而是指定某个范围时，我们会像上面这么写法。我相信大家在工程中见到最多的就是这种写法了吧。但是，我相信很多朋友并不知道这么写法的意思是什么。

它的意思是：HYBUnicodeReadable的版本可以是1.1.0到2.0.0，但是不包括2.0.0。

使用这种写法是很有用的，因此小版本的升级一般是fix bug，当有bug被fix时，我们确实应该更新。从1.9.9升级到2.0.0时，不会去更新到2.0.0版本。我们认为从2.0.0是一个大版本，大版本的发布，通常不是fix bug，而是增加功能或者改动较大。

那么有哪些符号可以指定范围呢：
* >= version 要求版本大于或者等于version，当有新版本时，都会更新至最新版本
* < version 要求版本小于version，当超过version版本后，都不会再更新
* <= version 要求版本小于或者等于version，当超过version版本后，都不会再更新
* ~> version 比如上面说明的version=1.1.0时，范围在[1.1.0, 2.0.0)。注意2.0.0是开区间，也就是不包括2.0.0。

## 使用本地库
```
pod 'AFNetworking', :path => '~/Documents/AFNetworking
```
如果我们的库是在本地的，那么我们可以通过这样的命令来指定。由于是引用目录，因此外部直接修改目录中的内容，CocoaPods也会更新到最新的，所以也挺不错的！
## 通过仓库的podspec引入
Sometimes you may want to use the bleeding edge version of a Pod. Or a specific revision. If this is the case, you can specify that with your pod declaration.
当我们需要使用库的混合边缘版本，或者指定的修订版本，我们可以通过指定像下面这样的声明。
使用仓库的master（主干）:
```
pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git'
```
不是使用master，而是使用指定的分支:
```
pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :branch => 'dev'
```
使用指定的tag（标签，发布库的版本时，通常版本号与tag号是一致的）:
```
pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :tag => '0.7.0'
```
使用指定的提交版本
```
pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :commit => '082f8319af'
```
官方明确说明要求podspec在根目录下：
The podspec file is expected to be in the root of the repository, if this library does not have a podspec file in its repository yet, you will have to use one of the approaches outlined in the sections below.

也就是说与工程同级！比如AFNetworking中的podspec文件与库目录是同级的，都在根目录下！
## 从外部podspec引入
```
pod 'JSONKit', :podspec => 'https://example.com/JSONKit.podspec'
```
如上，当我们发布到CocoaPods时，如果没有podspec不是在根目录下，而是在外部，可以通过’:podspec’命令来指定外部链接。
## podspec
Use just the dependencies of a Pod defined in the given podspec file. If no arguments are passed the first podspec in the root of the Podfile is used. It is intended to be used by the project of a library. Note: this does not include the sources derived from the podspec just the CocoaPods infrastructure.
大概意思是：使用给定的podspec所指定的pod依赖。如果没有指定参数，根目录下的podspec会被使用。

正常情况下，我们并不需要指定，一般所开源出来的库的podspec都是在根目录下，所以可放心地使用，不用考虑太多。
例如：
```
// 不指定表示使用根目录下的podspec，默认一般都会放在根目录下
podspec
// 如果podspec的名字与库名不一样，可以通过这样来指定
podspec :name => 'QuickDialog'
// 如果podspec不是在根目录下，那么可以通过:path来指定路径
podspec :path => '/Documents/PrettyKit/PrettyKit.podspec'
```

## target
Defines a CocoaPods target and scopes dependencies defined within the given block. A target should correspond to an Xcode target. By default the target includes the dependencies defined outside of the block, unless instructed not to inherit! them.

大概意思是：在给定的块内定义pod的target（Xcode工程中的target）和指定依赖的范围。一个target应该与Xcode工程的target有关联。默认情况下，target会包含定义在块外的依赖，除非指定不使用inherit!来继承（说的是嵌套的块里的继承问题）
例子：

我们指定HYBTestProject这个target可以访问HYBMasonryAutoCellHeight库：
```
target 'HYBTestProject' do
  pod 'HYBMasonryAutoCellHeight', '~>1.1.0'
end
```
指定HYBTestProject这个target可以访问SSZipArchive，但是它不能访问Nimble；但是，HYBTestProjectTests这个target可以访问Nimble，默认是继承块外的，而且这里指定了inherit!继承，因此它也能访问SSZipArchive：
```
target 'HYBTestProject' do
  pod 'SSZipArchive'
  target 'HYBTestProjectTests' do
    inherit! :search_paths
    pod 'Nimble'
  end
end
```

target块内可以有多个target子块：
```
target 'ShowsApp' do
  pod 'ShowsKit'
  # 可以访问ShowsKit + ShowTVAuth，其中ShowsKit是继承于父层的
  target 'ShowsTV' do
    pod 'ShowTVAuth'
  end
  # 可以访问Specta + Expecta
  # 同时也可以访问ShowsKit，它是明确指定继承于父层的所有pod
  target 'ShowsTests' do
    inherit! :search_paths
    pod 'Specta'
    pod 'Expecta'
  end
end
```
注意：Inheriting only search paths。也就是说inherit! :search_paths这是固定的写法。

## Target configuration
这里的配置会使用和控制工程的生成。

### platform
```
platform :ios, '7.0'
platform :ios
```
如果没有指定版本，官方默认值说明如下：

CocoaPods provides a default deployment target if one is not specified. The current default values are 4.3 for iOS, 10.6 for OS X, 9.0 for tvOS and 2.0 for watchOS.

也就是说，若不指定平台版本，各平台默认值如下：
* iOS：4.3
* OS X：10.6
* tvOS：9.0
* watchOS：2.0

### project
默认情况下是没有指定的，当没有指定时，会使用Podfile目录下与target同名的工程：
```
# MyGPSApp只有在FastGPS工程中才会链接
target 'MyGPSApp' do
  project 'FastGPS'
  ...
end
# MyNotesApp这个target只有在FastNotes工程中才会链接
target 'MyNotesApp' do
  project 'FastNotes'
  ...
end
```
一般情况下，我们不指定project，直接使用：
```
target 'MyApp' do
   pod ...
end
```
### inhibit_all_warnings!
inhibit_all_warnings!命令是不显示所引用的库中的警告信息。我们可以指定全局不显示警告信息，也可以指定某一个库不显示警告信息：
```
pod 'SSZipArchive', :inhibit_warnings => true
```
### use_frameworks!
通过指定use_frameworks!要求生成的是framework而不是静态库。
### workspace
默认情况下，我们不需要指定，直接使用与Podfile所在目录的工程名一样就可以了。如果要指定另外的名称，而不是使用工程的名称，可以这样指定：
```
workspace 'MyWorkspace'
```
### source
source是指定pod的来源。如果不指定source，默认是使用CocoaPods官方的source。通常我们没有必要添加。
```
// 如果不想使用官方的，而是在别的地方也有，可以这样指定
source 'https://github.com/artsy/Specs.git'
// 默认是官方的source
source 'https://github.com/CocoaPods/Specs.git'
```
### Hooks
Hooks可以叫它为勾子吧，与swizzling特性差不多，就是在某些操作之前，先勾起，而且让它执行我们特定的操作。
#### plugin
Specifies the plugins that should be used during installation.

Use this method to specify a plugin that should be used during installation, along with the options that should be passed to the plugin when it is invoked.

例如，指定在安装期间使用cocoapods-keys和slather这两个插件：
```
plugin 'cocoapods-keys', :keyring => 'Eidolon'
plugin 'slather'
```
#### pre_install

This hook allows you to make any changes to the Pods after they have been downloaded but before they are installed.

当我们下载完成，但是还没有安装之时，会勾起来，然后可以通过pre_install指定要做的事，做完后才进入安装阶段。

比如：在下载完成但未安装之前，我们就可以指定在干些什么：
```
pre_install do |installer|
  # Do something fancy!
end
```
#### post_install
既然有pre_install命令，自然会想到还有一个与之对应的命令。

This hook allows you to make any last changes to the generated Xcode project before it is written to disk, or any other tasks you might want to perform.

当我们安装完成，但是生成的工程还没有写入磁盘之时，我们可以指定要执行的操作。

比如，我们可以在写入磁盘之前，修改一些工程的配置：
```
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['GCC_ENABLE_OBJC_GC'] = 'supported'
    end
  end
end
```
#### def
我们还可以通过def命令来声明一个pod集：
```
def 'CustomPods'
   pod 'IQKeyboardManagerSwift'
end
```
然后，我们就可以在需要引入的target处引入之：
```
target 'MyTarget' do
   CustomPods
end
```
这么写的好处是：如果有多个target，而不同target之间并不全包含，那么可以通过这种方式来分开引入。
## 逗视项目的Podfile
下面是逗视项目的Podfile，这是经过笔者整理的：
```
platform :ios, '8.0'
use_frameworks!
inhibit_all_warnings!
def shared_pods
  pod 'Alamofire', '~> 3.0'
  pod 'Kingfisher', '~> 1.6'
  pod 'MJRefresh'
  pod 'SDCycleScrollView','~> 1.3'
  pod 'APParallaxHeader'
  pod 'RoundImageView', '~> 1.0.1'
  pod 'StrechyParallaxScrollView', '~> 0.1'
  pod 'TextFieldEffects'
  pod 'IQKeyboardManagerSwift'
  pod 'SwiftyJSON'
  pod 'Validator'
  pod 'Qiniu', '~> 7.0'
  pod 'Google-Mobile-Ads-SDK', '~> 7.0'
end
target 'ds_ios'  do
  shared_pods
end
```
[玩转Podfile](http://www.henishuo.com/podfile-use-perfectly/#rd?sukey=ecafc0a7cc4a741b59baa8160f11c9eeeac34a164a070a5f09e5a1360ab51ff731fed781ba0e8cbb45cc6ec3f417207d)
[官方文档](https://guides.cocoapods.org/syntax/podfile.html#podfile)
