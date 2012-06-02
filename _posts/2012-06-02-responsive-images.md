---
layout: post
title: Responsive images
image: ''
excerpt: |-
    Mobile phones don't need to download fullsize images. Although responsive images are still a work in progress, there's a quick way to save bandwidth and load small images for mobile phones.
---


### Mobile phones don't need to download fullsize images. Although responsive images are still a work in progress, there's a quick way to save bandwidth and load small images for mobile phones.


Mobile First
------------

__Mobile First__ is the approach proposed by [Luke Wroblewski](http://www.lukew.com/ "Luke Wroblewski") 
in this [article](http://www.lukew.com/ff/entry.asp?933 "article"). Ideally, we should start the design 
and development of mobile websites considering the mobile screen width, and progressively _adjusting_
our design to wider screens.

Instead of **gracefully degrading** our website for smaller screen, _Mobile First_ is about 
**gracefully enhancing** our website for larger screens.


Loading Images using Media Queries
----------------------------------


We can use the same approach also for images. Consider we want to display a small image for our
mobile website. Instead of starting with a full size image, we start with the mobile image.

Using **\<figure\>** _tag_ in combination with **Media Queries** it's possible to load different 
images depending on the screen width. As I've read somewhere, **first media query is no media query**.
Instead of using the typical **\<img\>** we can use  **\<figure\>** and setup a background image. 

    figure{
      background-image:url('/images/pixar-320.jpg');
      background-repeat:no-repeat;  
      width:320px;
      height:247px;
    }
    
<div class="center">
  <img src="http://responsive-images.herokuapp.com/images/pixar-320.jpg"/>
</div>
    
We are instructing the browser to load the _320px_ image, and that's the **default size** outside of
the media queries definition.

If the mobile is in **landscape** position, we can setup a media query to consider this case:

    /* Screen >= 480 */
    @media screen and (min-width:480px){
      body {
        width: 480px;
      }
      figure{
        background-image:url('/images/pixar-480.jpg');
        background-repeat:no-repeat;    
        width:100%;
        height:323px;
      }
    }

<div class="center">
    <img src="http://responsive-images.herokuapp.com/images/pixar-480.jpg"/>
</div>


We can also do the same for Tablets device:


    /* Screen >= 768 */
    @media screen and (min-width:768px){
        body {
          width: 768px;
        }
        figure{
          background-image:url('/images/pixar-768.jpg');
          background-repeat:no-repeat;
          width:100%;
          height:421px;
        }    
    }
    
    
And we could keep adding more media queries:

    /* Screen >= 960 */
    @media screen and (min-width:960px){    
        body {
          width:960px;
        }
        figure{
          background-image:url('/images/pixar-960.jpg');
          background-repeat:no-repeat;      
          width:100%;
          height:466px;
        }
    }

Where's the trick?
------------------

This is a proof of concept, but it works nicely for Android or iPhone/iPad devices.
If the user will load the page in portrait mode (thus 320px), the 480px image is loaded only 
by rotating the mobile in landscape position. 

When the mobile is already in landscape position, the mobile will load both images, 
320px and 480px. That's not completely bad, since users tend to swap between portrait 
and landscape most of the time.

The good news is that larger images (768px and 960px) won't be loaded in our mobiles, thus
saving bandwidth.

Here's a complete example with a [responsive images demo](http://responsive-images.herokuapp.com/ "responsive images demo")
and here's the [source code](https://github.com/rosario/responsive-images "source code")
