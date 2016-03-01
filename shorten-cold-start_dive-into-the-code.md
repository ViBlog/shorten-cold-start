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
