Essay that is published under [Profile Android Cold Starts](http://dubedout.eu/profile-android-cold-starts/) on [dubedout.eu](http://dubedout.eu/)

# Cold Starts
## What's that?
In Android, there is three types of starts: First Starts, Cold Starts, and Warm Starts.

- First starts, the slowest, is just after the application have been installed or updated. The app needs to (re)set up a database, load configuration, load the first batch of data. It's only occurring once, so it's not the most important.

- Warm starts, the fastest, is when you put your app in the background and go back to it. The app is still in memory, everything up and running. It's usually very fast and often doesn't need improvements.

- Cold start is between first and warm. When the device needs memory for launching new apps and that you app is in the background doing nothing, Android will release your app from the memory. Next time your app is launched, Android will have to initialize back every service it needs to run. It's frequently occurring, and launch time is often between one to few seconds. The user will notice and will not like it as you can see in this 2012's [report]. Time to launch is the first impression the user will have. 

## Why it's so slow
In everyday work, we use [libraries] to develop and achieve business requirements faster. Those focus on adding features instead of keeping an application quick and responsive because the latest is not adding business value, or not directly quantifiable.

A big part of used libraries are needed by the whole application and are often initialized in the ```Application.onCreate()```. When we initialize without any threading or lazy loading, we end up with an ```onCreate``` that takes more time to process. ```Application.onCreate()``` is the first method launched on every start. This method have to be completed before starting any Activity. So the longest it takes, the longest the first page will take to be displayed.

When the application is slow to start, it displays the [Zygote] longer, and it will break immersion. This root white start screen is not pretty by default. But instead of focusing on creating a beautiful app startup screen with the windowBackground we will concentrate on tracking what is taking time and reduce it.

# Profile YP Dine
[YP Dine] is a recent application of the play store built by another team in Yellow Pages Canada. His goal is to let you discover your next favorite restaurant, browse through handmade lists from local food experts or book a table directly from the app. I will dig into application initialization to check if we can have quick wins.

## What tools
Usually, I use [TraceView and DmTraceDump] to find the bottlenecks in the code and fix it. But today we will play with this new player: [NimbleDroid]. They use the same tools but display the results in a very easy to understand way. All performance tests are realized in the same conditions letting you compare with other applications. One of the nice tricks is that you can automatically hook your build from popular git hosting and therefore, keeping track about the cold start versions after version.
YP Dine isn't obfuscated so we can effortlessly check which part of the code is blocking. If you obfuscate yours,  it's still possible to add the ProGuard Mapping to reveal problematic methods.  

On a side note, a colleague shared me [Show Java app] that can be useful to see unobfuscated code apps.

## Tracing
Yp Dine is not a bad player here with 2400 ms to start. But we can see below that the DineApp.onCreate() is blocking the process for 2059 ms. Roughly 80% of his start time is blocked here, clicking on this line will open a new view with more details in it.

![2.6s launch time][YPDine_general]  

![onCreate 3 methods blocking startup][YPDine_onCreate]

On the call stack, we can see that three methods are taking most of the starting time:
- UIUtils.initReservation (JodaTime), 981ms
- UserPreferences, 553ms
- YpDine.Utils..., 364ms
If we can fix this, it will be a good improvement.

If those methods are blocking the UI, it probably means they are directly in the main thread, and if we can instantiate them asynchronously it will be a quick win.

Always keep in mind to work on the most important issues first, frames dropping on a ```ListView``` can be more frustrating that the start time. I like this post from coding horror on [Gold Plating], you should take a look if you don't know it.

# Let's go deeper
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

So, everything is on the onCreate without any threading. I have removed nonrelevant code. Let`s study the different blockers one by one.

## Analytic Commander
### What's going on
It's an analytic tool that helps to merge all analytics libraries in a single access point. As all analytic tools, it`s used everywhere in the application. To avoid creating the object every time used, it's done in the application initialization and stored in the ServiceRegistry (a Class/Instance map singleton).

[![Analytic Commander][callstack_oncreate_analcommander]][callstack_oncreate_link]

As you can see here, initialization is taking most of his time in **TagCommander** constructor and Tag parsings. This library is used everywhere in the app. Creating a background thread to set it up can cause errors later because trying to access an object not yet initialized.

### Fire and Forget
This one is used everywhere in the application, but it only sends data and does not interact with the UI. We do not have to wait for data to be returned from the API. In this case, we can use a Lazy Loading library like Dagger or [a Lazy Loading class] I have created some time ago. 

First you need to change the ServiceRegistry for the Lazy Loading class, create a method to access to this LazyLoading class accross whole application (like a static method) and finally create the initializer object:
```java
registry.addLazy(AnalCommander.class, new LazyServiceRegistry.LazySetter<AnalCommander>() {
      @Override
      public AnalCommander get() {
        return new AnalCommander(appContext);
      }
    });
```

Then, in a background thread, access to the instance and send the call. The Lazy Loading class will initialize it and store it locally for later use. We don't need to wait for the response (Retry on error should be part of the class). Now initialization and api call is async and not tied anymore in UIThread.

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        DineApp.getLazyRegistry().getInstance(AnalComander.class).sendPageView();
    }
}).start();
```

## UserPreferences
### What's going on
`UserPreferences.init(this)` loads all user data from JSON files stored in ```SharedPreferences```. Keeping it in memory during the whole application lifecycle for Search History, User favorites, etc...

[![User Preferences][callstack_oncreate_userpreference]][callstack_oncreate_link]

```java
String jsonFavPlaces = sp.getString(KEY_PREFS_FAVORITE_PLACES, "[]");
favPlaces = gson.fromJson(jsonFavPlaces, new TypeToken<HashMap<String, DineMerchantPreference>>() {
    }.getType());
```
During initialization, it relies heavily on GSON's reflection causing multiple heavy processing that blocks the main thread. The New York Times have written an excellent post on [Improving their Startup Time][newyorktimes_improvingstartuptime] where they go through the same problems. They solve it by using [gson with custom type adapters][gson_custom_types]. On our side, most of our objects are not used in the firsts seconds of startup, so we should be okay just creating them in a background thread.

### Don't put all eggs is the same basket
UserPreferences is maybe the more trickier. We needs his data to be displayed in the first ```Activity```. But this class have multiple problems, it's a **full static** class grouping very different data. So even if we need a little portion of it, it needs to be initialized fully. First we should split the data under a same package a bit like this:
- UserPreferences/```LastCities.class```
- UserPreferences/```FavoritePlaces.class```
- UserPreferences/```FavoriteCurators.class```
- UserPreferences/```FavoritePlaylists.class```
- UserPreferences/```SearchHistory.class```
- UserPreferences/```RecentMerchant.class```
- UserPreferences/```Reservations.class```  
etc...

Then remove all this static methods everywhere and use the Lazy Loading class to retain his instance.

```java
registry.addLazy(LastCities.class, new LazyServiceRegistry.LazySetter<LastCities>() {
      @Override
      public LastCities get() {
        return new LastCities(appContext);
      }
    });
```

Then we will access this object using the Async method of the LazyLoading class, once we get the object, we will update the UI directly there.

```java
DineApp.getRegistry().get(LastCities.class, new LazyServiceRegistry.Callback<LastCities>() {
  @Override
  public void onInstanceReceived(LastCities instance) {
    // update UI
  }
});
```

Yes it's a bit more code but once folded cmd + '-' on mac, it looks better.

```java
DineApp.getRegistry().get(LastCities.class, (instance) -> {
    // update UI
  });    
```

This service initialization is created at startup (App.onCreate). Once needed it's initialized in a background thread and updates the UI on the main Thread. Then if we still need to fasten the process, we can create a ```GSON``` custom type to avoid the costly reflection. 

## UIUtils.initReservation
### What's going on
This method creates people's default number, default reservation hours, default **day** available. It is used on the first ```Activity``` displayed. *I think that we can create those default objects by hand instead of parsing JSON data to avoid reflection costs when there are not many objects*

![UIUtils initReservation][callstack_oncreate_uiutils_joda]

```java
@NonNull
private static DateTime getSelectedDateTime(int days) {
    return DateTime.now().plusDays(days);
}
```

### How to patch
Let's take a look at this line, ```DateTime.now().plusDays(days)```. JodaTime is clearly blocking the MainThread for a bit less than 1 second. That is a huge impact. We should always take care of methods we are using. We always should do most of the work on a background thread to avoid this kind of unwanted behaviour. What can we do to fix the first call to DateTime.now() latency? First, why are we doing this... At the end of the method we can see this:

```java
defaultItems[0] = partySizes.get(2);//2 peoples
defaultItems[1] = daysAvailable.get(1);// today
defaultItems[2] = timesAvailable.get(3);// some hour near now
```

![Dine Main Screen][dine_main_screen]

At the time of writing this we are Sunday and it's 18h00. With very little changes, we could be avoid all this preloading all for default.
Party Size can always be **2** by default and the day available **today** no need for any big calculations on that. But how do we do for time available?

JodaTime is a pretty big library, usefull on some cases but we don't have to rely on it for everything. I have done this very scientific test on a new App's activity (runned the test 3 times with similar results on emulator).

```java
Log.d("APP", "StartTime Calendar")
var hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY); // ~ 0 to 1ms 
Log.d("APP", "StopTime Calendar")

Log.d("APP", "StartTime Joda")
var hour2 = DateTime.now().hourOfDay; // ~ 56 to 67ms
Log.d("APP", "StopTime Joda")
```

Do we really need to use JodaTime to get the hour of the day? My guess here is no so we can switch for the basic calendar and reduce the latency.

All the heavy calculations can be started on a thread to be ready when the user will click on the button and have near instant response.

# Conclusion
We have seen how to profile our startup time and extract the biggest problems. We had our hands on some code and how to modify it to get a better experience for the user. To finish we found that using a big library is not often the best tool for performance and we have to always think about the impact of our code on our user.  

As a side note, we should always keep track of the cold start, it's a good indicator about what is going on in the code and NimbleDroid offer this for free. This way we can keep an eye on builds after builds, check if we are correctly using new libraries and don't impact too much our users.

Have a great profiling day :)

Credits:
[Snow Flake photo](https://www.flickr.com/photos/chaoticmind75/8313222713) comes from [Alexey Kljatov](https://www.flickr.com/photos/chaoticmind75/), all rights reserved to the creator.


PS: for now I haven't any comments on my website, please send any comments/issues here https://github.com/ViBlog/shorten-cold-start

[comment]: <> (IMAGES)
[YPDine_logo]: images/ypdine_logo.webp
[YPDine_general]: images/dine_cold_startup.png
[YPDine_onCreate]: images/dine_callstack_onCreate.png
[callstack_oncreate_analcommander]: images/callstack_onCreate_analcommander.png
[callstack_oncreate_userpreference]: images/callstack_onCreate_userPreferences.png
[callstack_oncreate_uiutils_joda]: images/callstack_onCreate_UIUtils_initReservation.png
[dine_main_screen]: images/dine_main_screen.png


[comment]: <> (LINKS)
[Zygote]: http://cyrilmottier.com/2013/01/23/android-app-launching-made-gorgeous/
[report]: https://info.dynatrace.com/rs/compuware/images/Mobile_App_Survey_Report.pdf
[libraries]: https://github.com/codepath/android_guides/wiki/Must-Have-Libraries
[YP Dine]: https://play.google.com/store/apps/details?id=com.ypg.dine
[Show Java app]: https://play.google.com/store/apps/details?id=com.njlabs.showjava
[NimbleDroid]: https://nimbledroid.com/
[TraceView and DmTraceDump]: http://developer.android.com/tools/debugging/debugging-tracing.html
[gold plating]: http://blog.codinghorror.com/gold-plating/
[callstack_oncreate_link]: https://nimbledroid.com/play/com.ypg.dine?p=323DVxanEq1ssS#com.ypg.dine.DineApp.onCreate
[newyorktimes_improvingstartuptime]: http://open.blogs.nytimes.com/2016/02/11/improving-startup-time-in-the-nytimes-android-app/?_r=0
[gson_custom_types]: https://google-gson.googlecode.com/svn/trunk/gson/docs/javadocs/com/google/gson/TypeAdapter.html
