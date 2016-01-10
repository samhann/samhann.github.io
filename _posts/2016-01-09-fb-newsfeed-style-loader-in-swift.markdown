---
layout: post
title: "FB Newsfeed style loader in Swift"
tags: ['swift' , 'ios']
date: 2016-01-10
---

Facebook has a nifty animation for the placeholder while loading the newsfeed . Today we'll take a shot at replicating it on iOS.

![Final result](https://camo.githubusercontent.com/5b1b29c22d438bdb52429d083d84241775d083db/687474703a2f2f672e7265636f726469742e636f2f784156374b50356c437a2e676966)


I began as all of us do , by Googling for a ready-made implementation :P . As luck would have it, I found this excellent [deconstruction](http://cloudcannon.com/deconstructions/2014/11/15/facebook-content-placeholder-deconstruction.html) and HTML+JS implementation by George Philips.

Turns out it isnt that hard, and its even easier on iOS as the animation and drawing APIs are way more powerful.


## Creating the Gradient

First , you need an animated gradient that looks like this :

![Animation](/images/loader.gif)


This gradient is then added to the content view of every cell. 

```swift

        var gradient = CAGradientLayer()
        gradient.startPoint = CGPointMake(-1.0 + CGFloat(gradientWidth), 0)
        gradient.endPoint = CGPointMake(1.0 + CGFloat(gradientWidth), 0)
        
        gradient.colors = [
            UIColor.backgroundFadedGrey().CGColor,
            UIColor.gradientFirstStop().CGColor,
            UIColor.gradientSecondStop().CGColor,
            UIColor.gradientFirstStop().CGColor,
            UIColor.backgroundFadedGrey().CGColor
        ]
```

Having set the colours, we set the locations or points for the gradient. I declared an extension on CGFloat to easily extract doubleValues.

```swift
    let startLocations = [NSNumber(double: gradient.startPoint.x.doubleValue()),
                            NSNumber(double:gradient.startPoint.x.doubleValue()),
                            NSNumber(double:0),NSNumber(double: gradientWidth),
                            NSNumber(double: 1 + gradientWidth)]

    gradient.locations = startLocations
```

Now , to animate the gradient using Core Animation.


```swift
  let gradientAnimation = CABasicAnimation(keyPath: "locations")
  gradientAnimation.fromValue = startLocations
  
  gradientAnimation.toValue = 
        [NSNumber(double: 0),NSNumber(double:1),
         NSNumber(double:1),
         NSNumber(double:1+(gradientWidth-gradientFirstStop)),
         NSNumber(double: 1 + gradientWidth)]
  
  gradientAnimation.repeatCount = Float.infinity
  gradientAnimation.fillMode = kCAFillModeForwards
  gradientAnimation.removedOnCompletion = false
  gradientAnimation.duration = loaderDuration
  gradient.addAnimation(gradientAnimation ,forKey:"locations")

```

A `CAGradientLayer` has a `locations` property which can be animated to move the gradient along the length of the view.

We create a `CABasicAnimation` for the locations key , with a repeat count of infinity so that it continues animating.


## Add the Cutout

Now that we've added the animated gradient to the contentview, we need to cover it with white and cut out "holes" where the other views are. Imagine a white wall in front of the animating gradient , with windows wherever the other views are.

This is also easy to do using CoreGraphics.

We create a class called CutoutView which fills itself with a white background but clears the rects where all the other views are.

First we create the CutoutView like so :

```swift

        let cutout = CutoutView()
        cutout.frame = self.bounds
        cutout.backgroundColor = UIColor.clearColor()
        
        self.addSubview(cutout)
        cutout.setNeedsDisplay()
        cutout.boundInside(self)
        
        for view in self.subviews {
            if view != cutout {
                view.alpha = 0
            }
        }

        self.ld_setCutoutView(cutout)
    
```

We set the alphas of all the other views to zero and set autolayout constraints in the `boundInside` method to handle layout changes.

So how does the cutout draw its windows ?

To draw the "holes" , we use the following `drawRect:` implementation :

```swift
    override func drawRect(rect: CGRect) {
        
        super.drawRect(rect)
        let context = UIGraphicsGetCurrentContext()
        CGContextSetFillColorWithColor(context, UIColor.whiteColor().CGColor)
        CGContextFillRect(context, self.bounds)
        
        for view in (self.superview?.subviews)! {

          if view != self {
            CGContextSetBlendMode(context, .Clear);
            CGContextSetFillColorWithColor(context, UIColor.clearColor().CGColor)
            CGContextFillRect(context, view.frame)
          }
        }
    }

```
The trick here is to use `CGContextSetBlendMode(context,.Clear)`, using which we clear the rects of all the other subviews of this superview except ofcourse the `CutoutView`.

This way of doing things allows for some degree of flexibility for changing your cell layouts in interface builder. However ,more complex layouts with multiline text fields or overlapping views may give unexpected results.

And voila !

![Final result](https://camo.githubusercontent.com/5b1b29c22d438bdb52429d083d84241775d083db/687474703a2f2f672e7265636f726469742e636f2f784156374b50356c437a2e676966)

While adding the animation to multiple cells , we do so in a `CATransaction` so that they all animate in sync.

An additional enhancement might be to draw the rects a little smaller in height if the view is a UILabel.

The full code is on [Github](https://github.com/samhann/Loader.swift).
