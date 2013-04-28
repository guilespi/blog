---
layout: post
title: "Can you make a native app using Phonegap?"
date: 2013-02-10 21:02
comments: true
categories: 
- Phonegap
- Mobile
- Programming
- Javascript
---

I've been looking at Phonegap since it started, long before [Adobe bought it][12] in a desperate attempt to destroy it.

Up until now, I never got the chance to make something with it, but always had the doubt how good or bad the applications you could create were.

Last week I had an opportunity to take a look under the hood, and made the source code available [on GitHub][10].

The big question was, can you make native apps using Phonegap?

** What you mean by native? **

According to [this guy][1] in order to make native IOS applications you need to program in Objective-C.

I say that's misleading.

Native applications are applications that run on the phone and provide a `native experience`, for which I understand at least the following treats:

* Can access all of the device API, address book, camera, etc.
* Access to local storage
* Zero latency feedback
* Interoperability with other phone applications
* UX should respect the device culture and guidelines
,
If you have those, why should you care about the language the app is written in?

** The wrong reason to use Phonegap **

At first sight you may think that the main reason to use Phonegap is program once, run everywhere.

Wrong.

In order to provide a native experience you will need to design the UX of your app for every platform you're targeting, so at least the UX/UI code will be different.

Obviously you *can* use the same UI for all the platforms, but unless the purpose of your app is to alienate your users I wouldn't try it.

Software should behave as the user expects it to behave, you would not create new affordances for the sake of creativity, don't do it for the sake of saving money, 
because it ain't cheaper in the long run.

So, no matter what you're thinking about doing, save some time to read the UX/UI guidelines for each mobile platform you're targeting.

The great [Mike Lee][13] would tell you that you even need a different team for each of those platforms.

** WTF is Phonegap? **

You know the tagline *"easily create apps using web technology you love"*, does it mean the only thing you need to know is HTML and Javascript? 

Of course not.

Phonegap is an extensible framework for creating web applications, with the following properties:

* The framework exposes an API to access the device different components
  + Accelerometer
  + Camera
  + Compass
  + Contacts
  + Etc.
* The API is the same for the different supported platforms
  * IOS 4.2+
  * Android 2.1+
  * Blackberry OS6+
* You must code your program using HTML and Javascript.

You can think of it as a native host that lets you write your application in Javascript, abstracting the native layer components behind a uniform API. That's all.

So you'll end up creating your app inside XCode, dealing with code signing nightmares and taking a lot of walks inside the Apple walled gardens.

{% img center /images/blog/phonegapide.png 680 780 'Phonegap Application Inside XCODE' %} 

And you _will_ need to learn the Phonegap API.

** It doesn't have to be 100% HTML **

The first reaction is to think that since Phonegap uses a webview you will have to create your application using only HTML, but it's not the case.

Phonegap supports [plugins][3], which is a mechanism for exposing different native features not already exposed in Phonegap API. 
So you can create more native plugins and expose them to javascript, where javascript works as the *glue* that blends 
together the native controls but not necessarily is used to create all the UI.

The most common example is the TabBar and NavigationBar in IOS, plugins for those already exist, and lets you design a more native experience than the one 
you would get using only HTML.

{% img center /images/blog/phonegapplugins.png 280 480 'Phonegap TabBar and NavigationBar Plugins' %} 

Notice the `Back` button in there, it's just a custom button with the back text. If you want to create an arrow like back button you'll need to go
down the [same path][11] as if you were doing Objective-C IOS development.

** Mobile Frameworks **

There are many mobile frameworks available [out there to be compared][4].

Among the most well known are [jQuery Mobile][5] and [Sencha Touch][6]. JQM development being more *web like*, something to consider if your team is already 
comfortable with HTML, Sencha generates its own DOM based on Javascript objects 100% programatically.

I haven't dig deep enough in order to write a complete evaluation, you may found some interesting ones [here][7], [here][8] and [here][9].

Almost everybody agrees in one important point:

JQM is sluggish and transitions doesn't feel native enough, something I easily verified testing the app in my IPad I, even the `slider` was sluggish.

** Using Phonegap Plugins **

Plugins are usually composed of two parts:

* The native bridge between Objective-C and Javascript
* The Javascript exposing the plugin

Usually you'll need to copy the `m` and `h` plugin files to the `Plugins` directory of your project, you will also need to declare the plugins 
being used in the `config.xml` project file.

```xml
 <plugins>
   ...  
  <plugin name="TabBar" value="TabBar"/>
  <plugin name="NavigationBar" value="NavigationBar"/>
  ...
 </plugins>
```

And then include the Javascript files in your `index.html`.

```javascript
 <script type="text/javascript" src="js/TabBar.js"></script>
 <script type="text/javascript" src="js/NavigationBar.js"></script>
```

Using the native plugins from your applications is straightforward, you initialize, create, setup and show the buttons you want.

```javascript

  plugins.navigationBar.init();
  plugins.tabBar.init();
     
  plugins.navigationBar.create();
  plugins.tabBar.create()
        
  plugins.navigationBar.setTitle("Navigation Bar")
  plugins.navigationBar.showLeftButton()
  plugins.navigationBar.showRightButton()
  plugins.navigationBar.setupLeftButton("Back",
                                        "",
                                        function() {
                                          $(window).unbind("scrollstop");
                                          history.back();
                                          return false;
                                        });
  plugins.navigationBar.setupRightButton(
                                         "Alert",
                                         "barButton:Bookmarks",
                                         function() {
                                          alert("right nav button tapped")
                                         });
  plugins.navigationBar.show()
        
  plugins.tabBar.createItem("contacts", "", "tabButton:Contacts", {onSelect: app.loadNews});
  plugins.tabBar.createItem("recents", "", "tabButton:Recents")
  plugins.tabBar.createItem("another", "Branches", "www/images/map-marker.png", 
                                        {
                                          onSelect: function() {
                                                            app.loadMap();
                                                        }});
  plugins.tabBar.show()
  plugins.tabBar.showItems("contacts", "recents", "another")

```

Using this strategy lets you extend your app for a more native experience exposing to javascript even custom controls you may design. 
This way you can have some members of your team focused on the native code of the app and exposing the building blocks to the web developers *assembling* the pieces.

{% img center /images/blog/phonegapplugincomparison.png 280 480 'Phonegap TabBar comparison against HTML' %} 

As you see in the last image, the TabBar is shown in the native version and the HTML version side-by-side. The HTML version was created using jQuery Mobile.

** Debugging is hell **

Well maybe it's not hell, but it's not a pleasant experience either. 

If you include the following line in your html using your own id:

```javascript
<script src="http://debug.phonegap.com/target/target-script-min.js#guilespi"></script>
```

You'll have easy access to debug your app using [weinre][15] without the need to set it up, at least it's good for HTML inspection.

{% img center /images/blog/weinredebug.png 680 780 'Weinre Debugging inside Chrome' %} 

If you want to debug javascript, you'll certainly end up using `alert` and `console.log`, even [the guys at Phonegap][16] are recommending the poor's man debugger.

Be ready to waste some of the time you gained by choosing Javascript doing print based debugging.

**Update**
_Raymond Camden pointed out in the comments that [a better approach to debugging exists][17], specially with Safari and IOS6_

** Conclusion **

Tools are picked for the team, so that's what you should think about when choosing or not to pursue the Phonegap path. If you already have members on your team 
who are great at web development, Phonegap may be an option, certainly it's fast for prototyping and seems to be a great asset for product discovery and validation.

If charging for your app is among your goals, I wouldn't pick Phonegap or any other framework that uses the webview renderer as the main application. 
Also, for most tasks the Javascript VM would be alright, but if you have inner loops cpu intensive, such as in game development, using Phonegap it's not really an option.

Reviewing the main points considered to categorize a mobile application as native, web frameworks will provide you a sub-par experience regarding feedback, 
latency and usability. Using the Phonegap plugins to avoid it will only go so far before the cost being so high you'll better be programming in Java or Objective-C anyway.

If you still have doubts, [fork the code][10] and give yourself a try.

_And don't forget to follow me on [twitter][14]_

[1]: http://stackoverflow.com/questions/4656475/iphone-native-application-vs-web-application
[2]: http://stackoverflow.com/questions/11251106/phonegap-minimal-platform-versions-supported
[3]: https://github.com/phonegap/phonegap-plugins
[4]: http://www.markus-falk.com/mobile-frameworks-comparison-chart/
[5]: http://jquerymobile.com/
[6]: http://www.sencha.com/products/touch/
[7]: http://altabel.wordpress.com/2012/12/14/mobile-web-app-frameworks-review-sencha-touch-vs-jquery-mobile/
[8]: http://www.collabera.com/blog/default/2012/12/12/1355310199068.html
[9]: http://blog.roveb.com/post/17259708005/our-experience-with-jquery-mobile-and-sencha-touch
[10]: https://github.com/guilespi/phonegap-ios-plugins
[11]: http://stackoverflow.com/questions/227078/creating-a-left-arrow-button-like-uinavigationbars-back-style-on-a-uitoolba
[12]: http://news.cnet.com/8301-30685_3-20114857-264/adobe-buys-phonegap-typekit-for-better-web-tools/
[13]: http://www.infoq.com/presentations/Product-Engineering
[14]: http://www.twitter.com/guilespi
[15]: http://people.apache.org/~pmuellr/weinre/docs/latest/
[16]: http://phonegap.com/2011/05/18/debugging-phonegap-javascript/
[17]: http://www.raymondcamden.com/index.cfm/2013/1/21/Did-you-know--Safari-Remote-Debugging-and-PhoneGap
