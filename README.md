# ReactNative Android通信原理

本文基于release 0.29,从源码角度剖析react native for android的java<=>js通信机制。

## 整体结构

rn分为三大模块,java层、js层和bridge,java和js通过bridge实现通信,由于java不能直接调用JavaScriptCore,bridge层由C/C++实现,整体结构图如下。<br>

![](pic/1.png)<br>

## 基本原理

结构图如下<br>

![](pic/2.png)<br>

先来看rn初始化的过程, java => js<br>

### 初始化bridge

![](pic/3.jpeg)<br>

#### 1. ReactRootView.startReactApplication()
```java
  public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle launchOptions) {
    ......
    if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
      mReactInstanceManager.createReactContextInBackground();
    }
    ......
  }
```

#### 2. ReactInstanceManagerImpl.createReactContextInBackground()
```java
  public void createReactContextInBackground() {
    ......
    mHasStartedCreatingInitialContext = true;
    recreateReactContextInBackgroundInner();
  }
```
经过几次派发,在AsyncTask中创建react context.

#### 3. ReactInstanceManagerImpl.ReactContextInitAsyncTask.doInBackground()
```java
    protected Result<ReactApplicationContext> doInBackground(ReactContextInitParams... params) {
      ......
        JavaScriptExecutor jsExecutor =
            params[0].getJsExecutorFactory().create(
              mJSCConfig == null ? new WritableNativeMap() : mJSCConfig.getConfigMap());
        return Result.of(createReactContext(jsExecutor, params[0].getJsBundleLoader()));
      ......
    }
```

#### 4. ReactInstanceManagerImpl.createReactContext()
```java
  private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor,
      JSBundleLoader jsBundleLoader) {
    NativeModuleRegistry.Builder nativeRegistryBuilder = new NativeModuleRegistry.Builder();
    JavaScriptModuleRegistry.Builder jsModulesBuilder = new JavaScriptModuleRegistry.Builder();

    final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);
    try {
      CoreModulesPackage coreModulesPackage =
          new CoreModulesPackage(this, mBackBtnHandler, mUIImplementationProvider);
      processPackage(coreModulesPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
    } finally {
      Systrace.endSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE);
    }

    for (ReactPackage reactPackage : mPackages) {
      try {
        processPackage(reactPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
      } finally {
        Systrace.endSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE);
      }
    }

    NativeModuleRegistry nativeModuleRegistry;
    try {
       nativeModuleRegistry = nativeRegistryBuilder.build();
    } finally {
    }

    CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
        .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
        .setJSExecutor(jsExecutor)
        .setRegistry(nativeModuleRegistry)
        .setJSModuleRegistry(jsModulesBuilder.build())
        .setJSBundleLoader(jsBundleLoader)
        .setNativeModuleCallExceptionHandler(exceptionHandler);

    final CatalystInstance catalystInstance;
    try {
      catalystInstance = catalystInstanceBuilder.build();
    } finally {
    }

    try {
      catalystInstance.getReactQueueConfiguration().getJSQueueThread().callOnQueue(
        new Callable<Void>() {
          @Override
          public Void call() {
            try {
              catalystInstance.runJSBundle();
            } finally {
            }
            return null;
          }
        }).get();
    } catch (InterruptedException | ExecutionException e) {
      throw new RuntimeException(e);
    }

    return reactContext;
  }
```
