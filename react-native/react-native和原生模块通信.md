### react-native和原生模块通信

* 创建一个原生模块，原生模块是一个继承了`ReactContextBaseJavaModule`的类，可以实现一些js所需的功能

  > 以下方法名ToastAndroid在官方api中已存在，所以实际操作中需要改名

  * 必须实现方法String getName()

    ```java
    @Override
    public String getName() {
      return "ToastAndroid"; //这个名字即在js中使用的模块名
    }
    ```

  * 选择实现 `Map<String, Object> getConstants()`导出js使用的常量

    ```java
    @Override
      public Map<String, Object> getConstants() {
        final Map<String, Object> constants = new HashMap<>();
        constants.put(DURATION_SHORT_KEY, Toast.LENGTH_SHORT);
        constants.put(DURATION_LONG_KEY, Toast.LENGTH_LONG);
        return constants;
    }
    ```

  * 导出一个方法给js调用，编码时需用注解`@ReactMethod`，返回类型为`void`，React Native的跨语言访问是异步进行的，所以想要给JavaScript返回一个值的唯一办法是使用回调函数或者发送事件

    ```java
    @ReactMethod
    public void show(String message, int duration) {
        Toast.makeText(getReactApplicationContext(),message,duration).show();
    }
    ```

  * 下面的参数类型在`@ReactMethod`注明的方法中，会被直接映射到它们对应的JavaScript类型

    ```java
    Boolean -> Bool
    Integer -> Number
    Double -> Number
    Float -> Number
    String -> String
    Callback -> function
    ReadableMap -> Object
    ReadableArray -> Array
    ```

* 注册模块，在应用的Package类的`createNativeModules`方法中添加这个模块

  * 创建一个类（一个package，最后会在MainApplication类中提供）继承自ReactPackage

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

  * 在`MainApplication.java` 的`getPackage` 方法中提供上述包

    ```java
    protected List<ReactPackage> getPackages() {
        return Arrays.<ReactPackage>asList(
                new MainReactPackage(),
                new AnExampleReactPackage()); // <-- 添加这一行，类名替换成你的Package类的名字.
    }
    ```

* 为了使这个功能在js端访问起来更加方便，通常将这个原生模块封装成一个js模块

  ```javascript
  'use strict'

  import { NativeModules } from 'react-native';

  //ToastAndroid对应上述getName()中返回的字符串
  export default NativeModules.ToastAndroid;
  ```

* 调用原生模块

  ```javascript
  import ToastAndroid from './ToastAndroid';
  ToastAndroid.show('awesome', ToastAndroid.SHORT);
  ```

  ​