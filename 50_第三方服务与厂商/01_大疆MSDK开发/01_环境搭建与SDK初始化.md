---
type: 知识点
status: 已整理
tags:
  - 类型/知识点
  - 生态/移动
  - 生态/大疆MSDK
created: 2026-10-21
updated: 2026-10-21
source: SampleCode-V5 / DJISDKManager / ISDKManager_SDKManagerCallback
---

# 环境搭建与 SDK 初始化

## 一、功能说明

把 MSDK 接进一个 Android 工程，跑通「初始化 → 注册 → 产品连接」三步。注册成功后，Key 才能取到真实数据，所有飞行功能才可用。

## 二、相关 API

| 类 | 方法 | 说明 |
|----|------|------|
| `dji.v5.manager.SDKManager` | `getInstance()` | SDK 单例入口（文档中接口名为 `ISDKManager`） |
| | `init(Context, SDKManagerCallback)` | 初始化，**必须先调**，回调里返回初始化进度 |
| | `registerApp()` | 联网向 DJI 服务器校验 App Key，须在 `INITIALIZE_COMPLETE` 后调用 |
| | `isRegistered()` | 是否已注册成功 |
| | `getSDKVersion()` | SDK 版本号 |
| | `getProductCategory()` | 当前 SDK 包类型（`AIRCRAFT` / `HANDHELD` / `ALL`） |
| | `destroy()` | 释放 SDK（示例在退出时调用，文档未列出） |
| `dji.v5.manager.interfaces.SDKManagerCallback` | `onInitProcess(DJISDKInitEvent, int)` | 初始化进度；`INITIALIZE_COMPLETE` 时去注册 |
| | `onRegisterSuccess()` | 注册成功 |
| | `onRegisterFailure(IDJIError)` | 注册失败（多为网络/Key/签名不匹配） |
| | `onProductConnect(int productId)` | 硬件连接成功 |
| | `onProductDisconnect(int productId)` | 硬件断开 |
| | `onProductChanged(int productId)` | 产品变更 |
| | `onDatabaseDownloadProgress(long, long)` | 限飞数据库下载进度 |
| `dji.v5.network.DJINetworkManager` | `addNetworkStatusListener(listener)` | 网络恢复后补一次 `registerApp()` |
| `dji.v5.manager.ldm.LDMManager` | `enableLDM(Context, callback, LDMExemptModule...)` | 本地数据模式（需向 DJI 申请 License） |

`DJISDKInitEvent` 只有两个值：`START_TO_INITIALIZE`、`INITIALIZE_COMPLETE`。

## 三、最简实现

### 1. Gradle 依赖

```groovy
// 仓库（示例工程配置）
repositories {
    google()
    mavenCentral()
}

dependencies {
    implementation 'com.dji:dji-sdk-v5-aircraft:5.18.0'          // 运行时实现
    compileOnly    'com.dji:dji-sdk-v5-aircraft-provided:5.18.0' // 编译期 API
    implementation 'com.dji:dji-sdk-v5-networkImp:5.18.0'        // 网络能力（注册/RTK/直播）
}

android {
    defaultConfig {
        minSdk 24
        ndk { abiFilters 'arm64-v8a' }   // 官方只提供 arm64 的 so
    }
    packagingOptions {
        pickFirst 'lib/arm64-v8a/libc++_shared.so'   // 与其他 native 库冲突时必须
    }
}
```

### 2. AndroidManifest.xml

```xml
<application
    android:name=".MyApplication"
    android:extractNativeLibs="true">

    <!-- App Key：值来自 DJI 开发者后台，必须与包名 + 签名一致 -->
    <meta-data
        android:name="com.dji.sdk.API_KEY"
        android:value="你的AppKey" />
</application>
```

> 权限不用手写太多：`INTERNET`、`ACCESS_NETWORK_STATE`、`ACCESS_WIFI_STATE`、`BLUETOOTH`、定位、存储等已由 SDK 的 AAR 清单自动合并。应用侧只需补自己用到的（如 `RECORD_AUDIO`、`MANAGE_EXTERNAL_STORAGE`）。

### 3. Application：初始化 + 注册

```java
import android.app.Application;
import android.content.Context;

import androidx.annotation.NonNull;

import dji.v5.common.error.IDJIError;
import dji.v5.common.register.DJISDKInitEvent;
import dji.v5.manager.SDKManager;
import dji.v5.manager.interfaces.SDKManagerCallback;
import dji.v5.network.DJINetworkManager;
import dji.v5.utils.common.LogUtils;

public class MyApplication extends Application {

    private static final String TAG = "MSDK";
    private volatile boolean sdkInited = false;

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        // 官方示例在 attachBaseContext 里装载 SDK 自带的 native 保护库
        // （类 com.cySdkyc.clx.Helper 由 dji-sdk-v5-aircraft.aar 提供）
        com.cySdkyc.clx.Helper.install(this);
    }

    @Override
    public void onCreate() {
        super.onCreate();
        initSDK();
    }

    private void initSDK() {
        SDKManager.getInstance().init(this, new SDKManagerCallback() {

            @Override
            public void onInitProcess(DJISDKInitEvent event, int totalProcess) {
                // 初始化完成后才能注册
                if (event == DJISDKInitEvent.INITIALIZE_COMPLETE) {
                    sdkInited = true;
                    SDKManager.getInstance().registerApp();
                }
            }

            @Override
            public void onRegisterSuccess() {
                LogUtils.i(TAG, "注册成功");
                // 注册成功后才开始连接硬件，之后可监听 ProductKey.KeyConnection
            }

            @Override
            public void onRegisterFailure(IDJIError error) {
                // 常见原因：无网络、App Key 与包名/签名不匹配、Key 被禁用
                LogUtils.e(TAG, "注册失败: " + error.errorCode() + " " + error.description());
            }

            @Override
            public void onProductConnect(int productId) {
                LogUtils.i(TAG, "产品已连接, productId=" + productId);
            }

            @Override
            public void onProductDisconnect(int productId) {
                LogUtils.i(TAG, "产品已断开, productId=" + productId);
            }

            @Override
            public void onProductChanged(int productId) {
                LogUtils.i(TAG, "产品变更, productId=" + productId);
            }

            @Override
            public void onDatabaseDownloadProgress(long current, long total) {
                // 限飞数据库下载进度，0~100
            }
        });

        // 注册失败多半是断网导致的：网络恢复且仍未注册时自动补一次
        DJINetworkManager.getInstance().addNetworkStatusListener(isAvailable -> {
            if (sdkInited && isAvailable && !SDKManager.getInstance().isRegistered()) {
                SDKManager.getInstance().registerApp();
            }
        });
    }
}
```

### 4. 注册成功后确认机型

```java
import dji.sdk.keyvalue.key.KeyTools;
import dji.sdk.keyvalue.key.ProductKey;
import dji.sdk.keyvalue.value.product.ProductType;
import dji.v5.common.callback.CommonCallbacks;
import dji.v5.manager.KeyManager;

// 同步读缓存（注册 + 连接后才会有值），读不到时返回默认值
ProductType type = KeyManager.getInstance().getValue(
        KeyTools.createKey(ProductKey.KeyProductType), ProductType.UNRECOGNIZED);

// 异步从硬件读一次（更可靠，适合首次进入页面）
KeyManager.getInstance().getValue(KeyTools.createKey(ProductKey.KeyProductType),
        new CommonCallbacks.CompletionCallbackWithParam<ProductType>() {
            @Override
            public void onSuccess(ProductType productType) { /* 机型 */ }

            @Override
            public void onFailure(@NonNull IDJIError error) { /* 未连接 */ }
        });
```

## 四、注意事项

1. **App Key 与签名绑定**：DJI 后台登记的是「包名 + 签名指纹」，所以 debug 包也要用同一套 keystore（示例工程 `debug` 也指向 `msdkkeystore.jks`）。换签名会直接注册失败。
2. **`SDKManagerCallback` 会被 SDK 长期强引用**：不要在回调里持有 Activity/Fragment，否则内存泄漏。用 Application 或全局单例持有。
3. **注册结果有本地缓存**：注册成功后再次启动无需联网，直到应用被重装。
4. **不要提前调 `registerApp()`**：必须在 `onInitProcess` 收到 `INITIALIZE_COMPLETE` 之后。
5. **`extractNativeLibs` 必须为 true**，否则 so 加载失败。
6. **LDM（本地数据模式）** 会切断 SDK 的网络访问，需要先在 DJI 官网申请 License；开启时可用 `LDMExemptModule` 放行个别模块（如 RTK）。详见 [[09_设备维护与负载扩展]]。
