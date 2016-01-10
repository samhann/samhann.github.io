---
layout: post
title: "FB Newsfeed style loader in Swift"
tags: ['swift','ios']
date: 2016-01-10
---

Facebook has a really nifty animation while loading the newsfeed . Today we'll take a shot at doing this on iOS using Swift.

Ofcourse , I began , like all professional programmers by Googling for a ready-made implementation :P . As luck would have it I got this excellent [deconstruction](http://cloudcannon.com/deconstructions/2014/11/15/facebook-content-placeholder-deconstruction.html) of the animation by George Philips.

Turns out this isnt really that hard to do. Its even easier on iOS than the above one in CSS and JS.


## Creating the Gradient

First , you need an animated gradient that looks like this :

![Animation](/images/loader.gif)


This gradient is added to the content view of every cell. 

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


Next we set the locations or points for the gradient. I declared an extension on CGFloat to easily extract doubleValues.

```swift
    let startLocations = [NSNumber(double: gradient.startPoint.x.doubleValue()),
                            NSNumber(double:gradient.startPoint.x.doubleValue()),
                            NSNumber(double:0),NSNumber(double: gradientWidth),
                            NSNumber(double: 1 + gradientWidth)]

    gradient.locations = startLocations
```

Now we animate the gradient using Core Animation.


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

The gradient has a locations property which we animate to move the gradient along the view. We give it a repeat count of infinity so that it continues animating.


## Add the Cutout

So now we've added the gradient to the contentview. Now we need to cover up the gradient with white with "holes" where the other views. This is also easy to do. 

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

We set the alphas of all the other views to zero and set autolayout constraints in the boundInside method to handle layout changes.

Now to draw the "holes" , we use the following `drawRect:` implementation :

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
The trick here is in using `CGContextSetBlendMode(context,.Clear)`.
That clears the rects of all other subviews of this superview except the cutout.

This allows for some degree of flexibility for changing your layouts using Storyboards. However ,more complex layouts with multiline text fields or overlapping views may give unexpected results.

And voila !

![Final result](https://camo.githubusercontent.com/5b1b29c22d438bdb52429d083d84241775d083db/687474703a2f2f672e7265636f726469742e636f2f784156374b50356c437a2e676966)

The full code is on [Github](https://github.com/samhann/Loader.swift).
