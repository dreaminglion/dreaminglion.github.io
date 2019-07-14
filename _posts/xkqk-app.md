---
layout: post
title:  "侠客群控"
categories: xkqk
---

## 项目组成

1.MsgActivity  
  没有任何逻辑，以及一个简单的 view 视图。（应该是一个 Android 应用必须注册一个 activity ？）

2.InstrumentationTestRunner 与 runner
  功能测试相关的测试包。

3.xIME
  继承 Android 系统 InputMethodService 的服务。

4.配置 meta-data xposed

5.项目中自定义，且并没有明确引用的 v4 v7 包的组件。

<!-- more -->

## xIME

1. xIME 继承于 InputMethodService
  - 对 Android 手机进行信息输入的核心服务，直接调用方法可以模拟输入事件。

2. 复写 onCreateInputView
  - 用户创建用户输入区域的按键视图。
  - 当输入的区域第一次展示的时候仅回调一次。
  - InputMethodService 默认直接返回 null 不做处理。
  - **项目中，这个回调主要用于实例化「BroadcastReceiver」实现广播事件的接收。**

3. AdbReceiver
  - 功能：通过 adb 发送广播，得到对应的「参数」后使用 InputConnection 对手机系统进行输入操作。
  - 消息类型：`IME_MESSAGE`，`IME_CHARS`，`IME_KEYCODE`，`IME_EDITORCODE`。
  - 事件：ic.commitText ic.performEditorAction ic.sendKeyEvent ic.performEditorAction 传递不同的事件到手机系统。

4. onStartInputView
  - 调用时机：当输入视图出现并开始编辑的时候，进行多次调用。
  - **SendMsgToServer("msg", "inputfoucs"); 方法中开启线程在文件夹 sdcard 中创建 share_`randomId` 的文件**
  - 在 onStartInputView 中调用 super.onStartCandidatesView(info, restarting); 不清楚用意。

##  XposedBridge

下载运行中，还没开始学习。

## 自定义 v4 v7 的模块功能

### v4 MyServer
- 通过 String[] args 参数，拿到一些参数配置信息。
- 开启线程读取存储在 sdcard 文件夹中的命令行文件，然后进行执行 RunCmd(cmd, msg); 就是使用 Socket 发送数据到服务器。
- 确定发送视频的模式「image、video」得到 ibinder 对象，AliveTick() 把录制的图片转为二进制，使用 socket 给服务器发送图片。
- 如果视频模式开启，使用 addSample 函数，SendDataToAll 函数，把视频转为二进制发送到服务器。

```
private void addSample(ByteBuffer buf, int size, long timeUs, int flags) {

    try {

        byte[] data = new byte[size];
        buf.duplicate().get(data);
        if (flags == MediaCodec.BUFFER_FLAG_CODEC_CONFIG) {

            //解码信息
            configbyte = new byte[size];
            configbyte = data;

        } else if (flags == MediaCodec.BUFFER_FLAG_KEY_FRAME || flags == 9) {

            //关键帧
            byte[] keyframe = new byte[size + configbyte.length];
            System.arraycopy(configbyte, 0, keyframe, 0, configbyte.length);
            System.arraycopy(data, 0, keyframe, configbyte.length, data.length);
            SendDataToAll(keyframe, (byte) 0);
        } else {
            SendDataToAll(data, (byte) 0);
        }

    } catch (Throwable e) {
        e.printStackTrace();
    }
}

private void SendDataToAll(final byte[] data, final byte index) {

    try {
        int length = clients.size();
        for (int i = 0; i < length; i++) {
            final Socket socket = clients.get(i);


            SendData(data, index, socket);

        }
    } catch (Throwable e) {
        log("循环出错：" + e.getMessage());
    }

}

```

### v7

还没开始看。
