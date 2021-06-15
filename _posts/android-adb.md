---
layout: post
title:  "android adb"
date:   2016-11-18 22:15:00 +0800
categories: android
---


## android 运行 java main 函数

### 在 android app 环境下运行

1.在 activity 中，使用 classLoader 调用 com.wanjian.puppet.Main 的 main 方法，可以成功启动但是与下文启动 runtime 相同。

<!-- more -->

```
ClassLoader classLoader = MainActivity.class.getClassLoader();
Class<?> loadClass = classLoader.loadClass("com.wanjian.puppet.Main");
Method method = loadClass.getMethod("main", String[].class);
method.invoke(null, new Object[]{new String[]{}});

```

2.在 activity 中创建实例调用

```
Main m = new Main();
m.main(null);

```

### 使用 app_process 调用 java main 函数

1. adb shell pm path com.xiake.xiami 得到 apk 安装路径
2. adb shell CLASSPATH=" + apkPath（1 得到的路径） + " exec app_process /system/bin com.wanjian.puppet.Main"

## adb 与 adb shell

`adb shell` 是 android Linux 的控制台，控制手机系统，android 并没有 adb 工具，所以不能在 android app 执行 adb 命令。
`adb` adb.exe 是一个 PC 的文件，android debug bridge 用于 PC 操作手机的工具，所以 adb 可以操作 android 手机的 adb shell 功能。

### java 环境执行 adb

**使用 android activity 运行 execAdb（adb） 出现 IOException，只能用 java 环境运行。**

定义一个 java 文件编写 main 函数，点击 run 直接运行。

```
public static String execAdb(String adb) {

    try {

        Process process =  Runtime.getRuntime().exec(adb);

        BufferedReader mReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        StringBuffer mRespBuff = new StringBuffer();
        char[] buff = new char[1024];
        int ch = 0;
        while ((ch = mReader.read(buff)) != -1) {
            mRespBuff.append(buff, 0, ch);
        }
        mReader.close();
        String result = mRespBuff.toString();
        return result;

    } catch (IOException e) {
        e.printStackTrace();
    }

    return "";
}

public class Adb {

    public static void main(String[] args) {

        String path = AdbUtil.execAdb("adb shell pm path com.xiake.xiami");

        System.out.println("knight path is " + path);

        String ph = "package:";

        if (path != null && path.startsWith(ph)) {
            path = path.substring(ph.length());
        }

        final String finalPath = path;

        Thread t1 = new Thread() {
            @Override
            public void run() {
                AdbUtil.execAdb("adb shell CLASSPATH=" + finalPath + " exec app_process /system/bin com.wanjian.puppet.Main");
            }
        };
        t1.setDaemon(true);
        t1.start();

        AdbUtil.execAdb("adb forward tcp:12580 localabstract:puppet-ver1");

        System.out.println("knight com.wanjian.puppet.Main start 666...");
    }

}
```

### android activity 执行 adb shell

**在 android activity 中可以使用 adb shell 执行 app_process 相当于 java 环境运行 main 函数。**

```
private void startMainServer() {
    ShellUtils.CommandResult result = ShellUtils.execCommand(
            "pm path com.xiake.xiami");

    ToastUtil.show(UIUtil.getContext(), result.successMsg);

    String ph = "package:";

    if (result.successMsg != null && result.successMsg.startsWith(ph)) {
        result.successMsg = result.successMsg.substring(ph.length());
    }

    final String finalPath = result.successMsg;

    Thread t1 = new Thread() {
        @Override
        public void run() {
            ShellUtils.execCommand("CLASSPATH=" + finalPath + "  exec app_process /system/bin  com.wanjian.puppet.Main");
        }
    };
    t1.setDaemon(true);
    t1.start();

    log("adb is started 666...");
}

```

ShellUtils 核心代码

```
// 执行命令
process = Runtime.getRuntime().exec(isRoot ? “su” : “sh”);
os = new DataOutputStream(process.getOutputStream());
for (String command : commands) {
    if (command == null) {
        continue;
    }

    // donnot use os.writeBytes(commmand), avoid chinese charset error
    os.write(command.getBytes());
    os.writeBytes(COMMAND_LINE_END);
    os.flush();
}
os.writeBytes(COMMAND_EXIT);
os.flush();

// 拿到返回结果
successMsg = new StringBuilder();
errorMsg = new StringBuilder();

successResult = new BufferedReader(new InputStreamReader(process.getInputStream()));
errorResult = new BufferedReader(new InputStreamReader(process.getErrorStream()));
String s;
while ((s = successResult.readLine()) != null) {
    successMsg.append(s);
}
while ((s = errorResult.readLine()) != null) {
    errorMsg.append(s);
}

```
