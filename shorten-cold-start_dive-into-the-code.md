Dive into the code, Real Plan  
- Application onCreate
  - Analytic Commander
  - UserPreferences
  - UIUtils.initReservatoni
  - Bilan
- Clean code
  - Do one thing and do it well
  - onCreate things done
    - 
- 


- Open onCreate
  - find the methods
  - verify their dependencies accross the application
  - any Unit Tests?
  - Service registry concept
- refactor
  - multidex? already up (progard?)
  - quick test with 3 threads
  - Results on NimbleDroid
  - problems that threads can cause
    - instance not initialized when needed -> NPE
- Lazy Loading
  - Concept
  - Singleton vs Single Instance
  - Dagger 2
    - pros
    - why not using it
  - Lazy Loading class
    - add
    - addLazy
    - getInstance
  - replace ServiceRegistry
    - Replace Structurally
    - fix UnitTests

_________________________________________________
part 2 : Dive into the code
*** paste this data at the end of the previous post ***

# Dive into the code
## Application onCreate
As we have seen in the previous post, those are the three methods that are blocking the startup:
- AnalCommander // This is analytics ;)
- UserPreferences
- initReservation


```java
@Override public void onCreate() {
  DineApp.appContext = getApplicationContext();
  super.onCreate();
  [...]
  final Map<Class, Object> registry = new HashMap<>();
  registry.put(AnalCommander.class, new AnalCommander(appContext)); // this one is blocking
  ServiceRegistry.create(registry);

  UserPreferences.init(this); // this one too
  [...]
  UIUtils.initReservation(this); // and finally this one
}
```

So, everything is on the onCreate without any threading. I have removed non relevant code. Let`s study the different blockers one by one.

### Analytic Commander
It's an analytic tool that helps merging all libraries of this purpose in a single access point. As all analytic tools, it`s used everywhere in the application and to avoid recreating everytime, is created in the initialization of the application and stored in the ServiceRegistry (a Class/Instance map singleton).

[![Analytic Commander][callstack_oncreate_analcommander]][callstack_oncreate_link]

As you can see here, his initialization is taking most of his time with a `javax.xml.xpath` and different tag parsings. This librarie is used everywhere in the app. Creating a normal thread to set it up can cause errors farther.

### UserPreferences
`UserPreferences.init(this)` loads all user data from json files stored in SharedPreferences. Then it keeps it in memory during the whole application lifecycle. That can be Search History, User favorites etc...

[![User Preferences][callstack_oncreate_userpreference]][callstack_oncreate_link]

```java
String jsonFavPlaces = sp.getString(KEY_PREFS_FAVORITE_PLACES, "[]");
favPlaces = gson.fromJson(jsonFavPlaces, new TypeToken<HashMap<String, DineMerchantPreference>>() {
    }.getType());
```
During initialization, it relies heavily on gson reflection causing multiple heavy {action/time hog/processing/script} that blocks the main thread. The New York Times have written a really good post on [Improving their Startup Time][newyorktimes_improvingstartuptime] where they encounter the same problem. They solve it by using [gson with custom type adapters][gson_custom_types]. On our side, most of our objects are not used in the firsts seconds of startup so we will be fine just creating them in a background thread.

### UIUtils.initReservation
This method creates default items as people number, hour of reservation etc... It's only used in a second activity: Search and so we can delay the creation.

![UIUtils initReservation][callstack_oncreate_uiutils_joda]

```java
@NonNull
private static DateTime getSelectedDateTime(int days) {
    return DateTime.now().plusDays(days);
}
```

But let's take a look on this line, ```DateTime.now().plusDays(days)```. is blocking the UIThread for around 1 second. We always should do most of the work on a background thread to avoid this behaviour.

### Sum Up
So we have Analytic Commander that takes time to initialize, used everywhere in the app but is just an api call and do not interact with the UI. We can launch it in a thread as a fire and forget mode.  
Then we have UserPrefences that are needed not long after the launch screen, that impacts UI. This one should be initialized first and we should have a way to access this data the sonner without blocking the UI.  
To finish, UIUtils is not needed right after the launch screen but impact the UI. This one can be initialized in a low priority thread at startup.

___________________
Page 3?

# Modify code
## Clean Up
Code should be self explanatory and a method should do only one thing. This is not the case in this onCreate so let's split it up.

```java
@Override public void onCreate() {
    DineApp.appContext = getApplicationContext();
    super.onCreate();

    RegisterToAppForeground();

    buildAppServices();
    Fabric.with(this, new Crashlytics());
    setUpAppsFlyerLib();
    setUpUrbanAirshipLib();
    setUpAppRateLib();
    setUpFacebookLib();
    UserPreferences.init(this);
    UIUtils.initReservation(this);
    setDebugTheme();
    //LeakCanary.install(this);
  }
```

Now that we have a much cleaner and comprehensive onCreate we can start fixing the boot time. But how to do it cleanly?

## What's Lazy Loading
- what's lazy loading
- Dagger
  - Do not use magics
  - if you have to debug it you must understand it
  - Not the case of all the developpers
  - Will speak about it in another project
- link to lazyloading github

```java
public class LazyServiceRegistry {
    [...]

    public interface Callback<T> { void onInstanceReceived(T instance); }

    public interface LazySetter<T>{ T get(); }

    public <T> void addInstance(Class clazz, T instance) {...}

    public <T> void addLazy(Class clazz, LazySetter<T> lazySetter) {...}

    public <T> T get(Class<T> clazz) {...}

    public <T> void get(final Class<T> clazz, final Callback<T> callback) {...}
} 
```
  
### How it works
This class is an extended version of the basic ServiceRegistry we were using. It adds the possibility to delay instance creation by using the LazySetter interface. If the object is not instanciated, it will be created the first time it's called and will be put in memory for later use. It's also possible to ask for a creation on a thread and use a callback for when it's ready. 
***Use of interfaces to access a class*** 

## Replace Service Registry
First of all we need to replace the previous Service registry in the ```DineApp``` class. 

```java
protected static LazyServiceRegistry registry; 

private void buildAppServices() {
    registry = new LazyServiceRegistry();
    registry.addInstance(AnalCommander.class, new AnalCommander(appContext)); // adding in synchronously to begin
}

public static LazyServiceRegistry getRegistry() { // to access it easily in the whole application
    return registry;
}  
```

Previous ServiceRegistry was static for convenience in the code. The problem is that it's hard to mock in unit tests. That's why I'm using it as a single instance. To avoid NullPointerExceptions on some constructors, I have added a ```isInitialized(Class clazz)``` method whose goal is to check if the method have already been initialized. Well... Self explanatory name :)

```java
public CurrentLocation() {
-    if (ServiceRegistry.getInstance(CurrentLocation.class) != null) {}
+    if (getRegistry().isInstanciated(CurrentLocation.class)) { }
}
```

########## No increase in performance -> check in nimbledroid

## AnalyticCommander 
### Initialization
Let's start by Analytics Commander, we will add through ```addLazy(Class, LazySetter)``` method
- lazySetter in onCreate

### Calls async
- getInstanceAsync -> fire and forget

### NimbleDroid speed up?
_______________________

## UserPreferences
### Initialization
- remove static from class

### Start instance to preload

### Calls Async

### NimbleDroid speed up?

_______________________
## UIUtils
### Initialization
- remove static from class

### Start instance to preload

### Calls sync

### NimbleDroid speed up
_______________________

# Conclusion









What we will see in next one + link











- Singleton pattern is not the solution, welcome single instance






[comment]: <> (IMAGES)
[callstack_oncreate_analcommander]: images/callstack_onCreate_analcommander.png
[callstack_oncreate_userpreference]: images/callstack_onCreate_userPreferences.png
[callstack_oncreate_uiutils_joda]: images/callstack_onCreate_UIUtils_initReservation.png

[comment]: <> (LINKS)
[callstack_oncreate_link]: https://nimbledroid.com/play/com.ypg.dine?p=323DVxanEq1ssS#com.ypg.dine.DineApp.onCreate
[newyorktimes_improvingstartuptime]: http://open.blogs.nytimes.com/2016/02/11/improving-startup-time-in-the-nytimes-android-app/?_r=0
[gson_custom_types]: https://google-gson.googlecode.com/svn/trunk/gson/docs/javadocs/com/google/gson/TypeAdapter.html
