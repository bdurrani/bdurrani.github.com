---
layout: post
title:  "The trouble with panels"
date:   2014-01-25 
comments: false
tags: jquerymobile js
---

I've recently started playing with [jQuery Mobile][jqm]. And as some with very little
experience with modern HTML/js/CSS, I found it overwhelming.

v1.4 was released recently and while the actual libraries were published, the documentation left must to be
desired. Granted I might have been trying an advanced use case that might not have been very well tested.

My goal was to make use of their most excellant [panel] widget. I wanted to recreate the navigation tree as seen
in their demo page. I have a bunch of pages on my site that I set up using their [single-page template] as a 
starting point.

{% highlight html %}
<!DOCTYPE html>
<html>
<head>
    <title>Page Title</title> 
    <meta name="viewport" content="width=device-width, initial-scale=1"> 
    <link rel="stylesheet" href="http://code.jquery.com/mobile/1.4.0/jquery.mobile-1.4.0.min.css" />
    <script src="http://code.jquery.com/jquery-1.9.1.min.js"/>
    <script src="http://code.jquery.com/mobile/1.4.0/jquery.mobile-1.4.0.min.js"/>
</head>
<body>
<div data-role="page">

    <div data-role="header">
        <h1>Page Title</h1>
    </div><!-- /header -->

    <div role="main" class="ui-content">
        <p>Page content goes here.</p>
    </div><!-- /content -->

    <div data-role="footer">
        <h4>Page Footer</h4>
    </div><!-- /footer -->
</div><!-- /page --> 
</body>
</html>
{% endhighlight %}
The next thing I needed to use was an [external panels]. External panels are pretty great if you have a panel
that needs to show up across multiple pages. You declare it outside the page, but before the body so it lives outside
the DOM of the page being displayed.

A limitation I ran into was trying to use external panels that have collapsible list items. Turns out, this panel 
will loose all of its formatting.

{% highlight html %}
<!DOCTYPE html> 
<html>
<head>
    <title>Page Title one</title>
    <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
    <link rel="stylesheet" href="http://code.jquery.com/mobile/1.4.0/jquery.mobile-1.4.0.min.css" />
<script src="http://code.jquery.com/jquery-1.9.1.min.js"></script>
<script src="http://code.jquery.com/mobile/1.4.0/jquery.mobile-1.4.0.min.js"></script><script id="panel-init">
        $(function() {
            $( "body>[data-role='panel']" ).panel().enhanceWithin();
        });
    </script>
</head> 
<body>
    <div data-role="page">

    <div data-role="header">
        <h1>Page Title one</h1>
    </div><!-- /header -->

    <div role="main" class="ui-content">
        <p>Page one content goes here.</p>
        <p><a href="two.html">Navigate to page 2</a></p>
        <a href="#leftpanel3" class="ui-btn ui-shadow ui-corner-all ui-btn-inline ui-mini">Overlay</a>
    </div><!-- /content -->

    <div data-role="footer">
        <h4>Page Footer</h4>
    </div><!-- /footer -->

</div><!-- /page --> 
<!-- external panel starts here -->
     <div data-role="panel" id="leftpanel3" data-position="left" data-display="overlay" data-theme="a"> 
        <ul data-role="listview">
            <li data-icon="delete"><a href="#" data-rel="close">Close</a></li>
            <li data-icon="back"><a href="#demo-intro" data-rel="back">Demo intro</a></li>
            <li data-role="list-divider">Categories</li>

            <li data-role="collapsible" data-inset="false" data-iconpos="right"> 
              <h3>Bikes</h3> 
              <ul data-role="listview">
                <li><a href="#">Road</a></li>
                <li><a href="#">ATB</a></li>
                <li><a href="#">Fixed Gear</a></li>
                <li><a href="#">Cruiser</a></li>
              </ul> 
            </li><!-- /collapsible -->
            </ul>
        </div>
</body>
</html>
{% endhighlight %}

To have it look like a jquery panel, you will need to call `enhanceWithin()` as part of the initialization.
I found that part via the magic of stackoverflow.

After than, you need a bit of tweaking to get the collapsible listviews to look exactly right.

[Here][fiddle] is the jsfiddle for the final result. The CSS is from the jquery mobile sample [panel style] sample.

In the end, I stayed away from external panels. I ended up creating a Grunt-based workflow that would
add the panel html into all the pages I was using, just to be on the safe side.


[jqm]:    http://jquerymobile.com/
[panel]: http://demos.jquerymobile.com/1.4.0/panel/
[single-page template]: http://demos.jquerymobile.com/1.4.0/pages-single-page/
[external panel]: http://demos.jquerymobile.com/1.4.0/panel-external/
[fiddle]: http://jsfiddle.net/bilal33/BSJQd/4/
[panel style]: http://demos.jquerymobile.com/1.4.0/panel-styling/