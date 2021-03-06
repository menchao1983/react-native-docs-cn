有时候App需要访问平台API，但React Native可能还没有相应的模块包装；或者你需要复用一些Java代码，而不是用Javascript重新实现一遍；又或者你需要实现某些高性能的、多线程的代码，譬如图片处理、数据库、或者各种高级扩展等等。

我们把React Native设计为可以在其基础上编写真正的原生代码，并且可以访问平台所有的能力。这是一个相对高级的特性，我们并不认为它应当在日常开发的过程中经常出现，但具备这样的能力是很重要的。如果React Native还不支持某个你需要的原生特性，你应当可以自己实现该特性的封装。

## Toast模块

本向导会用[Toast](http://developer.android.com/reference/android/widget/Toast.html)作为例子。假设我们希望可以从Javascript发起一个Toast消息（Android中的一种会在屏幕下方弹出、保持一段时间的消息通知）

我们首先来创建一个原生模块。一个原生模块是一个继承了`ReactContextBaseJavaModule`的Java类，它可以实现一些JavaScript所需的功能。我们这里的目标是可以在JavaScript里写`ToastAndroid.show('Awesome', ToastAndroid.SHORT);`，来调起一个Toast通知。

```java
package com.facebook.react.modules.toast;

import android.widget.Toast;

import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;

import java.util.Map;

public class ToastModule extends ReactContextBaseJavaModule {

  private static final String DURATION_SHORT_KEY = "SHORT";
  private static final String DURATION_LONG_KEY = "LONG";

  public ToastModule(ReactApplicationContext reactContext) {
    super(reactContext);
  }
}
```

`ReactContextBaseJavaModule`要求派生类实现`getName`方法。这个函数用于返回一个字符串名字，这个名字在JavaScript端标记这个模块。这里我们把这个模块叫做`ToastAndroid`，这样就可以在JavaScript中通过`React.NativeModules.ToastAndroid`访问到这个模块。

```java
  @Override
  public String getName() {
    return "ToastAndroid";
  }
```

_译注_：模块名前的RCT前缀会被自动移除。所以如果返回的字符串为"RCTToastAndroid"，在JavaScript端依然通过`React.NativeModules.ToastAndroid`访问到这个模块。

一个可选的方法`getContants`返回了需要导出给JavaScript使用的常量。它并不一定需要实现，但在定义一些可以被JavaScript同步访问到的预定义的值时非常有用。

```java
  @Override
  public Map<String, Object> getConstants() {
    final Map<String, Object> constants = new HashMap<>();
    constants.put(DURATION_SHORT_KEY, Toast.LENGTH_SHORT);
    constants.put(DURATION_LONG_KEY, Toast.LENGTH_LONG);
    return constants;
  }
```

要导出一个方法给JavaScript使用，Java方法需要使用注解`@ReactMethod`。方法的返回类型必须为`void`。React Native的跨语言访问是异步进行的，所以想要给JavaScript返回一个值的唯一办法是使用回调函数或者发送事件（参见下文的描述）。

```java
  @ReactMethod
  public void show(String message, int duration) {
    Toast.makeText(getReactApplicationContext(), message, duration).show();
  }
```

### 参数类型

下面的参数类型在`@ReactMethod`注明的方法中，会被直接映射到它们对应的JavaScript类型。

```
Boolean -> Bool
Integer -> Number
Double -> Number
Float -> Number
String -> String
Callback -> function
ReadableMap -> Object
ReadableArray -> Array
```
参阅[ReadableMap](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/ReadableMap.java)和[ReadableArray](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/ReadableArray.java)。

### 注册模块

在Java这边要做的最后一件事就是注册这个模块。我们需要在应用的Package类的`createNativeModules`方法中添加这个模块。如果模块没有被注册，它也无法在JavaScript中被访问到。

```java
class AnExampleReactPackage implements ReactPackage {

  @Override
  public List<Class<? extends JavaScriptModule>> createJSModules() {
    return Collections.emptyList();
  }

  @Override
  public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.emptyList();
  }

  @Override
  public List<NativeModule> createNativeModules(
                              ReactApplicationContext reactContext) {
    List<NativeModule> modules = new ArrayList<>();

    modules.add(new ToastModule(reactContext));

    return modules;
  }
```

这个package需要在`MainApplication.java`文件的`getPackages`方法中提供。这个文件位于你的react-native应用文件夹的android目录中。具体路径是: `android/app/src/main/java/com/your-app-name/MainApplication.java`.

```java
protected List<ReactPackage> getPackages() {
    return Arrays.<ReactPackage>asList(
            new MainReactPackage(),
            new AnExampleReactPackage()); // <-- 添加这一行，类名替换成你的Package类的名字.
}
```

为了让你的功能从JavaScript端访问起来更为方便，通常我们都会把原生模块封装成一个JavaScript模块。这不是必须的，但省下了每次都从`NativeModules`中获取对应模块的步骤。这个JS文件也可以用于添加一些其他JavaScript端实现的功能。

```javascript
'use strict';

/**
 * This exposes the native ToastAndroid module as a JS module. This has a function 'show'
 * which takes the following parameters:
 *
 * 1. String message: A string with the text to toast
 * 2. int duration: The duration of the toast. May be ToastAndroid.SHORT or ToastAndroid.LONG
 */
var { NativeModules } = require('react-native');
module.exports = NativeModules.ToastAndroid;
```

现在，在别处的JavaScript代码中可以这样调用你的方法：

```javascript
var ToastAndroid = require('./ToastAndroid')
ToastAndroid.show('Awesome', ToastAndroid.SHORT);
```

## 更多特性

### 回调函数

原生模块还支持一种特殊的参数——回调函数。它提供了一个函数来把返回值传回给JavaScript。

```java
public class UIManagerModule extends ReactContextBaseJavaModule {

...

  @ReactMethod
  public void measureLayout(
      int tag,
      int ancestorTag,
      Callback errorCallback,
      Callback successCallback) {
    try {
      measureLayout(tag, ancestorTag, mMeasureBuffer);
      float relativeX = PixelUtil.toDIPFromPixel(mMeasureBuffer[0]);
      float relativeY = PixelUtil.toDIPFromPixel(mMeasureBuffer[1]);
      float width = PixelUtil.toDIPFromPixel(mMeasureBuffer[2]);
      float height = PixelUtil.toDIPFromPixel(mMeasureBuffer[3]);
      successCallback.invoke(relativeX, relativeY, width, height);
    } catch (IllegalViewOperationException e) {
      errorCallback.invoke(e.getMessage());
    }
  }

...
```

这个函数可以在JavaScript里这样使用：

```js
UIManager.measureLayout(
  100,
  100,
  (msg) => {
    console.log(msg);
  },
  (x, y, width, height) => {
    console.log(x + ':' + y + ':' + width + ':' + height);
  }
);
```

原生模块通常只应调用回调函数一次。但是，它可以保存callback并在将来调用。

请务必注意callback并非在对应的原生函数返回后立即被执行——注意跨语言通讯是异步的，这个执行过程会通过消息循环来进行。

## Promises

__译注__：这一部分涉及到较新的js语法和特性，不熟悉的读者建议先阅读ES6的相关书籍和文档。

原生模块还可以使用promise来简化代码，搭配ES2016(ES7)标准的`async/await`语法则效果更佳。如果桥接原生方法的最后一个参数是一个`Promise`，则对应的JS方法就会返回一个Promise对象。

我们把上面的代码用promise来代替回调进行重构：

```java
public class UIManagerModule extends ReactContextBaseJavaModule {

...

  @ReactMethod
  public void measureLayout(
      int tag,
      int ancestorTag,
      Promise promise) {
    try {
      measureLayout(tag, ancestorTag, mMeasureBuffer);

      WritableMap map = Arguments.createMap();

      map.putDouble("relativeX", PixelUtil.toDIPFromPixel(mMeasureBuffer[0]));
      map.putDouble("relativeY", PixelUtil.toDIPFromPixel(mMeasureBuffer[1]));
      map.putDouble("width", PixelUtil.toDIPFromPixel(mMeasureBuffer[2]));
      map.putDouble("height", PixelUtil.toDIPFromPixel(mMeasureBuffer[3]));

      promise.resolve(map);
    } catch (IllegalViewOperationException e) {
      promise.reject(e.getMessage());
    }
  }

...
```

现在JavaScript端的方法会返回一个Promise。这样你就可以在一个声明了`async`的异步函数内使用`await`关键字来调用，并等待其结果返回。（虽然这样写着看起来像同步操作，但实际仍然是异步的，并不会阻塞执行来等待）。

```js
async function measureLayout() {
  try {
    var {
      relativeX,
      relativeY,
      width,
      height,
    } = await UIManager.measureLayout(100, 100);

    console.log(relativeX + ':' + relativeY + ':' + width + ':' + height);
  } catch (e) {
    console.error(e);
  }
}

measureLayout();
```

### 多线程

原生模块不应对自己被调用时所处的线程做任何假设，当前的状况有可能会在将来的版本中改变。如果一个过程要阻塞执行一段时间，这个工作应当分配到一个内部管理的工作线程，然后从那边可以调用任意的回调函数。_译注_：我们通常用AsyncTask来完成这项工作。

### 发送事件到JavaScript

原生模块可以在没有被调用的情况下往JavaScript发送事件通知。最简单的办法就是通过`RCTDeviceEventEmitter`，这可以通过`ReactContext`来获得对应的引用，像这样：

```java
...
private void sendEvent(ReactContext reactContext,
                       String eventName,
                       @Nullable WritableMap params) {
  reactContext
      .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
      .emit(eventName, params);
}
...
WritableMap params = Arguments.createMap();
...
sendEvent(reactContext, "keyboardWillShow", params);
```

JavaScript模块可以通过`Subscribable`mixin的`addListenerOn`方法来接受事件。

```js
var { DeviceEventEmitter } = require('react-native');
...

var ScrollResponderMixin = {
  mixins: [Subscribable.Mixin],


  componentWillMount: function() {
    ...
    this.addListenerOn(DeviceEventEmitter,
                       'keyboardWillShow',
                       this.scrollResponderKeyboardWillShow);
    ...
  },
  scrollResponderKeyboardWillShow:function(e: Event) {
    this.keyboardWillOpenTo = e;
    this.props.onKeyboardWillShow && this.props.onKeyboardWillShow(e);
  },
```

你还可以直接使用`DeviceEventEmitter`模块来监听事件：

```js
...
componentWillMount: function() {
  DeviceEventEmitter.addListener('keyboardWillShow', function(e: Event) {
    // handle event.
  });
}
...
```

