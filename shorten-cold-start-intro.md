Essay that will be published under [dubedout.eu](http://dubedout.eu)

# Cold Starts
## What's that?
In Android, there is three types of starts: First Starts, Cold Starts and Warm Starts.

- First starts, the slowest, is just after it have been installed or updated. The app needs to set up database, load configuration, load the first batch of data. It's only occurring once so we will not talk about it.

- Warm starts, the fastest, is when you put your app in background and went back to it. The app is still launched; all services are already up running. It's usually very fast.

- Cold starts, in between first and warm, is frequently occurring and the user will notice the slow start time *(I wasn't able to retrieve data, even if I remember seeing one)*. When the application is not loaded by the system, it has to relaunch all his objects, services, assets and that can be slow.

## Why it's so slow
In everyday work, we try to use the [best tools] to fasten the development and achieve the business requirements. Unfortunately, they are often lined up over features and don't take into account the benefits of a snappy application.

To fasten development, we use a lot of libraries that are needed across the whole application. To do so, they are often instantiated in the ```Application.onCreate()```. The problem comes when we start using too much without any threading. The onCreate is one of the first methods to be called when the application launches and if have any blocking code, we will impact directly the launch time.

> "I have no time to fix this and it's not so slow. [10 libraries later]... Mmmm I guess we have a problem." -*Derp*

## SplashScreen mistake
Lot of blog posts have been written about this -> cyrilMottier post
*** summarize ***

As of many Android developers, I'm a partisan of the ***Show the content fast*** rule. And for me, It's like modifying this well-known quote from Uncle Bob's Clean Code book
> "Every time you write a ~~comment~~ *SplashScreen*, you should grimace and feel the failure of your ability of ~~expression~~ *coding a snappy app*."

Today, instead of focusing on creating a nice startup with the windowBackground we will focus on tracking what is taking time at startup and reduce that time.

# Real world study
## What kind of App
We will study an application built in the company I work for: [YP Dine].
![Logo YP Dine][YPDine_logo]  


## With what
Usually I use [TraceView and DmTraceDump], their purpose is to find the bottlenecks in your code and help you fix it, but I keep them for a dedicated post. This time we will rely on [NimbleDroid], they use the same tools but display the results in a very easy to understand way. What I like is that they tests performances in same conditions for each applications. So it's easy to compare your application with other ones like Facebook, WhatsApp and other big players.

## Tracing
Watch out, before starting enhancing the startup time, you should check if it is worth your time, how long it should take and avoid [gold plating].  

![2.6s launch time][YPDine_general]  

Our app is not that bad with 2400 ms to start. Just below on the Hung Methods, we can see that the DineApp.onCreate() is blocking the process by 2059 ms.  

- clicking on it shows that Methods are taking most of the time
  - UInit
  - JodaTime
  - third one I don't remember by head

![onCreate 3 methods blocking startup][YPDine_onCreate]

to gain lot of launching time easily, we need




[comment]: <> (IMAGES)
[YPDine_logo]: images/ypdine_logo.webp
[YPDine_general]: images/dine_cold_startup.png
[YPDine_onCreate]: images/dine_callstack_onCreate.png

[comment]: <> (LINKS)
[best tools]:https://github.com/codepath/android_guides/wiki/Must-Have-Libraries
[YP Dine]:https://play.google.com/store/apps/details?id=com.ypg.dine
[NimbleDroid]:https://nimbledroid.com/
[TraceView and DmTraceDump]:http://developer.android.com/tools/debugging/debugging-tracing.html
[gold plating]:https://en.wikipedia.org/wiki/Gold_plating_(software_engineering&#41;





_________________________________________________
part 2


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
