Essay that will be published under [shorten cold start on dubedout.eu](http://dubedout.eu/shorten-cold-start-intro/)

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

# Hands on YP Dine
[YP Dine] is a recent application of the play store built by another team in Yellow Pages Canada. His goal is to make you can discover your next favorite restaurant, browse through handmade lists from local food experts or book a table from this app. I will dig into the initialization to check if we can have quick wins.

## What tools
Usually, I use [TraceView and DmTraceDump] to find the bottlenecks in the code and fix it. I will keep them for a dedicated post and will play with [NimbleDroid]. They use the same tools but display the results in a very easy to understand way. All performance tests are realized in same conditions letting you compare with other applications, and you can check the cold start of your app by versions. Super nice to keep track of this starting time.
YP Dine isn't obfuscated so we can effortlessly check which part of the code is blocking. If yours is obfuscated, it's still possible to add the ProGuard Mapping to reveal problematic methods.

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

We end up the first part here, on the second part, we will go through the code of the ```Application.onCreate()```, check what is going on, probably refactor, test with basic threading, replace the ```ServiceRegistry``` by a fast Lazy Loading class.

***Part 2: Dive into the code***

Credits:
[Snow Flake photo](https://www.flickr.com/photos/chaoticmind75/8313222713) comes from [Alexey Kljatov](https://www.flickr.com/photos/chaoticmind75/), all rights reserved to the creator.


PS: for now I haven't any comments on my website, please send any comments/issues here https://github.com/ViBlog/shorten-cold-start

[comment]: <> (IMAGES)
[YPDine_logo]: images/ypdine_logo.webp
[YPDine_general]: images/dine_cold_startup.png
[YPDine_onCreate]: images/dine_callstack_onCreate.png

[comment]: <> (LINKS)
[splashScreen are evil]:http://www.cyrilmottier.com/2012/05/03/splash-screens-are-evil-dont-use-them/
[Zygote]:http://cyrilmottier.com/2013/01/23/android-app-launching-made-gorgeous/
[report]:https://info.dynatrace.com/rs/compuware/images/Mobile_App_Survey_Report.pdf
[libraries]:https://github.com/codepath/android_guides/wiki/Must-Have-Libraries
[YP Dine]:https://play.google.com/store/apps/details?id=com.ypg.dine
[NimbleDroid]:https://nimbledroid.com/
[TraceView and DmTraceDump]:http://developer.android.com/tools/debugging/debugging-tracing.html
[gold plating]:http://blog.codinghorror.com/gold-plating/;
