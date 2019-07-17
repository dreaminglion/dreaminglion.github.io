## 下载地址

http://dl-xda.xposed.info/framework/

## 修改 jar

1. jar 改为 apk，使用改之理进行包和代码里面包的编辑。
2. 改之理点击 ‘编译’ 生成 apk。
3. 取出生成 apk 中的 dex 等文件，替换回原 jar 中的对应文件。

说明：因为 jar 中没有文件完成性校验，所以可以替换回去。

## 重新打包

1. 使用改之理，修改 smali 文件相应的逻辑。
2. 编译打包为 apk。
3. 如果 2 报错，使用命令查看错误信息 - \Work\com.example.testfix> java -jar apktool.jar  b  ./
```
 1 java -jar 打印说明  2 ./  编译当前目录
```
4. 查看命令行报错信息，修改错误编辑文件，重新执行命令 查看是否继续报错。
5. 编译通过之后，使用 2 流程签名打包。


# xp 编译修改

## 修改特征

1. Xposed xposed_shared.h  
   installer 安装器包名 - de.robv.android.xposed.installer   -- com.jjdd.ppff
2. Xposed xposed.h
   xposed.prop
   libxposed_dalvik.so
   libxposed_art.so
   /system/framework/XposedBridge.jar
   bin/XposedBridge.jar.newversion
   de.robv.android.xposed.XposedBridge  jar 的包名

# so 二进制文件修改技巧

1. 把字符串转为 16 进制码，手敲进制码，在需要修改的多个文件中查找定位。
2. 找到特征码，确定开头修改的位置，以及最近的 00 结束位。
3. 从开始位，替换修改的内容，补充后面的内容到 00 结束位。
4. 3 中补充的 00 位和原本 00 位之间多出的内容，由于没有头文件的读取成为无效内容，完成修改。

5. 使用 ue 与原文件比较 二进制，确认修改内容。

## 进度

1. 修改 app_process32_xposed 中的 de.robv.android.xposed.installer -- com.jjdd.ppff
