---
layout: post  
title:  "DIY Eventbus with Rx"  
author: jude
avatar: assets/images/njan.jpg  
categories: [ android, tutorial, eventbus, rx ]  
tags: [sticky]
image: assets/images/diy-eventbus-rx.jpg 
---
  
Recently at work I ran into a situation where I needed an eventbus. Irrespective of the activity I was in, I needed to show a popup which had few CTAs. My approach was to register an eventbus listener in the base class and have all classes extend that base class.   
  
I could simply use an existing solution which I was comfortable with , Ottobus for example, but I did not want to have an extra library just for a popup functionality. Since we were using RxKotlin, I decided to do it myself.  
  
## prerequisites  
  
I am going to use RxKotlin for the project. If you don't have any idea about reactive programming,  it's high time you learn rx. Everything is turning reactive !! Other than that you should have basic idea of Android, Lifecycles and Kotlin.   
  
## misconception about the onPause method  
  
I have a question for you. In my application I will be registering the bus at onResume and unregister at onPause method. So If I have a popup that's currently shown, won't the onPause method be called and unregister my bus?  
If you are nodding your head, I am afraid you are wrong. This is a popular misconception that devs have.  
If you were asked this question in an interview, there are good chances it would turn into an argument.  
  
### the concept  
This misconception stems from the statement in documentation that showing a dialog triggers the onPause method.  That is correct but those **dialogs are activities themed as dialogs**.  
  
The onPause method is called when your activity moves from the top of the activity stack. To do that you have to show another activity on top of that. Dialogs are not activities,right?. So they have to be activities themed as a dialog. For example, permission dialogs, they trigger onPause , but showing a alert dialog doesn't.  
  
## why onPause and onResume , why not onStart and onStop?
This was my first implementation, but when I switch activities I will never get any events. Why?  
When we do `Context.startActivity(intent:Intent)` things don't go according to our imagination.  
When we move from `Activity A` to `Activity B`, life cycle events are called as follow.  
  
1->`onPause of A`  
  
2->`onCreate of B, onStart of B , onResume of B and then onStop of A (if it's no longer visible on the screen)`.  
  
You can find it at the official documentation page at [docs](https://developer.android.com/guide/components/activities/activity-lifecycle.html#coordinating-activities).  
  
So Activity B subscribes to the Rx Subject at onStart, and onStop of Activity A disposes the `CompositeDisposible` leaving B with no active subscription and therefore no events. So the best place to do subscription is at onResume and dispose at onPause.   
  
## let's code  
Open up your Android Studio and create a new project. You can call it whatever you want. I will be using  androidx.* artifacts for this. Let studio finish the build tasks.Once done, we will add the dependencies required. Open up your `build.gradle` inside your app folder.  
  
We need RxKotlin [link](https://github.com/ReactiveX/RxKotlin)  
`implementation("io.reactivex.rxjava2:rxkotlin:x.y.z")`  
  and we need the AndroidBindings via RxAndroid [link]([https://github.com/ReactiveX/RxAndroid](https://github.com/ReactiveX/RxAndroid))<br />
`implementation 'io.reactivex.rxjava2:rxandroid:x.y.z'`<br/>
Once all is set, Create a new kotlin file called `EventBus.kt`.  
```kotlin
class EventBus private constructor() {  
    fun register(listener: MainThreadBusListener) { }  
    fun unregister() {}  
    fun postEvent(key:EventKeys,data:Any){}
    
	private var mDisposable:Disposable?= null
    private val mEventSubject = PublishSubject.create<BusEvent>();  
  
    companion object{  
        private var mInstance:EventBus? = null  
        fun getInstance():EventBus?{  
            if(mInstance == null){  
                mInstance = EventBus()  
            }  
            return mInstance  
  }  
    }  
}
 ```
 I am creating a new PublishSubject of type `BusEvent`. `BusEvent` is a data class that has a key, which will help  to distinguish the events, and the data which can be of `Any` type. Here I am not making it nullable.  
Since there should only be a single instance of the bus, I have made it Singleton, but it should ideally be done by  Dependency Injection via Koin or similar frameworks. 
And now the **BusEvent class**
```kotlin
data class BusEvent(val key:EventKeys,val data:Any)
 ```
 **EventKeys Enum Class**
 
```kotlin
enum class EventKeys{  
    EVENTA  
}
``` 
  
The key for the EventBus comes from a enum class `EventKeys`. This will help us keep related things together in one place.  
  
Now we will create the interface for the callbacks
```kotlin
interface MainThreadBusListener{  
    fun onComplete()  
  
    fun onNext(t: BusEvent)  
  
    fun onError(e: Throwable)  
}
 ```

We will fill in the register and unregister functions.

*register function*
```kotlin
fun register(listener:MainThreadBusListener){
	mDisposable = mEventSubject.observeOn(AndroidSchedulers.mainThread())  
    .subscribeBy(  
        onNext = {  
          listener.onNext(it)  
        },  
        onError = {  
          listener.onError(it)  
        },  
        onComplete = {  
          listener.onComplete()  
        }  
  )
}
```
We will observe the subject on main thread since there might be view references at the listener side. If you want to have separate methods for subscribing on background thread, you can do so too by ignoring the `observeOn()` method.
You can also see that we are adding the subscription to the disposable we created so that the duty of cleaning up stays within the `EventBus` class. We wouldn't want each of the implementing classes to worry about that. 
We should remove the subscription once the class moves to `onPause` method.
We will do that by disposing the subscription in the `unregister` function

```kotlin
fun unregister() = mDisposable?.dispose()
```
Now all we need is a method to sent the event
```kotlin
fun postEvent(key:EventKeys,data:Any){  
    mEventSubject.onNext(BusEvent(key,data))  
  
}
```
That's it!!!

## Let's see it in Action

To demo this I will create a button and on clicking it we will send an event to the bus and a toast would be shown with the text send on the click.
I will add a button to the activity
```xml
<?xml version="1.0" encoding="utf-8"?>  
<androidx.constraintlayout.widget.ConstraintLayout  
  xmlns:android="http://schemas.android.com/apk/res/android"  
  xmlns:tools="http://schemas.android.com/tools"  
  xmlns:app="http://schemas.android.com/apk/res-auto"  
  android:layout_width="match_parent"  
  android:layout_height="match_parent"  
  tools:context=".MainActivity">  
 <Button  android:text="Button"  
  android:layout_width="wrap_content"  
  android:layout_height="wrap_content"  
  android:id="@+id/button" app:layout_constraintStart_toStartOf="parent"  
  app:layout_constraintEnd_toEndOf="parent"  
  app:layout_constraintTop_toTopOf="parent"  
  app:layout_constraintBottom_toBottomOf="parent"/>  
</androidx.constraintlayout.widget.ConstraintLayout>
```

Go to the `MainActivity.kt`.
```kotlin
class MainActivity : AppCompatActivity() {  
  
  
    private val mBus = EventBus.getInstance()  
    private val mEventListeners = BusEventListeners()  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
        button.setOnClickListener {  
  mBus?.postEvent(EventKeys.EVENT_A,"This is a sample data")  
        }  
  }  
  
    override fun onPause() {  
        super.onPause()  
        mBus?.unregister()  
  
    }  
  
    override fun onResume() {  
        super.onResume()  
        mBus?.register(mEventListeners)  
    }  
  
  
    inner class BusEventListeners:MainThreadBusListener{  
        override fun onComplete() {  
              
        }  
  
        override fun onNext(t: BusEvent) {  
            when(t.key) {  
                EventKeys.EVENT_A ->  
                Toast.makeText(applicationContext,
                 t.data as String,
                  Toast.LENGTH_SHORT).show()  
            }  
        }  
  
        override fun onError(e: Throwable) {  
              
        }  
  
    }  
}
```
I have created an inner class which will manage all the events. The responsibility will stay in one class.Now on button click, I will simple get the bus instance `mBus` and call `post` method to trigger an event. The listener will get a callback on `onNext` and if the keys match a toast will be shown.

Try it out and see for yourself.

## Final Thoughts
This is a very simple use case of Rx and how you can build a simple event bus. This has its shortcomings, but for a small project where you need to sent a data only once, or for a single event from a notification,for example, having an external library is too costly.

