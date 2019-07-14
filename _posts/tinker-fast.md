---
title: tinker fast
---

[Tinker](http://www.tinkerpatch.com/Docs/intro)补丁管理与快速接入，不修改代码逻辑。只需修改Application名称为SampleApplication。

<!-- more -->

## SDK 接入

### 在项目根目录修改gradle.properties文件

``` bash
TINKER_VERSION=1.7.6
TINKERPATCH_VERSION=1.0.1
```
引用tinker的版本号,以及快速接入SDK tinkerPatch 版本号。

### 添加 gradle 插件依赖

在项目根目录的build.gradle添加tinkerPatch gradle引用。 ${TINKERPATCH_VERSION} 为上一步添加在gradle.properties中的属性。

``` bash
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.3'
        //无需再单独引用tinker的其他库
        classpath "com.tinkerpatch.sdk:tinkerpatch-gradle-plugin:${TINKERPATCH_VERSION}"
    }
}
```

### 在App项目的build.gradle添加SKD的引用。

``` bash
// multidex SampleApplication 中使用到的一个类包
compile "com.android.support:multidex:1.0.1"
//若使用annotation需要单独引用,对于tinker的其他库都无需再引用
provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
compile("com.tinkerpatch.sdk:tinkerpatch-android-sdk:${TINKERPATCH_VERSION}") { changing = true }
```


### 在App目录下添加tinkerpatch.gradle，单独管理tinker的配置。需要在App的build.gradle中进行引用  **apply from: 'tinkerpatch.gradle'**。


tinkerpatch.gradle :

``` bash
apply plugin: 'tinkerpatch-support'
// ${buildDir} 取AS中App下build的文件夹路径
def bakPath = file("${buildDir}/bakApk/")

// 1.0.0 基准apk所在的文件夹
def appName = "app-1.0.0-0109-19-19-40"

/**
 * 对于插件各参数的详细解析请参考
 * http://tinkerpatch.com/Docs/SDK
 */
tinkerpatchSupport {
    // tinkerEnable 功能是否开启
    tinkerEnable = true
    // 是否反射 Application 实现一键接入；一般来说，接入 Tinker 我们需要改造我们的 Application, 若这里为 true， 即我们无需对应用做任何改造即可接入。
    reflectApplication = true
    // tinkerpatch 中注册的  appKey
    appKey = "9f202100fd2d769c"
    // 发布基准包apk的版本
    appVersion = "1.0.0"
    //取上文中def bakPath的路径
    autoBackupApkPath = "${bakPath}"

    // 基准apk文件位置
    baseApkFile = "${bakPath}/${appName}/app-release.apk"
    // rf 不进行混淆，不混淆不会有 mapping.txt 生成
    baseProguardMappingFile = "${bakPath}/${appName}/app-release-mapping.txt"
    // 基准apk中资源文件
    baseResourceRFile = "${bakPath}/${appName}/app-release-R.txt"
}

/**
 * 一般来说,我们无需对下面的参数做任何的修改
 * 对于各参数的详细介绍请参考:
 * https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
 */
tinkerPatch {
    ignoreWarning = false
    useSign = true
    dex {
        dexMode = "jar"
        pattern = ["classes*.dex"]
        loader = []
    }
    lib {
        pattern = ["lib/*/*.so"]
    }

    res {
        pattern = ["res/*", "r/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
        ignoreChange = []
        largeModSize = 100
    }

    packageConfig {
    }
    sevenZip {
        zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
//        path = "/usr/local/bin/7za"
    }
    buildConfig {
        keepDexApply = false
    }
}

```


### 在App项目的build.gradle引用tinkerpatch.gradle。
app build.gradle :

```
apply plugin: 'com.android.application'
// 引用tinkerpatch.gradle
apply from: 'tinkerpatch.gradle'

android {
    compileSdkVersion '19'
    buildToolsVersion "23.0.2"

    defaultConfig {
        applicationId "your applicationId"
        minSdkVersion 16
        targetSdkVersion 19
        versionCode 1
        versionName "1.0"
    }

    signingConfigs {
        release {
            storeFile rootProject.file("keystore/package.jks")
            storePassword "your storePassword"
            keyAlias "your keyAlias"
            keyPassword "your keyPassword"
        }
        debug {
            storeFile rootProject.file("keystore/debug.keystore")
        }
    }

    buildTypes {
        release {
            // rf 不进行混淆，不混淆不会有 mapping.txt 生成
            minifyEnabled false
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt')
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
    }

    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }

}

repositories {
    maven {
        url "https://jitpack.io"
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile "com.android.support:multidex:1.0.1"
    //若使用annotation需要单独引用,对于tinker的其他库都无需再引用
    provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    compile("com.tinkerpatch.sdk:tinkerpatch-android-sdk:${TINKERPATCH_VERSION}") { changing = true }
    //    compile 'com.android.support:appcompat-v7:23.4.0'
    compile 'eu.the4thfloor.volley:com.android.volley:2015.05.28'
    compile 'com.google.code.gson:gson:2.7'
    compile 'com.jakewharton:butterknife:7.0.1'
    compile 'com.android.support:support-v4:24.2.1'
    compile 'com.android.support:recyclerview-v7:23.1.0'
    compile 'com.github.luckyandyzhang:CleverRecyclerView:1.0.0'
    compile 'com.squareup.picasso:picasso:2.5.2'
    compile 'com.bm.photoview:library:1.3.6'
}

```


## 初始化 TinkerPatch SDK
SampleApplication :

```
public class SampleApplication extends Application {

    private static final String TAG = "Tinker.SampleApp";

    private ApplicationLike tinkerApplicationLike;
    public SampleApplication() {

    }

    @Override
    public void onCreate() {
        super.onCreate();

        //我们可以从这里获得Tinker加载过程的信息
        tinkerApplicationLike = TinkerPatchApplicationLike.getTinkerPatchApplicationLike();

        //开始检查是否有补丁，这里配置的是每隔访问3小时服务器是否有更新。
        TinkerPatch.init(tinkerApplicationLike)
                .reflectPatchLibrary()
                .setPatchRollbackOnScreenOff(true)
                .setPatchRestartOnSrceenOff(true);

        //每隔3个小时去访问后台时候有更新,通过handler实现轮训的效果
        new FetchPatchHandler().fetchPatchWithInterval(3);
    }

    @Override
    public void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        //you must install multiDex whatever tinker is installed!
        MultiDex.install(base);
    }

    /**
     * 在这里给出TinkerPatch的所有接口解释
     * 更详细的解释请参考:http://tinkerpatch.com/Docs/api
     */
    private void useSample() {
        TinkerPatch.init(tinkerApplicationLike)
                //是否自动反射Library路径,无须手动加载补丁中的So文件
                //注意,调用在反射接口之后才能生效,你也可以使用Tinker的方式加载Library
                .reflectPatchLibrary()
                //向后台获取是否有补丁包更新,默认的访问间隔为3个小时
                //若参数为true,即每次调用都会真正的访问后台配置
                .fetchPatchUpdate(true)
                //设置访问后台补丁包更新配置的时间间隔,默认为3个小时
                .setFetchPatchIntervalByHours(3)
                //向后台获得动态配置,默认的访问间隔为3个小时
                //若参数为true,即每次调用都会真正的访问后台配置
                .fetchDynamicConfig(new ConfigRequestCallback() {
                    @Override
                    public void onSuccess(HashMap<String, String> hashMap) {

                    }

                    @Override
                    public void onFail(Exception e) {

                    }
                }, false)
                //设置访问后台动态配置的时间间隔,默认为3个小时
                .setFetchDynamicConfigIntervalByHours(3)
                //设置当前渠道号,对于某些渠道我们可能会想屏蔽补丁功能
                //设置渠道后,我们就可以使用后台的条件控制渠道更新
                .setAppChannel("default")
                //屏蔽部分渠道的补丁功能
                .addIgnoreAppChannel("googleplay")
                //设置tinkerpatch平台的条件下发参数
                .setPatchCondition("test", "1")
                //设置补丁合成成功后,锁屏重启程序
                //默认是等应用自然重启
                .setPatchRestartOnSrceenOff(true)
                //我们可以通过ResultCallBack设置对合成后的回调
                //例如弹框什么
                .setPatchResultCallback(new ResultCallBack() {
                    @Override
                    public void onPatchResult(PatchResult patchResult) {
                        Log.i(TAG, "onPatchResult callback here");
                    }
                })
                //设置收到后台回退要求时,锁屏清除补丁
                //默认是等主进程重启时自动清除
                .setPatchRollbackOnScreenOff(true)
                //我们可以通过RollbackCallBack设置对回退时的回调
                .setPatchRollBackCallback(new RollbackCallBack() {
                    @Override
                    public void onPatchRollback() {
                        Log.i(TAG, "onPatchRollback callback here");
                    }
                });
    }

    /**
     * 自定义Tinker类的高级用法, 一般不推荐使用
     * 更详细的解释请参考:http://tinkerpatch.com/Docs/api
     */
    private void complexSample() {
        TinkerPatch.Builder builder = new TinkerPatch.Builder(tinkerApplicationLike);
        //修改tinker的构造函数,自定义类
        builder.listener(new DefaultPatchListener(this))
                .loadReporter(new DefaultLoadReporter(this))
                .patchReporter(new DefaultPatchReporter(this))
                .resultServiceClass(TinkerServerResultService.class)
                .upgradePatch(new UpgradePatch())
                .patchRequestCallback(new TinkerPatchRequestCallback());

        TinkerPatch.init(builder.build());
    }
}

```

## TinkerPatch使用步骤

- 使用命令行 ./gradlew assembleRelease 生成基准包。
![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/tinker-fast_bulid.jpg)
- 把基准包apk信息填入tinkerpatch.gradle文件中。
- 修改代码bug,完成修改与基准包代码生成差异。
- 使用命令行 ./gradlew tinkerPatchRelease 生成补丁包,补丁包将位于 build/outputs/tinkerPatch 中。
- 把build/outputs/tinkerPatch/patch_signed_7zip.apk在[tinkerpatch](http://www.tinkerpatch.com/Docs/start)中进行发布.
- [灰度与条件下发](http://www.tinkerpatch.com/Docs/rule)
