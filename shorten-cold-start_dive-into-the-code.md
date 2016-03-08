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
*** Add disclaimer, not working in this project and only focusing in improving it's performance on personal side project ***

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
During initialization, it relies heavily on gson reflection causing multiple heavy {action/time hog/processing/script} that blocks the main thread. The New York Times have written a really good post on [Improving their Startup Time][newyorktimes_improvingstartuptime] where they encounter the same problem. They solve it by using [gson with custom type adapters][gson_custom_types]. and the cost of reflection of this library. For us, most of the objects are not used in the firsts seconds of startup so we will probably

`UIUtils.initReservation`





***[ScreenShot dine Nimbledroid's onCreate]***
in the onCreate, we can see 3 methods taking most of the time
UserPreferences
JodaTime
other one

Checking in the code
***Screenshot of onCreate***
Separation and cleanup of onCreate



- Singleton pattern is not the solution, welcome single instance
-> [nice explanation](http://programmers.stackexchange.com/a/40610/212413)




**What's Lazy Loading**

From Wikipedia, Lazy




[comment]: <> (IMAGES)
[callstack_oncreate_analcommander]: images/callstack_onCreate_analcommander.png
[callstack_oncreate_userpreference]: images/callstack_onCreate_userPreferences.png

[comment]: <> (LINKS)
[callstack_oncreate_link]: https://nimbledroid.com/play/com.ypg.dine?p=323DVxanEq1ssS#com.ypg.dine.DineApp.onCreate
[newyorktimes_improvingstartuptime]: http://open.blogs.nytimes.com/2016/02/11/improving-startup-time-in-the-nytimes-android-app/?_r=0
[gson_custom_types]: https://google-gson.googlecode.com/svn/trunk/gson/docs/javadocs/com/google/gson/TypeAdapter.html
