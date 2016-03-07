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

# Dive into the code
## Application onCreate
As we have seen in the previous post, those are the three methods that are blocking the startup:
- AnalCommander // This is analytics ;)
- UserPreferences
- initReservation

So, everything is on the onCreate without any threading. I have removed non relevant code for this post.

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
}```

`AnalCommander` is an analytic tools that helps concentrate all analytics in one point. It's used in all the application and is instantiated into a ServiceRegistry. The service registry is a Class/Instance map whose goal is to keep in memory and provide object instances everywhere in the app.

`UserPreferences`

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


[New York Times, Improving Startup Time](http://open.blogs.nytimes.com/2016/02/11/improving-startup-time-in-the-nytimes-android-app/?_r=0)
