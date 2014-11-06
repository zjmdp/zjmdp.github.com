---
layout: post
title: "基于Proxy思想的Android插件框架"
description: ""
category: 
tags: []
---
{% include JB/setup %}

###本文所有代码托管在Github：[android-plugin](https://github.com/zjmdp/android-plugin)

##意义

研究插件框架的意义在于以下几点：

* 减小安装包的体积，通过网络选择性地进行插件下发
* 模块化升级，减小网络流量
* 静默升级，用户无感知情况下进行升级
* 解决低版本机型方法数超限导致无法安装的问题
* 代码解耦

##现状
Android中关于插件框架的技术已经有过不少讨论和实现，插件通常打包成apk或者dex的形式。

dex形式的插件往往提供了一些功能性的接口，这种方式类似于java中的jar形式，只是由于Android的Dalvik VM无法直接动态加载Java的Byte Code，所以需要我们提供Dalvik Byte Code，而dex就是Dalvik Byte Code形式的jar。

apk形式的插件提供了比dex形式更多的功能，例如可以将资源打包进apk，也可实现插件内的Activity或者Service等系统组件。

本文主要讨论apk形式的插件框架，对于apk形式又存在安装和不安装两种方式

* 安装apk的方式实现相对简单，主要原理是通过将插件apk和主程序共享一个UserId，主程序通过```createPackageContext```构造插件的context，通过context即可访问插件apk中的资源，很多app的主题框架就是通过安装插件apk的形式实现，例如Go主题。这种方式的缺点就是需要用户手动安装，体验并不是很好。

* 不安装apk的方式解决了用户手动安装的缺点，但实现起来比较复杂，主要通过```DexClassloader```的方式实现，同时要解决如何启动插件中Activity等Android系统组件，为了保证插件框架的灵活性，这些系统组件不太好在主程序中提前声明，实现插件框架真正的难点在此。


##DexClassloader

这里引用《深入理解Java虚拟机：JVM高级特性与最佳实践》第二版里对java类加载器的一段描述：

>虚拟机设计团队把类加载阶段中的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“类加载器”。

Android虚拟机的实现参考了java的JVM，因此在Android中加载类也用到了类加载器的概念，只是相对于JVM中加载器加载class文件而言，Android的Dalvik虚拟机加载的是Dex格式，而具体完成Dex加载的主要是```PathClassloader```和```Dexclassloader```。

```PathClassloader```默认会读取```/data/dalvik-cache```中缓存的dex文件，未安装的apk如果用```PathClassloader```来加载，那么在```/data/dalvik-cache```目录下找不到对应的dex，因此会抛出```ClassNotFoundException```。

```DexClassloader```可以加载任意路径下包含dex和apk文件，通过指定odex生成的路径，可加载未安装的apk文件。下面一段代码展示了```DexClassloader```的使用方法：

{% highlight java %}
final File optimizedDexOutputPath = context.getDir("odex", Context.MODE_PRIVATE);
try{
    DexClassLoader classloader = new DexClassLoader("apkPath",
            optimizedDexOutputPath.getAbsolutePath(),
            null, context.getClassLoader());
    Class<?> clazz = classloader.loadClass("com.plugindemo.test");
    Object obj = clazz.newInstance();
    Class[] param = new Class[2];
    param[0] = Integer.TYPE;
    param[1] = Integer.TYPE;
    Method method = clazz.getMethod("add", param);
    method.invoke(obj, 1, 2);
}catch(InvocationTargetException e){
    e.printStackTrace();
}catch(NoSuchMethodException e){
    e.printStackTrace();
}catch(IllegalAccessException e){
    e.printStackTrace();
}catch(ClassNotFoundException e){
    e.printStackTrace();
}catch (InstantiationException e){
    e.printStackTrace();
}
{% endhighlight %}


```DexClassloader```解决了类的加载问题，如果插件apk里只是一些简单的API调用，那么上面的代码已经能满足需求，不过这里讨论的插件框架还需要解决资源访问和Android系统组件的调用。

##插件内系统组件的调用

Android Framework中包含```Activity```，```Service```，```Content Provider```以及```BroadcastReceiver```等四大系统组件，这里主要讨论如何在主程序中启动插件中的Activity，其它3种组件的调用方式类似。

大家都知道Activity需要在AndroidManifest.xml中进行声明，apk在安装的时候```PackageManagerService```会解析apk中的AndroidManifest.xml文件，这时候就决定了程序包含的哪些Activity，启动未声明的Activity会报```ActivityNotFound```异常，相信大部分Android开发者曾经都遇到过这个异常。

启动插件里的Activity必然会面对如何在主程序中的AndroidManifest.xml中声明这个Activity，然而为了保证插件框架的灵活性，我们是无法预知插件中有哪些Activity，所以也无法提前声明。

为了解决上述问题，这里介绍一种基于Proxy思想的解决方法，大致原理是在主程序的AndroidManifest.xml中声明一些```ProxyActivity```，启动插件中的Activity会转为启动主程序中的一个```ProxyActivity```，```ProxyActivity```中所有系统回调都会调用插件Activity中对应的实现，最后的效果就是启动的这个Activity实际上是主程序中已经声明的一个Activity，但是相关代码执行的却是插件Activity中的代码。这就解决了插件Activity未声明情况下无法启动的问题，从上层来看启动的就是插件中的Activity。下面具体分析整个过程。

###PluginSDK

所有的插件和主程序需要依赖PluginSDK进行开发，所有插件中的Activity继承自PluginSDK中的```PluginBaseActivity```，```PluginBaseActivity```继承自```Activity```并实现了```IActivity```接口。

{% highlight java %}

public interface IActivity {
    public void IOnCreate(Bundle savedInstanceState);

    public void IOnResume();

    public void IOnStart();

    public void IOnPause();

    public void IOnStop();

    public void IOnDestroy();

    public void IOnRestart();

    public void IInit(String path, Activity context, ClassLoader classLoader, PackageInfo packageInfo);
}

{% endhighlight %}


{% highlight java %}

public class PluginBaseActivity extends Activity implements IActivity {
	...
	private Activity mProxyActivity;
	...
	
	@Override
    public void IInit(String path, Activity context, ClassLoader classLoader) {
        mProxy = true;
        mProxyActivity = context;

        mPluginContext = new PluginContext(context, 0, path, classLoader);
        attachBaseContext(mPluginContext);
    }
	
	@Override
    protected void onCreate(Bundle savedInstanceState) {
        if (mProxy) {
            mRealActivity = mProxyActivity;
        } else {
            super.onCreate(savedInstanceState);
            mRealActivity = this;
        }
    }

    @Override
    public void setContentView(int layoutResID) {
        if (mProxy) {
            mContentView = LayoutInflater.from(mPluginContext).inflate(layoutResID, null);
            mRealActivity.setContentView(mContentView);
        } else {
            super.setContentView(layoutResID);
        }
    }

    ...

    @Override
    public void IOnCreate(Bundle savedInstanceState) {
        onCreate(savedInstanceState);
    }

    @Override
    public void IOnResume() {
        onResume();
    }

    @Override
    public void IOnStart() {
        onStart();
    }

    @Override
    public void IOnPause() {
        onPause();
    }

    @Override
    public void IOnStop() {
        onStop();
    }

    @Override
    public void IOnDestroy() {
        onDestroy();
    }

    @Override
    public void IOnRestart() {
        onRestart();
    }
}

{% endhighlight %}

{% highlight java %}

public class ProxyActivity extends Activity {
    IActivity mPluginActivity;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Bundle bundle = getIntent().getExtras();
        if(bundle == null){
            return;
        }
        mPluginName = bundle.getString(PluginConstants.PLUGIN_NAME);
        mLaunchActivity = bundle.getString(PluginConstants.LAUNCH_ACTIVITY);
        File pluginFile = PluginUtils.getInstallPath(ProxyActivity.this, mPluginName);
        if(!pluginFile.exists()){
            return;
        }
        mPluginApkFilePath = pluginFile.getAbsolutePath();
        try {
            initPlugin();
            super.onCreate(savedInstanceState);
            mPluginActivity.IOnCreate(savedInstanceState);
        } catch (Exception e) {
            mPluginActivity = null;
            e.printStackTrace();
        }
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        if(mPluginActivity != null){
            mPluginActivity.IOnResume();
        }
    }

    @Override
    protected void onStart() {
        super.onStart();
        if(mPluginActivity != null) {
            mPluginActivity.IOnStart();
        }
    }
    
    ...
    
    private void initPlugin() throws Exception {
  		PackageInfo packageInfo = PluginUtils.getPackgeInfo(this, mPluginApkFilePath);

        if (mLaunchActivity == null || mLaunchActivity.length() == 0) {
            mLaunchActivity = packageInfo.activities[0].name;
        }

        ClassLoader classLoader = PluginUtils.getClassLoader(this, mPluginName, mPluginApkFilePath);

        if (mLaunchActivity == null || mLaunchActivity.length() == 0) {
            if (packageInfo == null || (packageInfo.activities == null) || (packageInfo.activities.length == 0)) {
                throw new ClassNotFoundException("Launch Activity not found");
            }
            mLaunchActivity = packageInfo.activities[0].name;
        }
        Class<?> mClassLaunchActivity = classLoader.loadClass(mLaunchActivity);

        getIntent().setExtrasClassLoader(classLoader);
        mPluginActivity = (IActivity) mClassLaunchActivity.newInstance();
        mPluginActivity.IInit(mPluginApkFilePath, this, classLoader);
    }
    
    ...
    
    @Override
    public void startActivityForResult(Intent intent, int requestCode) {
    	boolean pluginActivity = intent.getBooleanExtra(PluginConstants.IS_IN_PLUGIN, false);
        if (pluginActivity) {
            String launchActivity = null;
            ComponentName componentName = intent.getComponent();
            if(null != componentName) {
                launchActivity = componentName.getClassName();
            }
            intent.putExtra(PluginConstants.IS_IN_PLUGIN, false);
            if (launchActivity != null && launchActivity.length() > 0) {
                Intent pluginIntent = new Intent(this, getProxyActivity(launchActivity));

                pluginIntent.putExtra(PluginConstants.PLUGIN_NAME, mPluginName);
                pluginIntent.putExtra(PluginConstants.PLUGIN_PATH, mPluginApkFilePath);
                pluginIntent.putExtra(PluginConstants.LAUNCH_ACTIVITY, launchActivity);
                startActivityForResult(pluginIntent, requestCode);
            }
        } else {
			super.startActivityForResult(intent, requestCode);
        }
    }

{% endhighlight %}


```PluginBaseActivity```和```ProxyActivity```在整个插件框架的核心，下面简单分析一下代码：

首先看一下```ProxyActivity#onResume```：

{% highlight java %}

@Override
protected void onResume() {
    super.onResume();
    if(mPluginActivity != null){
        mPluginActivity.IOnResume();
    }
}

{% endhighlight %}

变量```mPluginActivity```的类型是```IActivity```，由于插件Activity实现了```IActivity```接口，因此可以猜测```mPluginActivity.IOnResume()```最终执行的是插件Activity的```onResume```中的代码，下面我们来证实这种猜测。

```PluginBaseActivity```实现了```IActivity```接口，那么这些接口具体是怎么实现的呢？看代码：

{% highlight java %}

@Override
public void IOnCreate(Bundle savedInstanceState) {
    onCreate(savedInstanceState);
}

@Override
public void IOnResume() {
    onResume();
}

@Override
public void IOnStart() {
    onStart();
}

@Override
public void IOnPause() {
    onPause();
}

...

{% endhighlight %}

接口实现非常简单，只是调用了和接口对应的回调函数，那这里的回调函数最终会调到哪里呢？前面提到过所有插件Activity都会继承自```PluginBaseActivity```，也就是说这里的回调函数最终会调到插件Activity中对应的回调，比如```IOnResume```执行的是插件Activity中的```onResume```中的代码，这也证实了之前的猜测。

上面的一些代码片段揭示了插件框架的核心逻辑，其它的代码更多的是为实现这种逻辑服务的，后面会提供整个工程的源码，大家可自行分析理解。

##插件内资源获取

实现加载插件apk中的资源的一种思路是将插件apk的路径加入主程序资源查找的路径中，下面的代码展示了这种方法：

{% highlight java %}

private AssetManager getSelfAssets(String apkPath) {
	AssetManager instance = null;
	try {
		instance = AssetManager.class.newInstance();
		Method addAssetPathMethod = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
		addAssetPathMethod.invoke(instance, apkPath);
	} catch (Throwable e) {
		e.printStackTrace();
	}
	return instance;
}
	
{% endhighlight %}
	
为了让插件Activity访问资源时使用我们自定义的Context，我们需要在```PluginBaseActivity```的初始化中做一些处理：

{% highlight java %}

public void IInit(String path, Activity context, ClassLoader classLoader, PackageInfo packageInfo) {
    mProxy = true;
    mProxyActivity = context;

    mContext = new PluginContext(context, 0, mApkFilePath, mDexClassLoader);
    attachBaseContext(mContext);
}

{% endhighlight %}

```PluginContext```中通过重载```getAssets```来实现包含插件apk查找路径的Context：

{% highlight java %}

public PluginContext(Context base, int themeres, String apkPath, ClassLoader classLoader) {
	super(base, themeres);
	mClassLoader = classLoader;
	mAsset = getPluginAssets(pluginFilePath);
	mResources = getPluginResources(base, mAsset);
	mTheme = getPluginTheme(mResources);
}

private AssetManager getPluginAssets(String apkPath) {
	AssetManager instance = null;
	try {
		instance = AssetManager.class.newInstance();
		Method addAssetPathMethod = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
		addAssetPathMethod.invoke(instance, apkPath);
	} catch (Throwable e) {
		e.printStackTrace();
	}
	return instance;
}

private Resources getPluginAssets(Context ctx, AssetManager selfAsset) 	{
	DisplayMetrics metrics = ctx.getResources().getDisplayMetrics();
	Configuration con = ctx.getResources().getConfiguration();
	return new Resources(selfAsset, metrics, con);
}

private Theme getPluginTheme(Resources selfResources) {
	Theme theme = selfResources.newTheme();
	mThemeResId = getInnerRIdValue("com.android.internal.R.style.Theme");
	theme.applyStyle(mThemeResId, true);
	return theme;
}

@Override
public Resources getResources() {
	return mResources;
}

@Override
public AssetManager getAssets() {
	return mAsset;
}

...

{% endhighlight %}

##总结
本文介绍了一种基于Proxy思想的插件框架，所有的代码都在Github中，代码只是抽取了整个框架的核心部分，如果要用在生产环境中还需要完善，比如```Content Provider```和```BroadcastReceiver```组件的Proxy类未实现，Activity的Proxy实现也是不完整的，包括不少回调都没有处理。同时我也无法保证这套框架没有致命缺陷，本文主要是以总结、学习和交流为目的，欢迎大家一起交流。
