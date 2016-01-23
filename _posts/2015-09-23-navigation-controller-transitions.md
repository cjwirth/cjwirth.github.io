---
layout: post
title: Keeping Swipe Back with UINavigationController Transitions
---

Pretty much everyone uses the Swipe Back feature that you get for free with _UINavigationController_. But what happens when you add some awesome animated transitions for your push actions? You can't swipe back anymore!

I found a way around this! Though, it does feel a bit hacky...

<!--excerpt-->

### tl;dr

You will lose swipe back if you overwrite the delegate of _UINavigationController_. So, after the animation is complete, return it to what it used to be.

### It's Chicken and Egg Problem

To make a custom transition for a _UINavigationController_'s push or pop animations, I need to become the navigation controller's delegate property, so I can provide it with the _UIViewControllerAnimatedTransitioning_ to perform the animation.

The problem is that if I overwrite the navigation controller's delegate property, I can no longer use the default swipe back function!

Sure, I could go through the effort and reimplement the swipe back functionality using a _UISwipeGestureRecognizer_ and probably a _UIPercentDrivenInteractiveTransition_...

That sounds like a lot of work, and my custom swipe back could look out of place if Apple ever decides to change that animation.

### The Hacky Workaround

All I want to do is provide a custom transition, so really all you need to implement is one method. I don't care about the rest. I only want this one:

```Swift
func navigationController(navigationController: UINavigationController, animationControllerForOperation operation: UINavigationControllerOperation, fromViewController fromVC: UIViewController, toViewController toVC: UIViewController) -> UIViewControllerAnimatedTransitioning? {
	    // Some transition from somewhere
	    return transition
}
```

Wouldn't it be nice if I could _just_ intercept that one method? I tried making a wrapper class that returns the transition, and hands the delegate keys back to the whoever was the previous delegate.

```Swift
func navigationController(navigationController: UINavigationController, didShowViewController viewController: UIViewController, animated: Bool) {
	    // Now that the transition is complete, 
	    // so we'll return delegate to what it used to be
	    navigationController.delegate = previousDelegate
	    previousDelegate = nil
}
```

And that's really all it took. I get my awesome custom animation, and my users can still swipe back to the previous screen like they would expect.

If you need more context, I made a [Sample Project on Github](https://github.com/cjwirth/NavigationControllerTransitions) with a working [NavigationTransitionController](https://github.com/cjwirth/NavigationControllerTransitions/blob/master/NavigationTransitions/NavigationTransitionController.swift) included. 

If you use the sample code, you might want to forward some delegate calls to the `previousDelegate` -- it was unnecessary in the sample, but it probably wouldn't hurt to be a good citizen.


_This is a problem I encountered quite a long time ago. If there is a better, more correct way to fix this, let me know in the comments!_
