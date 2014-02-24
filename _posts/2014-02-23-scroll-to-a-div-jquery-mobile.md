---
layout: post
title:  "Scroll to a div using Jquery Mobile"
---

Recently, I had a project where I was using Jquery Mobile within a UIWebview control.
I had to implement linking to an id on another page (i.e. href=#topic ) while using jquery Mobile (jqm). 
The documentation warns you that this will not work out of the box [because of the way the system navigates between pages][links].
There are
So you have to do a little bit of extra work to get that bit of functionality to work. 

This is not the only way to solve this ofcourse.
This problem is a very common one and you can find a lot of references to it on google. 
I got to this solution after referencing a lot of solutions on stackoverflow, with some tweaks.

{% highlight js %}

$(window).load(function() {    
    scrollToDiv();
});

function scrollToDiv() {
  var location = window.location;
  //get the hash associated with the url. We want to
    //scroll to the div with that id.
    var hash = location.hash;

  if ($(hash).length) {
    //found the id. 
      $('html,body').animate({
          scrollTop: $(hash).offset().top
      }); 
  }
}

{% endhighlight %}

The key was animating the scroll. Quite a few solutions referenced the [silent scroll][silentscroll]
function, but for some reason, The page would get scrolled down the part I was interested in only momentarily.
After that, the page would scroll back to the top.
Animating the scroll was the only way I got it to stick.

Also, setting the ```data-ajax="false"``` attribute to the href that was doing the linking did not seem to make
much of a difference. I don't know if it was because I was using UIWebview or something else changed within jqm, 
but ultimately, this little bit of js code things to work.



[links]:http://demos.jquerymobile.com/1.4.1/pages/
[silentscroll]: https://api.jquerymobile.com/jQuery.mobile.silentScroll/

