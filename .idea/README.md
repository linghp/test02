# 说明文档
整体架构dagger2+retrofit2+okhttp3+rxjava2+mvp，gson解析，glide图片加载，
## dagger2
### dagger的好处
1. 增加开发效率，省去重复的体力劳动：
   省去new一个实例的操作，提供单例并不用担心是否线程安全。
2. 更好的管理类实例：
   每个app中的ApplicationComponent管理整个app的全局类实例，所有的全局类实例都统一交给ApplicationComponent管理，并且它们的生命周期与app的生命周期一样。
   每个页面对应自己的Component，页面Component管理着自己页面所依赖的所有类实例。因为Component，Module，整个app的类实例结构变的很清晰。
3. 解耦：
   不管被注入类的构造函数如何变化，在注入的地方都不需要改变；解耦还有个好处，就是方便测试，比如我切换到mock，只需要修改相应的Module即可。
4. 不用关心初始化顺序：
   它会在容器中寻找所依赖的对象，能找到就用，找不到就需要自己创建。
### dagger的用法
#### dagger的几个概念
* @Inject Inject主要有两个作用，一个是使用在构造函数上，通过标记构造函数让Dagger2来使用（Dagger2通过Inject标记可以在需要这个类实 例的时候来找到这个构造函数并把相关实例new出来）从而提供依赖，另一个作用就是标记在需要依赖的变量让Dagger2为其提供依赖。
* @Provide 用Provide来标注一个方法，该方法可以在需要提供依赖时被调用，从而把预先提供好的对象当做依赖给标注了@Injection的变量赋值。provide主要用于标注Module里的方法
* @Module 用Module标注的类是专门用来提供依赖的。有的人可能有些疑惑，看了上面的@Inject，需要在构造函数上标记才能提供依赖，那么如果我们需要提供 的类构造函数无法修改怎么办，比如一些jar包里的类，我们无法修改源码。这时候就需要使用Module了。Module可以给不能修改源码的类提供依 赖，当然，能用Inject标注的通过Module也可以提供依赖
* @Component Component一般用来标注接口，被标注了Component的接口在编译时会产生相应的类的实例来作为提供依赖方和需要依赖方之间的桥梁，把相关依赖注入到其中。
**ps：Component在注入对象的时候先去Module中找，如果找不到就会检查所有被@Inject标注的构造函数；**

#### 在本项目中的应用
本项目只有一个component（AppComponent），所有的module提供的数据都是通过这个component在application中进行初始化了。其实此时的初始化应该是全局通用的数据，专用的数据可以在具体用到的地方依赖相应的module，比如首页升级可以在activitymodules中的首页那个地方添加如下,首页有多个这种依赖的话逗号隔开。
```kotlin
@ContributesAndroidInjector(modules = [UpdaterModule::class])
HomeActivity HomeActivityInjector();
```
moudule应根据功能模块分类，不然会太多。
项目中额外增加了如下依赖是为了简化注入操作，具体可参考[此链接](https://blog.csdn.net/IO_Field/article/details/71730248)
```gradle
compile 'com.google.dagger:dagger-android:2.x'
annotationProcessor 'com.google.dagger:dagger-android-processor:2.x'
```


## 网络请求（retrofit2+okhttp3+rxjava2）
* 配置：在NetworkModule中提供了升级的retrofit、okhttp的配置及初始化；提供了msapi、保险、安逸花的retrofit，根据baseurl对应写一个retrofit，用@Qualifier限定；
**ps：如果只是变换baseurl可以用@Url（全路径）就可以，如果还需要更改头，请求时间等参数可以在networkmodule中提供retrofit**
* 请求封装：由于请求数据的生命周期应与页面一致（如果不一致，会造成数据返回时页面destroy，回调更新是可能会造成崩溃），
所以在baseactivity中统一加上RxPresenterDelegate，在生命周期destroy时进行释放。
* 请求拦截：okhttp可以添加拦截器对请求和响应数据进行拦截，如添加统一请求头、对数据统一篡改，添加日志等
* 请求响应操作：retrofit结合rxjava返回observable对请求进行组合，过滤，转换等操作

## mvp
view层负责界面的显示，一些事件监听，回调处理；presenter负责非ui相关的操作，如数据的处理,包括activity之间的传递的数据，sharedpreference等，调用接口，充当view和model的交互的桥梁；
model层负责数据的请求，接口配置，实体，数据源的变换。项目中具体指的是：
* m:data，api，net
* v:activiy,fragment,widget,adapter
* p:presenter

## gson解析

## 项目整体说明
* src下面有三个文件夹，分别为internal，main，production；internal和production对应着app/build.gradlew里面productFlavors的，如果还想要debug和release，
那么可以再加四个文件夹internal、production和debug、realease的组合。internal是我们开发环境，production是正式的；像url地址正式的和测试就用Flavor来分开，物理地址不一样，其实包地址是一样的；还有mock。
* 数据源的变换：DataSourceModule提供数据的来源，可以是正式的，测试的，mock的。其中mock时可以实现虚实相间，如下
```java
  @Inject public MsHttpUtilMock(Retrofit retrofit) {
    objectBaseMsObserver.code = 200;
    this.msApi = retrofit.create(MsApi.class);
  }
```
* 日志：日志有两种，一种电脑上logcat显示的，一种是app侧滑显示，这两种还是有些区别的；
第一种添加日志如下：
```java
    if (BuildConfig.DEBUG_ENABLE) {
      Timber.plant(new Timber.DebugTree() {
        @Override protected String createStackElementTag(StackTraceElement element) {
          return super.createStackElementTag(element) + ":" + element.getLineNumber();
        }
      });
    }
 ```
 这种加了if判断在debug下才显示日志，所以用Timber不用担心线上的日志问题；这里可以添加显示行号，方法名等。

 第二种侧滑日志是在debugger依赖库下面的LumberYard下面设置的tree
 ```java
   LumberYard(Context context) {
     this.context = context.getApplicationContext();
     cleanUp(this.context);
     Timber.plant(tree());
   }

   private Timber.Tree tree() {
     return new Timber.DebugTree() {
       @Override protected void log(int priority, String tag, String message, Throwable t) {
         addLog(new Log(priority, tag, message));
       }
     };
   }

   private synchronized void addLog(Log entry) {
     entries.addLast(entry);
     if (entries.size() > BUFFER_SIZE) {
       entries.removeFirst();
     }

     entrySubject.onNext(entry);
   }

     Flowable<Log> logs() {
       return entrySubject.onBackpressureBuffer();
     }
   ```
   entrySubject是rxjava下面的一个被观察者，是FlowableProcessor的子类，支持背压，在LogPrinterLayout的布局中订阅实时显示日志，代码如下
```java
  @Override protected void onAttachedToWindow() {
    super.onAttachedToWindow();

    adapter.setLogs(lumberYard.bufferedLogs());
    subscriptions.add(lumberYard.logs()
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Consumer<Log>() {
          @Override public void accept(Log log) throws Exception {
            adapter.addLog(log);
          }
        }, new Consumer<Throwable>() {
          @Override public void accept(Throwable throwable) throws Exception {
            Timber.e(throwable.getMessage());
          }
        }));
  }
```

* 环境选择：开发环境才有，目前包括dev，mock，product，自定义。可以扩展，在ENV枚举添加就行。在env中对应着环境值，每增加一个url都应在其extraUrls中增加相应的值，mock可以不管。
获取url时在Flavor中根据顺序值来获取即可。

* BaseActivity：为了环境选择和自带日志加入了appContainer.bind(this)，如下，在布局R.layout.layout_container中可以放些通用的头布局
```java
        ViewGroup rootView = appContainer.bind(this);
        getLayoutInflater().inflate(R.layout.layout_container, rootView);
```
sendMsg方法是用来替换handler的，presenter也是有的，尽量用persenter的，mvp嘛

* 数据保存：sharedpreference保存数据到本地，可以序列化成json，获取时再解析；也可以用fit库。

## pms-module

## device-module
