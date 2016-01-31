---
layout: post
title:  "Inspect a UIWebView without the official Safari Web Inspector"
date:   2014-08-13 00:00:00 +0000
---

During the last months I've been busy to develop a JavaScript wrapper that allows to interact
with a third-party JavaScript library. The wrapper exposes a bunch of APIs that are invoked from the
native Objective-C code. The interaction is accomplished by means of
<code>-stringByEvaluatingJavaScriptFromString:</code>
method.
Since each <code>UIWebView</code> has its own JavaScript engine, this method is used to evaluate arbitrary scripts.
As a side note, since this method API is very limiting, starting with iOS 7 and OS X 10.9 you can take advantage of
JavaScript core
framework that, as described in the Apple documentation, provides Objective-C wrapper classes for many standard
JavaScript objects.
To debug HTML/JavaScript code loaded within a <code>UIWebView</code> you can take advantage of the official Safari
Web Inspector. This tool is available from iOS 6 and Safari 6 and it can be used both in simulator and real devices.
Amazing!

But wait a minute. What about if you want to debug iOS 5.x apps? The WebKit version shipped with iOS 5.x can differ
from the one included in iOS 6.x and later. Code that works well under iOS 7 could not work under iOS 5 and without
inspection, it could difficult to find a solution. I surfed the web and I read a lot of posts and blogs. Based on the readings
I found a very simple solution.
First of all you need to download Xcode version 4.3 with the iOS 5.1.1 simulator. Then, you need to grab a version of Chromium browser up to version 12.
As described in <a href="http://www.iwebinspector.com/help.html#ml" target="_blank">Why iWebInspector is not working in Mountain Lion (Mac OS 10.8)?</a>,
Safari 6 (and later) uses a socket version library that does not allow to communicate with a <code>UIWebView</code>.
Ok. The environment has been setup. Now open your Xcode project, locate the <code>application:didFinishLaunchingWithOptions:</code>
in your application delegate class and write the following:

<pre>
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

  [NSClassFromString(@"WebView") _enableRemoteInspector];

  // or [NSClassFromString(@"WebView") performSelector:@selector(_enableRemoteInspector)];
  // if the compiler will complain
}
</pre>

Run both Chromium and your app and navigate to the code responsible for displaying the HTML/JavaScript code.
Browsing to <a href="http://localhost:9999" target="_blank">http://localhost:9999</a> you will find an index page listing the URL of the web views currently displayed.
Inspect and debug as you prefer!

Notes for the reader. First, remember to remove the private <code>_enableRemoteInspector</code> if you release the app to the App Store. Second, If you don't want to run your entire project with Xcode 4.3, you could just create a dummy project that contains only the code necessary to load the HTML/JavaScript into your <code>UIWebView</code>.

Additional references: <a href="http://atnan.com/blog/2011/11/17/enabling-remote-debugging-via-private-apis-in-mobile-safari" target="_blank">Enabling Remote Debugging via Private APIs in Mobile Safari</a>,
<a href="http://www.iwebinspector.com" target="_blank">iWebInspector</a>
