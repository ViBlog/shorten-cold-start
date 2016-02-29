Essay that will be published under [dubedout.eu](http://dubedout.eu)

# Cold Starts
## What's that?
In Android, there is three types of starts: First Starts, Cold Starts and Warm Starts.

- First starts, the slowest, is just after the application have been installed or updated. The app needs to (re)set up database, load configuration, load the first batch of data. It's only occurring once so it's not the more important.

- Warm starts, the fastest, is when you put your app in background and go back to it. The app is still in memory, everything up and running, it's usually very fast.

- Cold starts, to finish, is between first and warm. Application have been released from the memory, and will have to initialize back every services, that it needs to run. It's frequently occurring and if takes a lot of time to startup, the user will notice it, and will not like it as you can see in this 2012's [report].

## Why it's so slow
In everyday work, we use [libraries] to develop and achieve business requirements faster. Those are often focused on adding features instead of keeping an application fast and responsive because the latests is not adding business value. or not directly.

A lot of Libraries are needed across the whole application and are often initialized in the ```Application.onCreate()```. When we initialize too much without any threading or lazy loading, we end up with an ```onCreate``` that takes more time to pass through. As it's one of the methods called each time the application is launched, and that an ```Activity``` can't be started until it's completed, we slow down every launch.

> "Oops!... I did it again" -*Britney Spears*

That's when the management comes with a great idea.

## Splash Screens
As many Android developers, I'm a supporter of showing the data the user is looking for the sooner, the better. I like joking modifying this well-known quote from Uncle Bob's Clean Code book

> "Every time you write a ~~comment~~ *SplashScreen*, you should grimace and feel the failure of your ability of ~~expression~~ *coding a snappy app*."

I will not repeat the well-known blog post of Cyril Mottier ([splashScreen are evil]) where he explains the reason why you should avoid it.

Today, instead of focusing on creating a nice startup screeb with the windowBackground we will focus on tracking what is taking time at startup and reduce it.

# Hands on
## YP Dine
[YP Dine] is a recent application on the play store built by Yellow Pages Canada.

`With YP Dine, discover your next favourite restaurant! Browse through handmade lists from trusted local food experts, take a peek at the menu, book a table and get directions – all in one convenient app.
YP Dine is packed with exciting features:

• Choose from curated lists customized to your location, the kind of food you’re craving, the time of day and the type of occasion
• Access restaurant details including menus, address, phone number, opening hours, maps, and more.
• Book a table right in the app`


## What tools
Usually I use [TraceView and DmTraceDump], their purpose is to find the bottlenecks in your code and help you fix it, but I will keep them for a dedicated post. This time we will rely on [NimbleDroid], they use the same tools but display the results in a very easy to understand way. All performance tests are realized in same conditions letting you compare with other applications, and you can check the cold start of your app by versions.
YP Dine is not obfuscated so we can easily check which part of the code is blocking. If yours is obfuscated, it's still possible to add the ProGuard Mapping to reveal the problematic methods.

## Tracing

Yp Dine is not a bad player here with 2400 ms to start. On the other hand, we can see below that the DineApp.onCreate() is blocking the process by 2059 ms. 80% of his start time is blocked there, let's see if we can find something more ```detailed``` by clicking on it.

![2.6s launch time][YPDine_general]  

On the call stack, we can see that three methods are taking most of the starting time:
- UIUtils.initReservation (JodaTime), 981ms
- UserPreferences, 553ms
- YpDine.Utils..., 364ms

![onCreate 3 methods blocking startup][YPDine_onCreate]

Those methods looks to be in the main thread, if we are able to instantiate them on a thread, we should be able to quickly win some time here.

Always keep in mind to ***fix performance issues*** on the most blocking issues, don't do this for the sake of doing it. [Gold plating], time that could have been on more higher priority part of code.

*** PART 2 : Dive into the code***


[comment]: <> (IMAGES)
[YPDine_logo]: images/ypdine_logo.webp
[YPDine_general]: images/dine_cold_startup.png
[YPDine_onCreate]: images/dine_callstack_onCreate.png

[comment]: <> (LINKS)
[splashScreen are evil]:http://www.cyrilmottier.com/2012/05/03/splash-screens-are-evil-dont-use-them/
[report]:https://info.dynatrace.com/rs/compuware/images/Mobile_App_Survey_Report.pdf
[libraries]:https://github.com/codepath/android_guides/wiki/Must-Have-Libraries
[YP Dine]:https://play.google.com/store/apps/details?id=com.ypg.dine
[NimbleDroid]:https://nimbledroid.com/
[TraceView and DmTraceDump]:http://developer.android.com/tools/debugging/debugging-tracing.html
[gold plating]:https://en.wikipedia.org/wiki/Gold_plating_(software_engineering&#41;





_________________________________________________
part 2 : Dive into the code

## Dive into the code
Always keep in mind to ***fix performance issues*** on the most blocking issues, don't do this for the sake of doing it. [Gold plating], time that could have been on more higher priority part of code.



*** Library + no threads = errors ***

- User not able to use it fast
- User loose interest in launching it




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
