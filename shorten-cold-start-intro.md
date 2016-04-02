Essay that will be published under [Profile cold starts on dubedout.eu](http://dubedout.eu/profile-cold-starts/)

# Cold Starts
## What's that?
In Android, there is three types of starts: First Starts, Cold Starts and Warm Starts.

- First starts, the slowest, is just after the application have been installed or updated. The app needs to (re)set up a database, load configuration, load the first batch of data. It's only occurring once, so it's not the most important.

- Warm starts, the fastest, is when you put your app in the background and go back to it. The app is still in memory, everything up and running, it's usually very fast.

- Cold starts, to finish, is between first and warm. The application has been released from the memory, and will have to initialize back every service, which it needs to run. It's frequently occurring, and it takes time to startup, the user will notice it, and will not like it as you can see in this 2012's [report].

## Why it's so slow
In everyday work, we use [libraries] to develop and achieve business requirements faster. Those focus on adding features instead of keeping an application quick and responsive because the latest is not adding business value, or not directly quantifiable.

A big part of used libraries are needed by the whole application and are often initialized in the ```Application.onCreate()```. When we initialize without any threading or lazy loading, we end up with a ```onCreate``` that takes more time to be processed. As it's one of the methods called each time the application start, and that an ```Activity``` can't be started until it's complete, we slow down every launch.

When the application is slow to start, it displays the [Zygote] longer, and it will break immersion. This root white start screen is not pretty by default... But instead of focusing on creating a beautiful app startup screen with the windowBackground we will concentrate on tracking what is taking time and reduce it.

# Profile YP Dine
[YP Dine] is a recent application of the play store built by another team in Yellow Pages Canada. His goal is to make you can discover your next favorite restaurant, browse through handmade lists from local food experts or book a table from this app. I will dig into the initialization to check if we can have quick wins.

## What tools
Usually, I use [TraceView and DmTraceDump] to find the bottlenecks in the code and fix it. But today we will play with [NimbleDroid]. They use the same tools but display the results in a very easy to understand way. All performance tests are realized in same conditions letting you compare with other applications, and you can check the cold start of your app by versions. Super nice to keep track of this starting time.
YP Dine isn't obfuscated so we can effortlessly check which part of the code is blocking. If yours is obfuscated, it's still possible to add the ProGuard Mapping to reveal problematic methods.  
On a side note, a colleague shared me [Show Java app] that can be usefull to see unobfuscated code apps.

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

Always keep in mind to work on the most important issues first, frames dropping on a ```ListView``` can be more frustrating that the start time. I like this post from coding horror on [Gold Plating], you should take a look if you don't know this.

# We should go deeper
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
It's an analytic tool that helps merging all analytics libraries in a single access point. As all analytic tools, it`s used everywhere in the application and to avoid recreating everytime, is created in the initialization of the application and stored in the ServiceRegistry (a Class/Instance map singleton).

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

But let's take a look on this line, ```DateTime.now().plusDays(days)```. It's blocking the UIThread for around 1 second. We always should do most of the work on a background thread to avoid this kind of unwanted behaviour.

# How to solve
## Analytics Commander
This one is used everywhere in the application but it only sends data and does not interacts with the UI. We don't have to wait for data to be returned from the API. In this case we can use a Lazy Loading library like Dagger. Then, in a thread, access to the instance and send the call. Doing that, the initialization will be async and once the object is created send the data without being tied to the UI.

## UserPreferences
Maybe the more complicated one. As it is tied to the first activity, we should split the data needed on the first page from the one needed later. Then it can be usefull to create a gson custom type to avoid the reflection. This service should be initialized the sooner (App.onCreate), in a thread, and accessed in the first Activity and update the UI once you get the data needed.

## UIUtils
It's a particular case, we don't need it right away but it should still be initialized in the onCreate but in a low priority thread. Doing this, it will be ready when needed on the second screen.

# SumUp


So we have Analytic Commander that takes time to initialize, used everywhere in the app but is just an api call and do not interact with the UI. We can launch it in a thread as a fire and forget mode.  
Then we have UserPrefences that are needed not long after the launch screen, that impacts UI. This one should be initialized first and we should have a way to access this data the sonner without blocking the UI.  

*** Image cpu time by dependencies ***

To finish, UIUtils is not needed right after the launch screen but impact the UI. This one can be initialized in a low priority thread at startup.


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
