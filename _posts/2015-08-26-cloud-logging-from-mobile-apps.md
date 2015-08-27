---
layout: post
title: Cloud Logging from mobile apps
comments: True
---

During development of a mobile app you have full control over what's happening in your code.
You have the debugger, you have console logging, performance profiling etc.
However, **after you have released your app, you have almost no tools for checking how your app
behaves in the wild!** In most cases, you get your feedback from emails to your
support address, App Store reviews and crash reporting (if you included Crashlytics or some similar tool in your app).

Crash reporting services are great.
But, there is one obvious limitation with crash reporting: your app needs to crash before any information
is sent to you.

<div class="message">
Wouldn't it be great if you could keep screening the log printouts that you already have in your app, also
after your app has been released?
</div>

 The problem is, how can you access the logs and how can you find useful information
in the vast amount of logs that your users will create?

## Enter... Cloud Logging services

***

Cloud logging is basically services that will capture logs from all kinds of sources and applications. These logs
will then be indexed for searching. There is normally a web UI that gives you advanced tools for searching and analyzing the log data.

There are many players in the cloud logging market: [Splunk](http://www.splunk.com/), [Sumo Logic](https://www.sumologic.com/), [Loggly](https://www.loggly.com/) etc. Some of these are free up to a certain level of data volumes and retention periods. There are also open source log management systems that work the same way, but you will need to host the servers yourself.
An example of this is [Logstash](https://www.elastic.co/products/logstash) in combination with [Kibana](https://www.elastic.co/products/kibana) or [Graphite](http://graphite.wikidot.com/).

For some reason, **Cloud logging services have mainly been used for replicating server logs.**
This is of course extremely useful, because you can do aggregated searches over several server logs, set up automatic alarms and do all kinds
of log analysis on all logs from all your servers.

**In the server case though, you already have the log data somewhere.** You can always read your logs on your individual servers if you really need to. **For apps, you have nothing!** You simply don't have the log source! So the incentive for cloud logging is much greater when it comes to
apps. For some reason though, it doesn't seem common to use cloud logging for apps.

A little over a year ago, I became interested in using one of the cloud logging services for my iPhone apps. To my surprise, there were
no iOS client libs available from any of the logging services themselves. I couldn't even find any open sourced libs on Github.
I decided to write an Objective-C library, [LogglyLogger-CocoaLumberjack](https://github.com/melke/LogglyLogger-CocoaLumberjack), which extends the iOS logging framework [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack).
I chose to integrate against Loggly, because they have an excellent REST API for remote logging. For Objective-C projects, I would definitely recommend using CocoaLumberjack.

When Swift came along, it turned out that CocoaLumberjack didn't work well in Swift projects, and at the time of writing this article, there are still some issues with using CocoaLumberjack from Swift. Also, I wanted a much smaller framework, so I wrote a minimalistic logging framework in Swift called [SlimLogger](https://github.com/melke/SlimLogger), that can be used for console logging, for Loggly logging, or for both.

<div class="message">
If you prefer another cloud logging service than Loggly, please contribute to SlimLogger by adding a class, `SlimYourFavoriteServiceDestination` that implements the SlimLogger protocol `LogDestination`. You can have a look in `SlimLogglyDestination.swift` to see how it can be done.
</div>

***

Here, I will describe how to do cloud logging to Loggly using SlimLogger in a Swift iOS app.

## Prerequisites

  * Create an account at https://www.loggly.com

## Installation

  * git clone https://github.com/melke/SlimLogger
  * Add `SlimLogger.swift`, `SlimLogglyDestination.swift`, `SlimLoggerConfig.template` and `SlimLoggerDestinationConfig.template` to your project
  * Rename `SlimLoggerConfig.template` to `SlimLoggerConfig.swift`
  * Rename `SlimLogglyDestinationConfig.template` to `SlimLogglyDestinationConfig.swift`

## Configuration

Edit the SlimConfig struct in `SlimLoggerConfig.swift`

{% highlight swift %}
struct SlimConfig {
    // Enable or disable console logging. When releasing your app, you should set this to false.
    static let enableConsoleLogging = true

    // Log level for console logging, can be set during runtime
    static var consoleLogLevel = LogLevel.trace

    // Either let all logging through, or specify a list of enabled source files.
    // So, either let all files log:
    static let sourceFilesThatShouldLog:SourceFilesThatShouldLog = .All
    // Or let specific files log:
    static let sourceFilesThatShouldLog:SourceFilesThatShouldLog = .EnabledSourceFiles([
             "AppDelegate.swift",
             "AnotherSourceFile.swift"
    ])
    // Or don't let any class log (use to turn off all logging to for all destinations):
    static let sourceFilesThatShouldLog:SourceFilesThatShouldLog = .None
}
{% endhighlight %}

Edit `SlimLogglyDestinationConfig.swift` and change the api key and app name in the Loggly URL.

{% highlight swift %}
struct SlimLogglyConfig {
    // Replace your-loggly-api-key below with a "Customer Token" (you can create a customer token in the Loggly UI)
    // Replace your-app-name below with a short name for your app (no spaces or crazy characters).
    // You can use this tag in the Loggly UI to create Source Group for each app you have in Loggly.
    static let logglyUrlString = "https://logs-01.loggly.com/bulk/your-loggly-api-key/tag/your-app-name/"

    // Number of log entries in buffer before posting entries to Loggly. Entries will also be posted when the user
    // exits the app.
    static let maxEntriesInBuffer = 100

    // Loglevel for the Loggly destination. Can be set to another level during runtime
    static var logglyLogLevel = LogLevel.info
}
{% endhighlight %}

## Usage

In `didFinishLaunchingWithOptions` in your app delegate, add the Loggly destination:

{% highlight swift %}
Slim.addLogDestination(SlimLogglyDestination())
{% endhighlight %}


Now you can start logging from any class.

{% highlight swift %}
// I recommend logging something that can be casted to an NSDictionary
// This way, searchable fields will be created in Loggly for every json key.
Slim.trace(["logtype": "user-interaction", "tapped": "Remove-item-button", "longmsg": "Removing item number \(itemnumber)"])
Slim.debug(["logtype": "state", "state": "\(state)"])

// But you can also log any type that implements the Swift protocol Printable
Slim.trace("my message")
Slim.debug(4711)
Slim.info(["one","two","three"])
Slim.info(someNSURLvariable)
{% endhighlight %}


That's all there is to it. The log posts will include your log message plus some standard fields that SlimLogger will add automatically:

  - **level** - The log level
  - **timestamp** - Timestamp in iso8601 format (required by Loggly)
  - **sourcelocation** - Source file and line number in that file (nice for doing facet searches in Loggly)
  - **appname** - The name of your app
  - **appversion** - The version of your app
  - **devicemodel** - The device model
  - **devicename** - The device name
  - **lang** - The primary lang the app user has selected in Settings on the device
  - **osversion** - the iOS version
  - **rawmsg** - The log message that you sent, unparsed. This is also where simple non-JSON log messages will show up.
  - **sessionid** - A generated random id, to let you search in loggly for log statements from the same session.
  - **userid** - A userid string. Note, you must set this userid yourself in the SlimLogglyDestination object. No default value.

## Structured log messages

You can log any type that implements Printable, but if you want a better search and analysis experience in the Loggly UI, **you should log a type that can be casted to an NSDictionary. If you do that, all dictionary keys will be logged as separate fields
in Loggly. This makes it much easier to do filtered and faceted field searches in Loggly.**
Word of warning, don't use too many different keys, it will make it harder to get a good overlook of your data
in the Loggly UI. Figure out smart keys that you can reuse in many of your log statements.

You should really think carefully about which fields you want to log with each request. Note that you don't have to be totally consistent, logging
the same fields for every log statement. If some fields don't make sense for a certain log message, just omit them.

### Example

Let's say that you want to log three types of log messages:

   * user interactions
   * user state
   * Bad stuff, warnings and errors

These types will require different kinds of fields. An error has an error object that you can log, a warning may not have an error object, and
user state and interactions serve other purposes altogether.

{% highlight swift %}
// User interaction
Slim.trace(["logtype": "user-interaction", "tapped": "Remove-item-button", "longmsg": "Removing item number \(itemnumber)"])

// State
Slim.debug(["logtype": "state", "state": "\(state)"])

// Bad stuff
Slim.warn(["logtype": "badstuff", "shortmsg": "No items", "longmsg": "No items for user \(username)"])
Slim.error(["logtype": "badstuff", "shortmsg": "User session timed out", "longmsg": "\(error)"])
{% endhighlight %}

As you can see, the field **logtype** exists in all log statements. Use this field as a filter when you
don't want to mix the different log types in the search results. There are also **shortmsg** and **longmsg** fields in some log statements. These are good when you want to see the sum of certain errors. **The shortmsg should only be unique enough to identify the error. The longmsg can contain unique
data for that particular log statement.**

##Use cases


###Find common errors
You can do aggregated searches in Loggly where you can see the **most common errors, warnings etc.**

###Alarms

In most cloud logging services, including Loggly, you can **setup alarms for lots of different things.** The most simple alarm condition would be if
the number of errors during a certain period of time rises above zero, or some other acceptable level of error count. You can also
be more specific, for example raising alarms whenever an In-App Purchase fails in your app.  

### Troubleshoot a session or a user

First of all, be very careful when logging anything that can be connected to a certain user. You always have to respect
the users privacy. Also, in some countries it is illegal to store such info altogether.

If you decide to track specific users you can **set the userid property on the SlimLogglyDestination object. The userid
will then be included in every log statement** until the app is terminated by iOS.

Let's say that a user complains about having problems in your app. You can then **search the Loggly UI for all log entries
that this user has created.** You can also have some secret button in your app, and when the user taps this
button, you can set the log level dynamically to a finer level in SlimLogglyConfig. If you don't want the secret button, **you
could add a way in your backend to change the loglevel for a specific user. The app would react to this backend change and
set it's loglevel in the SlimLogglyConfig dynamically.** Pretty nice, huh?

If you want your logs to be anonymized, you could use the **sessionid** that SlimLogger generates. This is useful when you want
to **see all log messages from a certain session.** It could also be combined with a secret button where the user can see the sessionid and
send it to the person troubleshooting the issue.

### Graphing errors (and non-errors)

In most cloud services, including Loggly, you can create graphs in the web UI for calculated numerical values. For example, you can set up a **graph that shows you the number of errors per hour.** You can also explicitly log numerical values in a dictionary field.
**Whenever you log a JSON number data type to Loggly, you can create a graph for it in the UI.** A useful example is to log different kinds of **response times** in your app.

If you also log app starts from your app, you can **combine app starts, response times and number of errors** as separate lines in the same graph, to see how they correlate.


## Summary

There is nothing worse than getting error reports from the users of your released app, but not being able to reproduce the error and not having enough information available to troubleshoot the error. Adding cloud logging for your app solves this problem. In fact, logging from apps may be the most appealing use case for cloud logging. I use it in my apps:

   * [**Baby Names Mania**](https://itunes.apple.com/app/baby-names-mania/id594784300?mt=8). Gives you loads of name suggestions for your baby.
   * [**GTDT - Get Things Done Today**](https://itunes.apple.com/app/gtdt-get-things-done-today/id977201254?mt=8). A pomodoro app that introduces a somewhat [**new philosophy for personal efficiency**](http://gtdt.melke.nu/).

Doing cloud logging in my apps has helped me to track down several issues that would otherwise have been hard to fix.
