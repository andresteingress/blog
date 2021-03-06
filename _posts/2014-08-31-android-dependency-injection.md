---
layout: post
title: Android Dependency Injection
categories:
- android
- java
tags: []
status: publish
type: post
published: true
---
I recently came across a [very good collection of open source projects](http://square.github.io/) maintained by the guys from [Square](https://squareup.com/), a commercial payment service. 

One of their projects came in use in one of my current Android projects: Dagger, a light-weight dependency injection library that can be used in Java and Android projects. Dagger kind of introduced me to the world of dependency injection libraries for Android.

### DI and Android

If you are like me with a history (and present) in writing Java backend code, you will consider dependency injection as an almost given pattern when you start writing applications. For Android applications, dependency injection not only gets interesting for injecting things such as services or repositories, but also for injecting views. 

Let's have a look at the more familiar pattern of injecting Java objects/beans like services into activities.

## DI with Dagger

Dagger is a light-weight dependency injection library that comes with support for Java's JSR-330 annotations like `@Inject` or `@Singleton` . The creation of object instances that need to be injected at runtime are created in a so-called module definition:

{% highlight java %}
@Module
public class ApplicationModule {

    @Provides
    @Singleton
    public AppConnector providesAppConnector() {

        OkHttpClient okHttpClient = new OkHttpClient();
        okHttpClient.setConnectTimeout(ApplicationConstants.HTTP_TIMEOUT, TimeUnit.MILLISECONDS);
        okHttpClient.setWriteTimeout(ApplicationConstants.HTTP_TIMEOUT, TimeUnit.MILLISECONDS);
        okHttpClient.setReadTimeout(ApplicationConstants.HTTP_TIMEOUT, TimeUnit.MILLISECONDS);

        try {
            File cacheDir = new File(application.getCacheDir(), "http-cache");
            Cache cache = new Cache(cacheDir, 1024 * 1024 * 5l); // 5 MB HTTP Cache

            okHttpClient.setCache(cache);
        } catch (IOException e) {
            Log.e("ApplicationModule", "Could not create cache directory for HTTP client: " + e.getMessage(), e);
        }

        RestAdapter restAdapter = new RestAdapter.Builder()
                .setEndpoint(application.getString(R.string.appEndPoint))
                .setLogLevel(RestAdapter.LogLevel.FULL)
                .setClient(new OkClient(okHttpClient))
                .setConverter(new GsonConverter(gson))
                .build();

        return restAdapter.create(AppConnector.class);
    }
}
{% endhighlight %}

In the case above, the module definition provides a REST interface called `AppConnector`. Therefore, it configures an `OkHttpClient` instance and the REST endpoint. At runtime, the `ApplicationModule` needs to be instantiated and the dependency object graph defined by the module needs to be bootstrapped. This can be done in a custom `Application` implementation:

{% highlight java %}
public class Application extends android.app.Application {

    private ObjectGraph objectGraph;

    @Override
    public void onCreate() {
        super.onCreate();

       objectGraph = ObjectGraph.create(new ApplicationModule());
    }

    public ObjectGraph getObjectGraph() {
        return objectGraph;
    }
}
{% endhighlight %}

In order to actually trigger the dependency injection for an activity, `ObjectGraph#inject()` needs to be called. This is typically done inside a common activity base class:

{% highlight java %}
public abstract class BaseActivity extends FragmentActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        ((Application) getApplication()).getObjectGraph().inject(this);
    }
}
{% endhighlight %}

Once this initial setup is done, activities can be injected with the objects created in the module:

{% highlight java %}
public class LoginActivity extends BaseActivity {

    @Inject
    AppConnector appConnector;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        appConnector.doSomething(...);
    }
}
{% endhighlight %}

Of course, Dagger also comes with more advanced conecepts and features. But with the features shown above, you can already start to use dependency injection for services, repositories and other patterns.

## VDI with Butterknife

When developing on Android, glue code to set instance variables which are all `View` descendants is frequently needed:

{% highlight java %}
public class LoginActivity extends Activity {

    private EditText username;
    private EditText password;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

	setContentView(R.layout.login);
        
        username = (EditText) findViewById(R.id.username);
        password = (EditText) findViewById(R.id.password);

        // ...
    }

    // ...
}
{% endhighlight %}

View dependecy injection is a way to get rid of glue code that is needed to link instance variables with resources from the layout, and to speed up development. One library that is determined to be used exactly for this case is [Butterknife](http://jakewharton.github.io/butterknife/). It comes with annotations that can be used to inject views, but also to annotated methods which at compile-time are transformed to listener instances.

But let's first of all have a look at view injection. Actually, it is pretty simple, the annotation to use is `@InjectView`. It needs a single annotation parameter which is the resource ID of the targeted view:

{% highlight java %}
public class LoginActivity extends BaseActivity {

    @InjectView(R.id.username)
    EditText username;

    @InjectView(R.id.password)
    EditText password;

    // ...
}
{% endhighlight %}

No more code needed for manually retrieving and casting the view instances in the `onCreate` Activity callback. One little detail that is hidden here is the call to `ButterKnife.inject(this)` which actually causes Butterknife to inject all the instance fields with the target views at runtime. I do the calls to `inject` usually in my `BaseActivity` which is shared across all activities in my project:

{% highlight java %}
public abstract class BaseActivity extends FragmentActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        ButterKnife.inject(this);
    }

    // ...
}
{% endhighlight %}

Among other nice Butterknife features, I really like the annotations for various listeners, take `@OnClick` for example. This is a method level annotation that annotates methods which are at compile-time transformed/wrapped with an actual listener implementation:

{% highlight java %}
public class LoginActivity extends Activity {

    @OnClick(R.id.btnLogin)
    public void login() {
        String user = username.getText().toString();
        // ...
    }
}
{% endhighlight %}

Or suppose the layout has a Spinner and we want to react when an item is selected:

{% highlight java %}
@OnItemSelected(R.id.mobilePrefix)
public void mobilePrefixSelected(int position) {
    // ...
}
{% endhighlight %}

The method targeted by `OnItemSelected` may come with any parameters also found in the actual listener interface.

### Conclusion

I used `Dagger` and `Butterknife` in one of my latest Android projects and I won't go without them anymore. Especially `Butterknife` comes in very handy as it allows to get rid of view glue code that is needed to get references to UI view elements.

It should be noted that there is also the [AndroidAnnotations](http://androidannotations.org/) project that comes with an even richer set of annotations to do basically everything with annotations. In my current projects, our needs were totally satisfied with the `Butterknife` feature set, so I can't say anything about `AndroidAnnotations` in depth. 
