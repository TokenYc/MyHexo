title: 【非日常】AndroidStudio，opncv，ndk，jni,人脸识别示例程序
---
毕设要做人脸识别。在网上查了资料，决定使用AS+NDK+JNI+OpenCV完成。于是就在坑摸爬滚打了几天。今天总算把OpenCV里的人脸识别示例程序运行起来了。

现在的情况是：熟悉android，c/c++以前学过基础，没有在项目中使用过，opencv没有碰过。

### 几个概念
 - **NDK**提供编译打包等工具，是一个工具的集合。提供一定的API。
 - **Jni** 需要明确的是：这是java的东西，不是android的东西。作用是提供api与其他语言通信。使java与本地已编译好的代码交互。
 - **.so** 与Android手机cpu架构有关的动态链接库。说到底就是一个库文件。其实本来是linux下的。比如某些地图api的集成需要导入.so库，多个文件对应着不同的cpu架构。
 - **静态链接库**所有的依赖文件都编译进去。
 - **动态链接库**编译的时候不把依赖编译进去，运行时动态查找。
 - **交叉编译**在一个平台生成另一个平台的可执行的代码。
 - **mk文件**指定需要编译的文件，还有一些库的引入。在ndk-build的时候会用到。

### 参考的文章

 - [Android Studio ndk-Jni开发详解](http://www.open-open.com/lib/view/open1451917048573.html)这篇文章主要介绍了如何实现HelloWorld。把具体的操作流程说了一遍。我按这个做的时候遇到了一个问题，找不到头文件具体的实现：c/c++文件。发现原来不能完全copy，c文件里的方法名要和生成头文件相同。另外这篇文章里的代码贴的代码有一些小问题，不过都比较容易解决。**在实现过后感觉懵懵懂懂，知道了怎么做，却不知道为什么这么做。**对照下面这个图会帮助理解。
 
![](http://7xpp4m.com1.z0.glb.clouddn.com/jni.png)


 - [Android Studio 使用jni编译opencv完整实例](http://www.tuicool.com/articles/M3mQVjq)这篇文章介绍了他如何运行opencv和遇到的坑。我就是因为这个坑找到这篇文章的。就是那个找不到头文件的问题。根据文章里的方法确实解决了，然后又遇到了另一个问题。运行后找不到对应的c方法。后来发现是因为没有生成对应我的手机的cpu架构的so文件。

### 具体实现流程
1.当然是创建project啦，这里需要注意的是包名。原因的话后面就知道了。其他的和平时创建没有什么不同。

2.下载opencv for android [OpenCV官网](http://opencv.org/)

3.安装，设置NDK，在Project Structure中设置NDK，如果没有的话直接可以点击下载。

4.导入OpenCV-android-sdk\samples\face-detection。在AS中，file->new->import module，直接下一步下一步，finish。

5.Message会提示
![](http://7xpp4m.com1.z0.glb.clouddn.com/useNDK.png)
在gradle.properties中添加android.useDeprecatedNdk=true即可。

6.看这个目录

![](http://7xpp4m.com1.z0.glb.clouddn.com/dir.png)
jni文件夹里这几个东西是是干嘛的？先不管，运行了再说。然后出现了这个。

7.编译so文件
![](http://7xpp4m.com1.z0.glb.clouddn.com/errorh.png)
找不到这个文件。其实，java代码里调用的是so库，而不是直接调用c文件。所以我们要先生成so库。
先到jni目录 cd 你的目录,然后调用 ndk-build命令(注意ndk要配置环境变量)。然后又出现了一个问题，如下图。

![](http://7xpp4m.com1.z0.glb.clouddn.com/error2.png)
在Android.mk里出的问题，找不到OpenCV.mk。去opencv的sdk里找找，发现其实是有opencv.mk的，那应该是路径问题了。把图里的位置改成自己sdk里opencv.mk的位置.
![](http://7xpp4m.com1.z0.glb.clouddn.com/error3.png)

8.运行
改好后再次ndk-build。应该可以成功运行了，然后在main文件夹下生成了obj和libs文件夹。这时候运行一下。那个找不到头文件的问题依然在。我按照上面参考文章说的，其实在生成so文件后那个文件就已经没有用了。把它删了就好。
再次运行，运行成功，在安装opencv manager后，又报错了！
![](http://7xpp4m.com1.z0.glb.clouddn.com/error4.png)
找不到方法的实现。这里是因为没有调用so文件。
在build.gradle中的android下添加代码

	android {
	    compileSdkVersion 14
	    buildToolsVersion "19.1.0"
	
	    defaultConfig {
	        applicationId "org.opencv.samples.facedetect"
	        minSdkVersion 8
	        targetSdkVersion 8
	
	        ndk {
	            moduleName "detection_based_tracker"
	        }
	    }
	
	    buildTypes {
	        release {
	            minifyEnabled false
	            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.txt'
	        }
	    }
	-----------------------------------------------
	    sourceSets {
	
	        main() {
	
	            jniLibs.srcDirs = ['src/main/libs']
	
	        }
	
	    }
	-----------------------------------------------
	}
再次点运行。可以运行了！心理瞬间有底了。如果还有找不到方法实现的问题，可能是因为不支持你的机器的cpu架构。在application.mk中可以修改。

现在思考的问题是，怎么调试程序。想想都烦的很，得找找有什么好的解决方案。



