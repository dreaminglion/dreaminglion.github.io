---
layout: post
title:  "视屏传输机制"
categories: xkqk
---

## 使用方法

1. 使用 adb 把 Main.dex 文件推送到手机硬盘，指定文件路径并运行 com.wanjian.puppet.Main class 文件。
2. 使用 adb 连接指定 ip 和 port adb forward tcp:8888 localabstract:puppet-ver1
3. 运行lib目录下的Client，用于显示和控制,点击连接按钮即可

<!-- more -->

## Main.dex

组成：由整个 lib 编辑成的 dex 文件。

main 函数：

1. 获取连接的 client。
2. 读取客服端的命令执行对应的操作。
3. 获取手机的屏幕显示，通过 socket 的输出流把 bitmap 转为二进制数组进行发送。

### screenshot() 函数，获取手机显示 Bitmap。

```
public static Bitmap screenshot() throws Exception {

    String surfaceClassName;
    Point size = SurfaceControlVirtualDisplayFactory.getCurrentDisplaySize(false);
    size.x *= scale;
    size.y *= scale;
    Bitmap b = null;
    if (Build.VERSION.SDK_INT <= 17) {
        surfaceClassName = "android.view.Surface";
    } else {
        surfaceClassName = "android.view.SurfaceControl";
        //  b = android.view.SurfaceControl.screenshot(size.x, size.y);
    }
    b = (Bitmap) Class.forName(surfaceClassName).getDeclaredMethod("screenshot", new Class[]{Integer.TYPE, Integer.TYPE}).invoke(null, new Object[]{Integer.valueOf(size.x), Integer.valueOf(size.y)});

    int rotation = wm.getRotation();

    if (rotation == 0) {
        return b;
    }
    Matrix m = new Matrix();
    if (rotation == 1) {
        m.postRotate(-90.0f);
    } else if (rotation == 2) {
        m.postRotate(-180.0f);
    } else if (rotation == 3) {
        m.postRotate(-270.0f);
    }
    return Bitmap.createBitmap(b, 0, 0, size.x, size.y, m, false);
}

```

### Android 手机服务器发送机制

**在子线程中获取 socket 的输出流，通过无限循环的获取手机显示的 bitmap 使用输出流进行发送 bitmap 的二进制数组**

```
private static void write(final LocalSocket socket) {
    new Thread() {
        @Override
        public void run() {
            super.run();
            try {
                final int VERSION = 2;
                BufferedOutputStream outputStream = new BufferedOutputStream(socket.getOutputStream());
                while (true) {
                    Bitmap bitmap = screenshot();
                    ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
                    bitmap.compress(Bitmap.CompressFormat.JPEG, 60, byteArrayOutputStream);

                    outputStream.write(2);
                    writeInt(outputStream, byteArrayOutputStream.size());
                    outputStream.write(byteArrayOutputStream.toByteArray());
                    outputStream.flush();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }.start();
}

```

## Client 接收展示

创建客户端，连接手机的服务器，获取手机服务器发送的 image 对应的二进制流，转为 image，在 jui 的界面进行展示。

```
/**
 *
 * @param ip
 * @param port
 * @throws IOException
 *
 * 通过 socket，连接手机做为的服务器端，获取服务器发送的文件流转为 image Image image = ImageIO.read(byteArrayInputStream);
 * label.setIcon(new ScaleIcon(new ImageIcon(image))); 进行显示
 *
 */
private void read(final String ip, final String port) throws IOException {
    new Thread() {
        @Override
        public void run() {
            super.run();
            try {
                Socket socket = new Socket(ip, Integer.parseInt(port));
                BufferedInputStream inputStream = new BufferedInputStream(socket.getInputStream());
                writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
                byte[] bytes = null;
                while (true) {
                    long s1 = System.currentTimeMillis();
                    int version = inputStream.read();
                    if (version == -1) {
                        return;
                    }
                    int length = readInt(inputStream);
                    if (bytes == null) {
                        bytes = new byte[length];
                    }
                    if (bytes.length < length) {
                        bytes = new byte[length];
                    }
                    int read = 0;
                    while ((read < length)) {
                        read += inputStream.read(bytes, read, length - read);
                    }
                    InputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
                    long s2 = System.currentTimeMillis();
                    Image image = ImageIO.read(byteArrayInputStream);
                    label.setIcon(new ScaleIcon(new ImageIcon(image)));
                    long s3 = System.currentTimeMillis();

                }

            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }.start();

}

```
