layout: react-native
title: android react-native 开发小结（基于react-native 0.39）
date: 2017-01-07 11:17:42
comments: true
toc: true

categories:
- 安卓

tags: 
- react-native
- android
---

## 1. 开发环境搭建

英文文档：https://facebook.github.io/react-native/docs/getting-started.html

中文文档：http://reactnative.cn/docs/0.39/getting-started.html

### 简要说明，省略android环境搭建过程：

#### mac：

Homebrew（包管理）——>node（js开发，包含npm命令）,watchman（文件变动检测），flow（[Flow](http://www.flowtype.org)是一个静态的JS类型检查工具）——>react-native-cli（react-native环境，使用npm命令安装）

#### windows：

Chocolatey（包管理）——>Python 2，node（js开发，包含npm命令）——>react-native-cli（react-native，使用npm命令安装）

#### 可选安装：

淘宝镜像，可加速node module下载，否则国内需要翻墙

```
npm config set registry https://registry.npm.taobao.org --global
npm config set disturl https://npm.taobao.org/dist --global
```

#### 生成react-native应用：

> react-native init AwesomeProject（初始化项目，项目名称AwesomeProject）
> cd AwesomeProject（进入项目目录）
> react-native start（开启node服务）
> react-native run-android（打包生成apk并安装）



## 2. 已有的原生应用，如何结合react-native

英文文档：https://facebook.github.io/react-native/docs/integration-with-existing-apps.html

中文文档：http://reactnative.cn/docs/0.39/integration-with-existing-apps.html#content

### 项目结构调整：

参照AwesomeProject目录结构，原有的代码需放在**android**目录下：

| **AwesomeProject**（名称可以根据自己需要） | **android**（名称不要修改）  | **app** |
| ------------------------------ | -------------------- | ------- |
|                                | **index.android.js** |         |
|                                | ios                  |         |
|                                | index.ios.js         |         |
|                                | node_modules         |         |
|                                | **package.json**     |         |

**package.json**用来记录项目中使用到的node_modules的依赖，在AwesomeProject（项目根目录）目录下执行***npm install***可以下载生成node_modules目录。也就是说node_modules是通过执行命令来生成的，不是项目本身的代码。
**index.android.js**用来记录react-native模块的加载入口，可以注册多个，原生程序加载模块时，对应模块相应的名称即可。

```javascript
AppRegistry.registerComponent('HomePage', () => AppointContainer);//首页
AppRegistry.registerComponent('NewsList', () => Root); //资讯主页面
AppRegistry.registerComponent('subInfo', () => subInfoRoot);//资讯的子页面
```

其余步骤可参照文档来。

### 命令解释：

> $ npm init（生成package.json文件，填写相关信息）
> $ npm install --save react react-native（下载并在项目中引用react-native模块，--save表示记录写入package.json）
> $ curl -o .flowconfig https://raw.githubusercontent.com/facebook/react-native/master/.flowconfig （添加flow配置，[Flow](http://www.flowtype.org)是一个静态的JS类型检查工具）

### 建议操作方式：

先使用react-native init AwesomeProject初始化一个项目，删除其中的android目录下源码，拷贝原有原生项目代码自android目录下，修改AwesomeProject名称（可选），由于已经有了初始化的**package.json**，无需再进行npm init操作，直接根目录下执行npm install。然后参照文档配置原生项目，原生项目build.gradle加入引用

```groovy
dependencies {
     ...
     compile "com.facebook.react:react-native:+" // From node_modules.
 }
```

后续等操作参照链接文档。

### Application示例：

```java
public class MyApplication extends Application implements ReactApplication {

  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
    @Override public boolean getUseDeveloperSupport() {
      return BuildConfig.DEBUG;
    }

    // 2. Override the getJSBundleFile method in order to let
    // the CodePush runtime determine where to get the JS
    // bundle location from on each app start
    @Override protected String getJSBundleFile() {
      Log.d("code-push", CodePush.getJSBundleFile());
      return CodePush.getJSBundleFile();
    }

    @Override public List<ReactPackage> getPackages() {
      // 3. Instantiate an instance of the CodePush runtime and add it to the list of
      // existing packages, specifying the right deployment key. If you don't already
      // have it, you can run "code-push deployment ls <appName> -k" to retrieve your key.

      Log.d("code-push", BuildConfig.CODEPUSH_KEY);

      CodePush codePush = new CodePush(BuildConfig.CODEPUSH_KEY, MyApplication.this,
          BuildConfig.DEBUG);
      String appversion = codePush.getAppVersion();
      String serverurl = codePush.getServerUrl();

      Log.d("code-push", "appversion->" + appversion);
      Log.d("code-push", "serverurl->" + serverurl);

      return Arrays.asList(new MainReactPackage(), new ZMCNativeBridgeReactPackage(), codePush);
    }
  };

  @Override public ReactNativeHost getReactNativeHost() {
    return mReactNativeHost;
  }
}
```

### BaseReactActivity示例：

```java
public abstract class BaseReactActivity extends AppCompatActivity
    implements DefaultHardwareBackBtnHandler {
  private ReactRootView mReactRootView;
  private ReactInstanceManager mReactInstanceManager;

  // This method returns the name of our top-level component to show
  public abstract String getMainComponentName();

  public abstract Bundle getMainComponentParams();

  public abstract void parseActivityParams();

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    //用来解析bundle等参数
    parseActivityParams();

    mReactRootView = new ReactRootView(this);
    mReactInstanceManager = ((MyApplication) getApplication()).
        getReactNativeHost().getReactInstanceManager();

    mReactRootView.startReactApplication(mReactInstanceManager, getMainComponentName(),
        getMainComponentParams());

    setContentView(mReactRootView);
  }

  @Override public void invokeDefaultOnBackPressed() {
    super.onBackPressed();
  }

  @Override protected void onPause() {
    super.onPause();
    if (mReactInstanceManager != null) {
      mReactInstanceManager.onHostPause(this);
    }
  }

  @Override protected void onResume() {
    super.onResume();
    if (mReactInstanceManager != null) {
      mReactInstanceManager.onHostResume(this, this);
    }
  }

  @Override protected void onDestroy() {
    super.onDestroy();

    if (mReactRootView != null) {
      //这里需要unmount，否则会内存泄露
      mReactRootView.unmountReactApplication();
      mReactRootView = null;
    }

    if (mReactInstanceManager != null) {
      mReactInstanceManager.onHostDestroy(this);
      mReactInstanceManager = null;
    }
  }

  @Override public void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (mReactInstanceManager != null) {
      mReactInstanceManager.onActivityResult(this, requestCode, resultCode, data);
    }
  }

  @Override public void onBackPressed() {
    if (mReactInstanceManager != null) {
      mReactInstanceManager.onBackPressed();
    } else {
      super.onBackPressed();
    }
  }
}
```

### ReactViewContainActivity示例：

该Activity可以包含任意React native的Component。

“target”中传打开React native的Component名称，String类型；“params”中传打开React native的Component所需要的参数，Bundle类型。

```java
public class ReactViewContainActivity extends BaseReactActivity {

  private String targetComponentName;
  private Bundle targetParams;

  @Override public String getMainComponentName() {
    return targetComponentName;
  }

  @Override public Bundle getMainComponentParams() {
    return targetParams;
  }

  @Override public void parseActivityParams() {
    Bundle bundle = getIntent().getExtras();
    targetComponentName = bundle.getString("target");
    targetParams = bundle.getBundle("params");
  }

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
  }
}
```

### BaseReactFragment示例:

```java
public abstract class BaseReactFragment extends Fragment {
  private ReactRootView mReactRootView;
  private ReactInstanceManager mReactInstanceManager;

  // This method returns the name of our top-level component to show
  public abstract String getMainComponentName();

  public abstract Bundle getMainComponentParams();

  @Override public void onAttach(Context context) {
    super.onAttach(context);
    mReactRootView = new ReactRootView(context);
    mReactInstanceManager = ((MyApplication) getActivity().getApplication()).
        getReactNativeHost().getReactInstanceManager();
  }

  @Nullable @Override
  public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
      @Nullable Bundle savedInstanceState) {
    return mReactRootView;
  }

  @Override public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    mReactRootView.startReactApplication(mReactInstanceManager, getMainComponentName(),
        getMainComponentParams());
  }

  @Override public void onDestroy() {
    super.onDestroy();

    if (mReactRootView != null) {
      //这里需要unmount，否则会内存泄露
      mReactRootView.unmountReactApplication();
      mReactRootView = null;
    }

    if (mReactInstanceManager != null) {
      mReactInstanceManager = null;
    }
  }

  protected ReactRootView getmReactRootView() {
    return mReactRootView;
  }
}
```

### Fragment用法：

```java
public class NewsReactFragment extends BaseReactFragment {

  @Override public String getMainComponentName() {
    return "NewsList";
  }

  @Override public Bundle getMainComponentParams() {
    return null;
  }
}
```



## 3. react-native、原生模块相互调用

### 根据具体情况分为以下两类模块：

1. react-native与原生互相调用，无view交互模块，继承**ReactContextBaseJavaModule**，添加交互方法。

   可参照：http://www.jianshu.com/p/07b928feee3b

2. 原生提供view模块供react-native于js代码上调用（包含view属性设置），则继承**SimpleViewManager**，实现相应逻辑。

   可参照：http://www.jianshu.com/p/31c4306a55ff

两者均需要实现**ReactPackage**接口，用来在初始化react组件的时候注册。

### js和原生数据类型转换工具类：

1. > com.facebook.react.bridge.Arguments

  ```java
     public class Arguments {

       /**
        * This method should be used when you need to stub out creating NativeArrays in unit tests.
        */
       public static WritableArray createArray() {
         return new WritableNativeArray();
       }

       /**
        * This method should be used when you need to stub out creating NativeMaps in unit tests.
        */
       public static WritableMap createMap() {
         return new WritableNativeMap();
       }

       public static WritableNativeArray fromJavaArgs(Object[] args) {
         WritableNativeArray arguments = new WritableNativeArray();
         for (int i = 0; i < args.length; i++) {
           Object argument = args[i];
           if (argument == null) {
             arguments.pushNull();
             continue;
           }

           Class argumentClass = argument.getClass();
           if (argumentClass == Boolean.class) {
             arguments.pushBoolean(((Boolean) argument).booleanValue());
           } else if (argumentClass == Integer.class) {
             arguments.pushDouble(((Integer) argument).doubleValue());
           } else if (argumentClass == Double.class) {
             arguments.pushDouble(((Double) argument).doubleValue());
           } else if (argumentClass == Float.class) {
             arguments.pushDouble(((Float) argument).doubleValue());
           } else if (argumentClass == String.class) {
             arguments.pushString(argument.toString());
           } else if (argumentClass == WritableNativeMap.class) {
             arguments.pushMap((WritableNativeMap) argument);
           } else if (argumentClass == WritableNativeArray.class) {
             arguments.pushArray((WritableNativeArray) argument);
           } else {
             throw new RuntimeException("Cannot convert argument of type " + argumentClass);
           }
         }
         return arguments;
       }

       /**
        * Convert an array to a {@link WritableArray}.
        *
        * @param array the array to convert. Supported types are: {@code String[]}, {@code Bundle[]},
        * {@code int[]}, {@code float[]}, {@code double[]}, {@code boolean[]}.
        *
        * @return the converted {@link WritableArray}
        * @throws IllegalArgumentException if the passed object is none of the above types
        */
       public static WritableArray fromArray(Object array) {
         WritableArray catalystArray = createArray();
         if (array instanceof String[]) {
           for (String v: (String[]) array) {
             catalystArray.pushString(v);
           }
         } else if (array instanceof Bundle[]) {
           for (Bundle v: (Bundle[]) array) {
             catalystArray.pushMap(fromBundle(v));
           }
         } else if (array instanceof int[]) {
           for (int v: (int[]) array) {
             catalystArray.pushInt(v);
           }
         } else if (array instanceof float[]) {
           for (float v: (float[]) array) {
             catalystArray.pushDouble(v);
           }
         } else if (array instanceof double[]) {
           for (double v: (double[]) array) {
             catalystArray.pushDouble(v);
           }
         } else if (array instanceof boolean[]) {
           for (boolean v: (boolean[]) array) {
             catalystArray.pushBoolean(v);
           }
         } else {
           throw new IllegalArgumentException("Unknown array type " + array.getClass());
         }
         return catalystArray;
       }

       /**
        * Convert a {@link Bundle} to a {@link WritableMap}. Supported key types in the bundle
        * are:
        *
        * <ul>
        *   <li>primitive types: int, float, double, boolean</li>
        *   <li>arrays supported by {@link #fromArray(Object)}</li>
        *   <li>{@link Bundle} objects that are recursively converted to maps</li>
        * </ul>
        *
        * @param bundle the {@link Bundle} to convert
        * @return the converted {@link WritableMap}
        * @throws IllegalArgumentException if there are keys of unsupported types
        */
       public static WritableMap fromBundle(Bundle bundle) {
         WritableMap map = createMap();
         for (String key: bundle.keySet()) {
           Object value = bundle.get(key);
           if (value == null) {
             map.putNull(key);
           } else if (value.getClass().isArray()) {
             map.putArray(key, fromArray(value));
           } else if (value instanceof String) {
             map.putString(key, (String) value);
           } else if (value instanceof Number) {
             if (value instanceof Integer) {
               map.putInt(key, (Integer) value);
             } else {
               map.putDouble(key, ((Number) value).doubleValue());
             }
           } else if (value instanceof Boolean) {
             map.putBoolean(key, (Boolean) value);
           } else if (value instanceof Bundle) {
             map.putMap(key, fromBundle((Bundle) value));
           } else {
             throw new IllegalArgumentException("Could not convert " + value.getClass());
           }
         }
         return map;
       }

       /**
        * Convert a {@link WritableMap} to a {@link Bundle}.
        * @param readableMap the {@link WritableMap} to convert.
        * @return the converted {@link Bundle}.
        */
       @Nullable
       public static Bundle toBundle(@Nullable ReadableMap readableMap) {
         if (readableMap == null) {
           return null;
         }

         ReadableMapKeySetIterator iterator = readableMap.keySetIterator();

         Bundle bundle = new Bundle();
         while (iterator.hasNextKey()) {
           String key = iterator.nextKey();
           ReadableType readableType = readableMap.getType(key);
           switch (readableType) {
             case Null:
               bundle.putString(key, null);
               break;
             case Boolean:
               bundle.putBoolean(key, readableMap.getBoolean(key));
               break;
             case Number:
               // Can be int or double.
               bundle.putDouble(key, readableMap.getDouble(key));
               break;
             case String:
               bundle.putString(key, readableMap.getString(key));
               break;
             case Map:
               bundle.putBundle(key, toBundle(readableMap.getMap(key)));
               break;
             case Array:
               // TODO t8873322
               throw new UnsupportedOperationException("Arrays aren't supported yet.");
             default:
               throw new IllegalArgumentException("Could not convert object with key: " + key + ".");
           }
         }

         return bundle;
       }
     }
  ```


2. 自定义ConversionUtil

  ```java
  /* package */ final class ConversionUtil {
    /**
     * toObject extracts a value from a {@link ReadableMap} by its key,
     * and returns a POJO representing that object.
     *
     * @param readableMap The Map to containing the value to be converted
     * @param key The key for the value to be converted
     * @return The converted POJO
     */
    public static Object toObject(@Nullable ReadableMap readableMap, String key) {
      if (readableMap == null) {
        return null;
      }

      Object result;

      ReadableType readableType = readableMap.getType(key);
      switch (readableType) {
        case Null:
          result = key;
          break;
        case Boolean:
          result = readableMap.getBoolean(key);
          break;
        case Number:
          // Can be int or double.
          double tmp = readableMap.getDouble(key);
          if (tmp == (int) tmp) {
            result = (int) tmp;
          } else {
            result = tmp;
          }
          break;
        case String:
          result = readableMap.getString(key);
          break;
        case Map:
          result = toMap(readableMap.getMap(key));
          break;
        case Array:
          result = toList(readableMap.getArray(key));
          break;
        default:
          throw new IllegalArgumentException("Could not convert object with key: " + key + ".");
      }

      return result;
    }

    /**
     * toMap converts a {@link ReadableMap} into a HashMap.
     *
     * @param readableMap The ReadableMap to be conveted.
     * @return A HashMap containing the data that was in the ReadableMap.
     */
    public static Map<String, Object> toMap(@Nullable ReadableMap readableMap) {
      if (readableMap == null) {
        return null;
      }

      ReadableMapKeySetIterator iterator = readableMap.keySetIterator();
      if (!iterator.hasNextKey()) {
        return null;
      }

      Map<String, Object> result = new HashMap<>();
      while (iterator.hasNextKey()) {
        String key = iterator.nextKey();
        result.put(key, toObject(readableMap, key));
      }

      return result;
    }

    /**
     * toList converts a {@link ReadableArray} into an ArrayList.
     *
     * @param readableArray The ReadableArray to be conveted.
     * @return An ArrayList containing the data that was in the ReadableArray.
     */
    public static List<Object> toList(@Nullable ReadableArray readableArray) {
      if (readableArray == null) {
        return null;
      }

      List<Object> result = new ArrayList<>(readableArray.size());
      for (int index = 0; index < readableArray.size(); index++) {
        ReadableType readableType = readableArray.getType(index);
        switch (readableType) {
          case Null:
            result.add(String.valueOf(index));
            break;
          case Boolean:
            result.add(readableArray.getBoolean(index));
            break;
          case Number:
            // Can be int or double.
            double tmp = readableArray.getDouble(index);
            if (tmp == (int) tmp) {
              result.add((int) tmp);
            } else {
              result.add(tmp);
            }
            break;
          case String:
            result.add(readableArray.getString(index));
            break;
          case Map:
            result.add(toMap(readableArray.getMap(index)));
            break;
          case Array:
            result = toList(readableArray.getArray(index));
            break;
          default:
            throw new IllegalArgumentException("Could not convert object with index: " + index + ".");
        }
      }

      return result;
    }
  }
  ```



## 4. 如何发布react-native开发的相关js代码

使用codepush：https://www.codeproject.com/

官方文档：https://microsoft.github.io/code-push/docs/getting-started.html

参考文章：http://www.jianshu.com/p/9e3b4a133bcc

1. 安装CodePush CLI：

  > npm install -g code-push-cli

2. 注册CodePush账号：

  > code-push register

3. 登录CodePush账号（注册完成会自动登录，无需此步骤）：

  > code-push login

4. 创建app关联账户：

  > code-push app add <appName>

5. 项目集成codepush SDK（见官方文档）

   两种方式：

   1. [**RNPM**](https://microsoft.github.io/code-push/docs/react-native.html#plugin-installation-android---rnpm)
   2. 手动集成

   建议使用手动集成，rnpm的作用是自动添加相关的内容至项目中，但是如果项目名称等和模板不一样，会添加失败，到头来还是要手动修改。

   注意事项：项目版本号必须为三位，例如 1.0.0，其他格式会出错。

6. 发布app bundle：

  > code-push release-react <appName> <platform>



## 5. jenkins自动打包的坑

release打包取消勾选该选项：![](https://ww4.sinaimg.cn/large/006tNbRwgw1fb0v46hiffj31f00ymq4n.jpg)

否则会中断打包进程，报类似以下错误：

> ```
> 19:08:12 :app:bundleZmlearnReleaseJsAndAssets
> 19:08:12 FAILURE: Build failed with an exception.
> 19:08:12 
> 19:08:12 * What went wrong:
> 19:08:12 Failed to capture snapshot of input files for task 'bundleZmlearnReleaseJsAndAssets' during up-to-date check.
> 19:08:12 > Failed to create MD5 hash for file E:\jenkins\workspace\zmlearn_android_am_release\caches\2.14.1\classAnalysis\cache.properties.lock.
> 19:08:12 
> 19:08:12 * Try:
> 19:08:12 Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.
> 19:08:12 
> 19:08:12 
> 19:08:12 BUILD FAILED
> 19:08:12 
> 19:08:12 Total time: 4 mins 39.797 secs
> 19:08:12 Build step 'Invoke Gradle script' changed build result to FAILURE
> 19:08:12 Build step 'Invoke Gradle script' marked build as failure
> ```

原因可参见：https://discuss.gradle.org/t/build-failure-with-failed-to-capture-snapshot-of-input-files-for-task-war-during-up-to-date-check/9132

