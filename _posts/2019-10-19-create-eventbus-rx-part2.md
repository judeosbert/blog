---
layout: post  
title:  "DIY Eventbus with Rx Part 2"  
author: jude
avatar: assets/images/njan.jpg  
categories: [ android, tutorial, eventbus, rx ]  
tags: [sticky,android, tutorial, eventbus, rx]
image: assets/images/diy-eventbus-rx.jpg 
---
  
# extending DIY EventBus to multiple activties  

> Find the code at [link]([https://github.com/judeosbert/DIY-EventBus/tree/part2](https://github.com/judeosbert/DIY-EventBus/tree/part2))

In a previous post, I had paved the idea for a simple eventbus. You can read it [here]([https://judeosbert.github.io/blog/create-eventbus-rx/](https://gnldr.website/tracker/click?redirect=https%3A%2F%2Fjudeosbert.github.io%2Fblog%2Fcreate-eventbus-rx%2F&dID=1570443589782&linkName=https://judeosbert.github.io/blog/create-eventbus-rx/)). I strongly suggest you read that before you continue. I had registered the listeners in my BaseActivity limiting the interaction to only that Activity and not others. This is not useful if you want to refresh the  data in an activity other than the base activity based on some events sent through the bus.  
In this post I will suggest a way to make our bus even more useful.  
  
## inheritance to the rescue  
In our `BaseActivity` we have a listener class, `BusEventListeners` extending the `MainThreadBusListener`. We will create indivdiual listeners for all activites which will listen to the bus events.  
We also don't want to duplicate functionalites and function calls for functionalities that are not part of the activity. The ideal event flow should be hence to get all events in the class listener, handle what is required or related to the that activity and pass the rest to the super class. This will also give us power to prevent some events getting executed by just handling it in a `when` case and not performing any action.  
So, we need to call `super.onNext(event)` , `super.onError(event)`,`super.onCompleted(event)`.  
The way to do this is the `@CallSuper` annotation.  
  
## @CallSuper  
When a method within a class is annotated with this , all children should have a call to the super method. This is why we are forced to do super call for lifecycle events. For example, `super.onStart()` 
  
## modifying the BaseEventListenerClass  
We can modify the class with the annotation like below  
```  
open inner class BusEventObserver : MainThreadBusEventObserver {  
@CallSuper  
override fun onNext(event: BusEvent) { }  
@CallSuper  
override fun onError(it: Throwable) { }  
@CallSuper  
override fun onComplete() { }  
}  
```  
We will also need to add the `open` keyword to the class to make it inheritable.  
Wiht the @CallSuper annotation, Studio will throw an error if the child class doesn't call the super method.  
This is exactly what we want.
# extending DIY EventBus to multiple activties  

Lets create a new Activity and call it `Activity2`.  I will create an inner `BusEventListener` class as below

    inner class Activity2BusEventListener:BusEventListeners(){  
  
    override fun onComplete() {  
        super.onComplete()  
    }  
  
    override fun onNext(t: BusEvent) {  
     super.onNext(t)  
    }  
  
    override fun onError(e: Throwable) {  
        super.onError(e)  
    }  
  
	}
I will add a button to the new Activity just like I did in the previous post to post events. The first button will take me to the second activity and the second button will post the event. 
Just for demo purpose,I will change the button text to current timestamp and base activity will make a toast.
Run the code and you can see that both the listeners are getting fired. 
With the same idea, we can extend the bus to fragments and activites as well.
You should create another listener class that is a child of the listener class of the Activity.

Hope you learned something new. If you see something is missing or wrong, post it in comments and would be happy to correct it. 

As always, you can find the code [here]([https://github.com/judeosbert/DIY-EventBus/tree/part2](https://github.com/judeosbert/DIY-EventBus/tree/part2))

