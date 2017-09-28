# AndroidAOP
Android studio AOP编程

重点如下：
1. 点击安装AndroidAOP\AspectJDemo-master\aspectj-1.8.5.jar
2. 如果安装到c盘：C:\aspectj1.8\   
    配置环境变量 ：
    （1）将C:\aspectj1.8\lib\aspectjrt.jar 添加到CLASSPATH 
    （2）将C:\aspectj1.8\bin添加到Path

3. Android Studio配置 build.gradle
------下面全部复制过来，注意sdk版本
pply plugin: 'com.android.application'
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.aspectj:aspectjtools:1.8.9'
        classpath 'org.aspectj:aspectjweaver:1.8.9'
    }
}
repositories {
    mavenCentral()
}
final def log = project.logger
final def variants = project.android.applicationVariants
variants.all { variant ->
    if (!variant.buildType.isDebuggable()) {
        log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
        return;
    }

    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                         "-1.8",
                         "-inpath", javaCompile.destinationDir.toString(),
                         "-aspectpath", javaCompile.classpath.asPath,
                         "-d", javaCompile.destinationDir.toString(),
                         "-classpath", javaCompile.classpath.asPath,
                         "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
        log.debug "ajc args: " + Arrays.toString(args)

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler);
        for (IMessage message : handler.getMessages(null, true)) {
            switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                    log.warn message.message, message.thrown
                    break;
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}

android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"
    defaultConfig {
        applicationId "com.example.administrator.aspectjdemo"
        minSdkVersion 19
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}



dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.3.1'
    testCompile 'junit:junit:4.12'
    compile files('libs/aspectjrt.jar')
}
--------------------------------------
4. 权限校验android文件 PermissionManager.java,就是正常的单例实现方式

package com.example.administrator.aspectjdemo;

import android.content.Context;
import android.content.pm.PackageManager;
import android.support.v4.content.ContextCompat;
import android.util.Log;

/**
 * Created by Administrator on 2016/12/26.
 */

public class PermissionManager {
    private static volatile PermissionManager permissionManager;
    private static  final  String TAG = PermissionManager.class.getSimpleName();


    public PermissionManager(){}

    //DCL单例模式
    public static PermissionManager getInstance(){
        if (permissionManager == null){
            synchronized (PermissionManager.class){
                if (permissionManager == null){
                    permissionManager = new PermissionManager();
                }
            }
        }
        return permissionManager;
    }

    private static class InnerInsatance{
        public static final PermissionManager instance = new PermissionManager();
    }

    //内部类单例模式
    public static PermissionManager getInnerInstance(){
        synchronized (PermissionManager.class){
            return InnerInsatance.instance;
        }
    }

    public boolean checkPermission(String permission){
        Log.i(TAG,"checkPermission  : 检查的权限："+permission);
//        if (ContextCompat.checkSelfPermission(context,permission) != PackageManager.PERMISSION_GRANTED){
//            return true;
//        }
        if (permission.equals("android.permission.CAMERA")){
            return true;
        }
        return false;
    }
}

5. 定义的面向切向注解类AspectJAnnotation.java,通过AspectJAnnotation的定义，在android 里面方法上进行注解使用，
 String value()；表示参数类型是String, 在面向切向编程里，可以通过该value()获取注解时代入参值。
 
package com.example.administrator.aspectjdemo;


import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Created by Administrator on 2016/12/26.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AspectJAnnotation {
    String value();
}

6. android应用主界面 MainActivity.java

package com.example.administrator.aspectjdemo;

import android.Manifest;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;

public class MainActivity extends AppCompatActivity {

    private static  final  String TAG = MainActivity.class.getSimpleName();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button bt =(Button) this.findViewById(R.id.bt);
        bt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i(TAG,"onClick-->");
                test();
            }
        });
    }
    // 采用注解方式，进行面向切向进行切入，String类型入参是 Manifest.permission.CAMERA
    @AspectJAnnotation(value = Manifest.permission.CAMERA)
    public void test(){
        Log.i(TAG,"检查权限");
    }
}
7. 面向切向编程的实际实现过程 AspectJTest.java

ackage com.example.administrator.aspectjdemo;

import android.app.Activity;
import android.util.Log;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;

/**
 * Created by Administrator on 2016/12/26.
 */
@Aspect
public class AspectJTest {
    private static  final  String TAG = AspectJTest.class.getSimpleName();
    @Pointcut("execution(@com.example.administrator.aspectjdemo.AspectJAnnotation  * *(..))")
    public void executionAspectJ() {
    }

    @Around("executionAspectJ()")
    public Object aroundAspectJ(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        // 打印log
        Log.i(TAG, "aroundAspectJ(ProceedingJoinPoint joinPoint)");
        
        AspectJAnnotation aspectJAnnotation = methodSignature.getMethod().getAnnotation(AspectJAnnotation.class);
        String permission = aspectJAnnotation.value();
        Object o = null;
        if (PermissionManager.getInnerInstance().checkPermission(permission)) {
            o = joinPoint.proceed();
            Log.i(TAG, "有权限");
        } else {
            Log.i(TAG, "没有权限，不给用");
        }
        return o;
    }

}


该例子的UI非常简单，一个按钮，点击后会调用经过注解方式面向切向的test()方法，并打印对应log:


09-28 17:28:03.356 13132-13132/com.example.administrator.aspectjdemo I/MainActivity: onClick-->
09-28 17:28:03.359 13132-13132/com.example.administrator.aspectjdemo I/AspectJTest: aroundAspectJ(ProceedingJoinPoint joinPoint)
09-28 17:28:03.362 13132-13132/com.example.administrator.aspectjdemo I/PermissionManager: checkPermission  : 检查的权限：android.permission.CAMERA
09-28 17:28:03.363 13132-13132/com.example.administrator.aspectjdemo I/MainActivity: 检查权限
09-28 17:28:03.363 13132-13132/com.example.administrator.aspectjdemo I/AspectJTest: 有权限

