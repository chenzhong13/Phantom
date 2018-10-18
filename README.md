[![license](http://img.shields.io/badge/license-Apache2.0-brightgreen.svg?style=flat)](https://github.com/ManbangGroup/Phantom/blob/master/LICENSE)
[![Release Version](https://img.shields.io/badge/release-3.0.0-brightgreen.svg)](https://github.com/ManbangGroup/Phantom/releases)

# Phantom — 唯一零 Hook 稳定占坑类 Android 热更新插件化方案

Phantom 是满帮集团开源的一套稳定、灵活、兼容性好的 Android 插件化方案。

## Phantom 的优势

* 兼容性好：**零** Hook，没有调用系统的 hidden API，完美兼容 Android 9.0
* 功能完整：插件支持独立应用的绝大部分特性
* 稳定可靠：历经货车帮旗下多款产品 50+ 插件两年多千万级用户验证（稳定性和兼容性指标都在 4 个 9 以上）
* 部署灵活：宿主无需升级（无需在宿主 AndroidManifest.xml 中预埋组件），即可支持插件新增组件，甚至新增插件
* 易于集成：无论插件端还是宿主端，只需『数行』就能完成接入，改造成本低

## Phantom 与主流开源插件框架的对比

| 特性 | Atlas | Small | VirtualAPK | RePlugin | Phantom |
| ---- | ---- | ---- | ---- | ---- | ---- |
| Hook 数量 | 大量 | 较多 | 较多 | 仅一处 | 无 |
| 四大组件 | 全支持 | 只支持 `Activity` | 全支持 | 全支持 | 除 `ContentProvider` 外，全支持 |
| 剔除公共库 | 支持 | 支持 | 支持 | 不支持 | 支持 |
| 兼容性适配 | 高 | 高 | 高 | 高 | 非常高 |
| 插件热更新 | 不支持 | 不支持 | 不支持 | 不支持 | 支持 |
| 插件快速部署 | 不支持 | 不支持 | 不支持 | 支持 | 支持 |
| 插件宿主通信 | 一般 | 一般 | 弱 | 一般 | 强 |

## 接入指南

### 宿主端

#### 添加 Gradle 配置

在宿主项目根目录下的 `build.gradle` 中增加宿主 gradle 依赖

```groovy
buildscript {
    dependencies {
      classpath 'com.wlqq.phantom:phantom-host-gradle:3.0.0'
    }
}
```

在宿主工程 Application 模块的 `build.gradle` 中增加宿主 library 依赖，并应用宿主 gradle 依赖包含的 gradle 插件 `com.wlqq.phantom.host`

```groovy
dependencies {
    compile 'com.wlqq.phantom:phantom-host-lib:3.0.0'
}

apply plugin: 'com.wlqq.phantom.host'
```

#### 初始化 Phantom 插件框架

在宿主工程 Application 模块中的 `Application#onCreate()` 方法中初始化 Phantom

```java
public class YourApplication extends Application {
    @Override
    public void onCreate() {
       super.onCreate();
       PhantomCore.getInstance().init(this, new PhantomCore.Config());
    }
}
```

#### 安装内置到宿主 assets 中的插件 APK 并启动插件中的 Activity

```java
// 安装打包到宿主 assets 中 plugins 目录下的插件
InstallResult ret = PhantomCore.getInstance().installPluginFromAssets("plugins/com.wlqq.phantom.pluign.component_1.0.0.apk");
// 插件安装成功后启动插件(执行插件的 Application#onCreate 方法)
if (ret.isSuccess() && ret.plugin.start()) {
    Intent intent = new Intent();
    // 指定插件 Activity 所在的插件包名以及 Activity 类名
    intent.setClassName("com.wlqq.phantom.pluign.component", "com.wlqq.phantom.pluign.component.MainActivity");
    PhantomCore.getInstance().startActivity(this, intent);
}
```

### 插件端

#### 添加 Gradle 配置

在插件项目根目录下的 `build.gradle` 中增加插件 gradle 依赖

```groovy
buildscript {
    dependencies {
      classpath 'com.wlqq.phantom:phantom-plugin-gradle:3.0.0'
    }
}
```

在插件项目 Application 模块的 `build.gradle` 中增加插件 library 依赖，并应用宿主 gradle 依赖包含的 gradle 插件 `com.wlqq.phantom.plugin`

```groovy
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            // Phantom 混淆配置文件
            proguardFile 'proguard-phantom.pro'
        }
    }
}

dependencies {
    provided 'com.wlqq.phantom:phantom-plugin-lib:3.0.0'
    compile 'com.android.support:support-v4:28.0.0'
}

apply plugin: 'com.wlqq.phantom.plugin'

phantomPluginConfig {
    // BEGIN 剔除公共库配置
    // 若插件中有使用 support-v4 ，则需要剔除掉(必须)
    excludeLib "com.android.support:support-v4:28.0.0"
    // END

    // BEGIN 生成插件额外的混淆配置文件，避免因剔除公共库引起的混淆问题
    libraryJarsProguardFile file('proguard-phantom.pro')
    // END

    // BEGIN 快速部署插件配置
    // 宿主包名
    hostApplicationId = "com.wlqq.phantom.sample"
    // 宿主 launcher Activity full class name
    hostAppLauncherActivity = "com.wlqq.phantom.sample.MainActivity"
    // 插件包名
    pluginApplicationId = android.defaultConfig.applicationId
    // 插件版本名
    pluginVersionName = android.defaultConfig.versionName
    // END
}
```

#### 在插件 `AndroidManifest.xml` 中申明对宿主 Phantom 插件框架版本依赖（目前版本名是 3.0.0，对应版本号为 30000）

```xml
<meta-data
    android:name="phantom.service.import.PhantomVersionService"
    android:value="30000"/>
```

#### 编译插件

与编译独立 APK 相同，如：

* `./gradlew assembleDebug`
* `./gradlew assembleRelease`

#### 编译插件并将插件 APK 安装到宿主

插件端使用的 Gradle 插件会自动为项目的 variant 生成相应的插件安装 task ，格式为 `phInstallPlugin${variant}` ，例如：

* `./gradlew phInstallPluginDebug`
* `./gradlew phInstallPluginRelease`

## 进阶说明

* [插件框架初始化](docs/phantom-core-init.md)
* [插件管理](docs/plugin-management.md)
* [四大组件](docs/components.md)
* [Phantom 通信服务](docs/phantom-service.md)
* [插件 meta-data](docs/android-manifest-metadata.md)
* [Phantom 安全签名校验](docs/security.md)
* [Native 支持](docs/native.md)
* [已知问题](docs/known-issues.md)

## 示例源代码

* [phantom-sample](phantom-sample)

## 联系我们

如果你在使用过程中遇到问题，或者有好的建议，欢迎给我们提 [issue](https://github.com/ManbangGroup/Phantom/issues) 或 [Pull Request](https://github.com/ManbangGroup/Phantom/pulls)。详细说明请移步 [Contributing Guide](CONTRIBUTING.md)

## License

Phantom is licensed under the Apache License 2.0. See the [LICENSE](LICENSE) file.

## 参考以及使用的开源项目

| 项目名称 | 开源协议 | 说明 |
| ---- | ---- | ---- |
| [Maven](https://github.com/apache/maven) | [Apache License](https://github.com/apache/maven/blob/master/LICENSE) | 依赖库版本比较 |
| [jsemver](https://github.com/zafarkhaja/jsemver) | [MIT License](https://github.com/zafarkhaja/jsemver/blob/master/LICENSE) | 依赖库版本比较 |
| [Atlas](https://github.com/alibaba/atlas) | [Apache License](https://github.com/alibaba/atlas/blob/master/LICENSE) | 首次加载插件提速 jar 包及 so 库 |
| [RePlugin](https://github.com/Qihoo360/RePlugin) | [Apache License](https://github.com/Qihoo360/RePlugin/blob/master/LICENSE) | gradle plugin 快速部署插件到宿主<br/>反射工具类 ReflectUtils |
