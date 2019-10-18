> 当前 Flutter 版本: 1.9.1+hotfix.4

项目资源管理一直都是应用开发领域炙手可热的话题，资源和源码是组成项目包的主要两个部分，他们都直接影响到应用程序的包大小并且一定程度会影响应用程序的运行速度。此文主要介绍 `Flutter 项目如何管理资源`，同样作为开发者`如何维持资源的可持续化管理`。

# 1. 资源管理

## 1.1. 资源声明

Flutter 使用 `pubspec.yaml` 文件来定位项目的根目录，任何项目中使用到的资源都需要在 `pubspec.yaml` 中进行声明。`pubspecl.yaml` 是个[YAML(/ˈjæməl/)](https://zh.wikipedia.org/wiki/YAML) 文件，资源声明在该文件的 `flutter->assets` 层级上，例如：

```yaml
flutter:
  assets:
    - assets/my_icon.png
    - assets/background.png
```

> 为了统一表述，声明的资源名称我们称之为 `资源标记`。

同样，也可以直接声明文件夹。以文件夹形式声明只会包含此文件夹`根目录`中的文件，`二级目录` 需要另行声明。

```yaml
flutter:
  assets:
    - assets/
    - assets/home/
```

### 1.1.1. 资源变种

Flutter 支持`资源变种`：在不同上下文中使用资源变种的技术，例如 iOS 程序中的 2x/3x 图。当在 `pubspecl.yaml` 中声明资源时，Flutter 会自动查找所有子目录中有关的变种文件并且关联到一起，运行时会根据上下文去获取合适的那个资源。资源变种有几个必须条件：

> **一. Flutter 必须知道该资源的资源标记**

假如项目中资源目录结构为:

```
pubspec.yaml
assets/home/icon.png
assets/home/2x/icon.png
assets/home/3x/icon.png
assets/home/2x/other_icon.png
assets/home/3x/other_icon.png
```

假如以文件夹形式标明资源位置：

```yaml
flutter:
  assets:
    - assets/home/
```

Flutter 只会找到一个资源标记 `assets/home/icon.png`，原因上节已经说明，Flutter 只会查找声明文件夹的根目录中存在的文件。所以只能显示声明每个资源才能达到预期效果：

```yaml
flutter:
  assets:
    - assets/home/icon.png
    - assets/home/other_icon.png
```

> **二. 变种标记必须在最后一个文件夹位置**

假如项目中资源如下: 注意 2x/3x 的位置变化了。

```
pubspec.yaml
assets/home/icon.png
assets/2x/home/icon.png
assets/3x/home/icon.png
assets/3x/home/other_icon.png
assets/3x/home/other_icon.png
```

`pubspec.yaml` 中声明如下：

```yaml
flutter:
  assets:
    - assets/home/icon.png
    - assets/home/other_icon.png
```

运行时使用资源标记不能查找到对应 2x/3x 的资源变种。

## 1.2. 资源在项目中的表现形式

直接看 Flutter 生成文件 `App.framework` 的结构。

```
App.framework
├── App
├── Info.plist
└── flutter_assets
    ├── AssetManifest.json
    ├── assets
    │   └── home
    |       ├── icon.png
    │       ├── 2x
    |       |   ├── icon.png
    │       │   └── other_icon.png
    │       └── 3x
    |           ├── icon.png
    │           └── other_icon.png
```

结论是 Flutter 把项目中的所有资源拷贝进了 `App.framework->flutter_assets`目录中，同时生成了一个缓存文件，该缓存文件保存了`资源标记与变种的对应关系`。内容如下：

```json
{
  "assets/home/icon.png": [
    "assets/home/icon.png",
    "assets/home/2x/icon.png",
    "assets/home/3x/icon.png"
  ],
  "assets/home/other_icon.png": [
    "assets/home/2x/other_icon.png",
    "assets/home/3x/other_icon.png"
  ]
}
```

## 1.3. 资源使用

运行时使用 `AssetBundle` 来访问资源，`AssetBundle` 是个虚类，主要有三个方法：

```dart
/// 加载资源数据，返回字节流
Future<ByteData> load(String key);
/// 加载资源数据，返回 UTF-8 编码的字符串
Future<String> loadString(String key, { bool cache = true });
/// 加载结构化数据，传入解码函数
Future<T> loadStructuredData<T>(String key, Future<T> parser(String value));
```

访问项目资源基本都直接或者间接使用到 `rootBundle` 对象，该对象是 `AssetBundle` 类型。

### 1.3.1. 直接访问资源内容

```dart
import 'dart:async' show Future;
import 'package:flutter/services.dart' show rootBundle;

Future<String> loadAsset() async {
  return await rootBundle.loadString('assets/config.json');
}
```

### 1.3.2. 访问图片

访问图片有两种方式，第一种是直接使用 `Image` 视图控件显示，第二种是使用 `AssetImage`，`AssetImage` 是一个 `ImageProvider` 对象。

> **一. 使用 `Image` 显示**

```dart
@override
Widget build(BuildContext context) {
  return Image.asset(
    "home/icon.png",
    width: 16,
    height: 16,
  );
}
```

> **二. 使用 `AssetImage` 显示**

```dart
@override
Widget build(BuildContext context) {
  return Image(
    image: AssetImage("home/icon.png"),
    width: 16,
    height: 16,
  );
}
```

### 1.3.3. 深入细节

> 以上说明图片主要是由 `Image` 或者 `AssetImage` 访问的，那么系统是如何知道应该是使用哪种资源变种的呢？我们可以深入到源码层面去研究。

首先我们先看一下 `Image.asset` 的源码:

```dart
Image.asset(
  String name, {
  /// 这部分省略
}) : image = scale != null
       ? ExactAssetImage(name, bundle: bundle, scale: scale, package: package)
       : AssetImage(name, bundle: bundle, package: package),
     loadingBuilder = null,
     assert(alignment != null),
     assert(repeat != null),
     assert(matchTextDirection != null),
     super(key: key);
```

可以发现 `Image.asset` 构造方法中 `image` 最终还是会赋值为 `ExactAssetImage/AssetImage`。

- `ExactAssetImage` 是用来拿特定 scale 的资源的，不会涉及到变种。
- `AssetImage` 是根据当前屏幕 scale 自动获取合适的变种。

我们主要将目光放在 `AssetImage` 上，`AssetImage` 实现了 `ImageProvider` 协议。`ImageProvider` 协议需要实现两个方法：

```dart
/// 根据当前设备情况，生成一个 key
Future<T> obtainKey(ImageConfiguration configuration);
/// 根据 Key 获取图片的数据流
ImageStreamCompleter load(T key);
```

再来看看 `AssetImage` 类中对应的实现，`load` 方法是调用原生接口读取 `obtainKey` 返回的文件路径并获取内容，主要匹配逻辑在 `obtainKey` 方法中：

```dart
Future<AssetBundleImageKey> obtainKey(ImageConfiguration configuration) {
  /// 找到当前的资源 bundle
  final AssetBundle chosenBundle = bundle ?? configuration.bundle ?? rootBundle;

  /// 加载 AssetManifest.json
  chosenBundle.loadStructuredData<Map<String, List<String>>>(_kAssetManifestFileName, _manifestParser).then<void>(
(Map<String, List<String>> manifest) {
    /// 找到当前资源标记的所有变种文件
    /// 在所有变种文件中获取最合适的变种
    final String chosenName = _chooseVariant(
      keyName,
      configuration,
      manifest == null ? null : manifest[keyName],
    );
    /// 根据变种地址获取资源 scale
    final double chosenScale = _parseScale(chosenName);
    /// 生成 key
    final AssetBundleImageKey key = AssetBundleImageKey(
      bundle: chosenBundle,
      name: chosenName,
      scale: chosenScale,
    );
  }
}
```

可以看到 Flutter 首先会加载 `AssetManifest.json` 文件，然后根据当前的`资源标记`获取到当前存在的所有资源变种，然后在所有变种选择合适的变种，核心方法 `_chooseVariant` 源码如下：

```dart
/// main 是资源标记
/// config 包含当前项目的屏幕 scale
/// candidates 指的是当前资源标记的所有变种文件，经过排序
String _chooseVariant(String main, ImageConfiguration config, List<String> candidates) {
  if (config.devicePixelRatio == null || candidates == null || candidates.isEmpty)
    return main;
  final SplayTreeMap<double, String> mapping = SplayTreeMap<double, String>();
  for (String candidate in candidates)
    mapping[_parseScale(candidate)] = candidate;
  return _findNearest(mapping, config.devicePixelRatio);
}

String _findNearest(SplayTreeMap<double, String> candidates, double value) {
  if (candidates.containsKey(value))
    return candidates[value];
  /// 找到例当前变种左侧最临近的变种
  final double lower = candidates.lastKeyBefore(value);
  /// 找到例当前变种右侧最临近的变种
  final double upper = candidates.firstKeyAfter(value);
  /// 如果没有左侧邻近的变种则选择右侧变种
  if (lower == null)
    return candidates[upper];
  /// 如果没有右侧邻近的变种则选择左侧变种
  if (upper == null)
    return candidates[lower];
  /// 如果当前屏幕 scale 比左侧和右侧的平均值大，则选择右侧变种
  if (value > (lower + upper) / 2)
    return candidates[upper];
  /// 选择左侧变种
  else
    return candidates[lower];
}
```

这两个方法概括一句话就是`选择当前存在的变种中最邻近的变种`。举几个例子来帮助理解：

- 假如项目中只存在 1x/4x 图，那么 1/2/3/4 屏幕密度上分别选择的是: 1/1/4/4
- 假如项目中只存在 2x/3x 图，那么 1/2/3/4 屏幕密度上分别选择的是: 2/2/3/3
- 假如项目中只存在 1x/3x 图，那么 1/2/3/4 屏幕密度上分别选择的是: 1/1/3/3
- 假如项目中只存在 2x/4x 图，那么 1/2/3/4 屏幕密度上分别选择的是: 2/2/2/4

> 可以看到最合理的情况下是只提供 2x/3x 图。

总结一下，Flutter 资源查找流程分为 4 步：

1. 获取资源配置文件 `AssetManifest.json` 内容，找到资源标记与变种文件的关系。
2. 根据当前资源标记获取其所有的变种文件。
3. 查找出当前存在的变种中与当前屏幕 scale 最邻近的变种。
4. 通过 Native API 获取变种文件的内容，传输给 Flutter 端进行显示。

## 1.4. 总结

> 以上便是 Flutter 主要的资源管理以及访问的流程，至于还有其他的例如访问依赖包中的资源等等情况，整体流程异曲同工的，所以没有一一列举。

根据以上的流程我们可以总结出 Flutter 资源管理存在的问题：

1. 以文件夹形式在 `pubspec.yaml` 中标明资源标记的情况下，除非所有资源都有对应的一倍图存在于根目录下，资源的其他变种才会发挥作用。
2. 资源是直接拷贝进 `App.framework/flutter_assets/` 文件夹中，不在 App Bundle 的根目录下，不会享受 Xcode 的自动 PNG 图片压缩。同时也不会享受 [Assets Slicing](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f)。

问题一有两个思路去解决：第一个思路：只在 `pubspec.yaml` 声明文件夹，每个资源都提供一倍图，但是现在 1x 的屏幕设备比例相当少，为了防止安装包体积的问题，伪造一个空的一倍图来代替；第二个思路：每个资源都在 `pubspec.yaml` 显式声明。实践过程中第一个思路会大大影响项目资源的管理过程，所以我们选择了第二个思路。

问题二解决思路就比较局限了，我们曾经的想法是资源还是放在原生端，通过 `MethodChannel` 来获取资源并且显示，那样资源可以自动被压缩并且切片。但是这样管理起来会相当麻烦并且实现成本也比较大。所以我们使用了退而求其次的方式：不考虑资源切片，但是必须要进行压缩。

至此整体的资源添加流程可以概括为：

1. 设计师提供视觉资源
2. 将视觉资源进行压缩处理
3. 将视觉资源添加进项目
4. 在 `pubspec.yaml` 中声明资源标记

不得不说，太复杂了，而且随着项目的增大，项目中的资源会越来越难以管理。所以就有了我们资源自动化的方案。

# 2. 自动化

> 资源自动化管理工具：[auto-assets](https://github.com/raomengyun/auto_assets)

## 2.1. 工具由来

资源自动化要解决的问题很简单，就是让添加资源到项目这个过程变得简单并且纯粹。最简单的流程是只需要把资源添加到项目，其他流程不用去关心。基于此需求上，自动化要解决的问题就变得明确了：

- 自动生成 `pubspec.yaml` 声明
- 自动压缩资源

至此，添加资源这个动作就已经变得简单且纯粹了。但是回想做 iOS 开发的过程中，最难的往往是可持续化的资源管理。或许大家都遇到过以下情况：

- 资源名称被修改了，项目里面引用的名称被修改。
- 资源被删除了，项目里面引用未删除。

以上问题主要的原因是 HardCode，解决的方式可以参考 `R.java`，主要思想是`资源类型化`。例如每个资源都生成一个对应的实例，使用资源改变为使用该实例：

```dart
class Assets {
   Assets._();
   static const String homeIcon = "assets/home/icon.png";
   static const String homeOtherIcon = "assets/home/other_icon.png";
}
```

## 2.2. 工具原理

工具原理其实非常简单，这里也不展开讲了，主要归纳为以下几个步骤：

1. 监听`资源根目录`
2. 资源修改/添加/删除的事件后
3. 重新生成 pubspec.yaml 声明
4. 重新压缩资源
5. 重新生成`资源类型化`代码

## 2.3. 工具限制

- 资源路径字符集：[a-z0-9_/]，建议使用 [snake_case](https://en.wikipedia.org/wiki/Snake_case)。
- 资源压缩目前只支持 `jpg`/`jpeg`/`png`/`svg`。

## 2.4. 工具安装

> 在语言选型的过程中也考虑过时候 Dart 来开发，但是 Dart Build Runner 还不够完善和成熟，为了开发速度最后还是选择了 [nodejs](https://nodejs.org/en/)。

auto_assets 是使用 nodejs 开发的工具，安装需要有 node 环境。

1. 安装 nodejs：https://nodejs.org/en/download/
2. 执行命令：npm install auto_assets -g
3. 安装完之后即可使用 auto_assets 命令

## 2.5. 工具使用

在 Flutter 项目根目录下新建 `assetsconfig.json` 文件，文件内容：

```json
{
  "assets": "assets/",
  "code": "lib/assets/"
}
```

- `assets` 代表项目中资源文件的根目录，有多个的时候可以传入数组。
- `code` 代表自动生成的代码存放的目录。

在命令行输入：

````
auto_assets [Flutter项目根目录]
````

## 2.6. VSCode Extension

使用 VSCode 开发的同学可以直接在插件商店里面搜索 [auto_assets](https://marketplace.visualstudio.com/items?itemName=mumu.auto-assets)。