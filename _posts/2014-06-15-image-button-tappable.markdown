---
layout: post
title:  "Making an UIButton’s image tappable"
date:   2014-06-15 00:00:00 +0000
---

In these days I'm developing again in Objective-C and yes, I have to admit, I'm a bit rusty.
Said this, the requirement I received is to create a sort of navigation bar where items
are <code>UIButtons</code>. Each button is of type <code>UIButtonTypeCustom</code> and it has a title and an image.
Ok. Easy for now. The code looks like the following.

<pre>
UIImage* image = [UIImage imageNamed:@"my_image"];
[self.myButton setImage:image forState:UIControlStateNormal];

[self.myButton setTitle:@"My button" forState:UIControlStateNormal];
</pre>

The "difficult" part is that both the button and the image need to be tappable. So, if I tap the image
a specific action, different from button one, is performed.
I racked my brain and I had a thought: subclassing <code>UIButton</code>. Ok this could be a wonderful idea. But it's not.

According to <a href="http://commandshift.co.uk/blog/2013/03/12/uibutton-edge-insets/" target="_blank">UIButton Edge Insets</a> you should avoid to
subclass or reimplement <code>UIButton</code>. In fact, as Richard Turton says

> Not only is UIButton a class cluster, so not really suitable for subclassing, it also does a lot for you, and with judicious use of the image view,
background image view and (as of iOS6) attributed titles, you can create almost any effect you want.

So, what's the solution? A <code>UIButton</code> has an <code>imageView</code> property that wraps the image you have set with
<code>setImage:forState:</code> method. Even if this property is read-only, as stated by Apple documentation,
you can attach a tap gesture and respond appropriately. For example.

<pre>
UITapGestureRecognizer* tapGestureRecognizer = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(imageViewTapped:)];
tapGestureRecognizer.numberOfTapsRequired = 1;

self.bindableButton.imageView.userInteractionEnabled = YES;    
[self.bindableButton.imageView addGestureRecognizer:tapGestureRecognizer];
</pre>

Since image view objects are configured to disregard user events by default, you must
explicitly change the value of the <code>userInteractionEnabled</code> property to <code>YES</code>
after initializing the object.

Now within imageViewTapped: method you can retrieve the button associated with the tapped image as follows:

<pre>
[[tapGestureRecognizer view] superview];
</pre>
