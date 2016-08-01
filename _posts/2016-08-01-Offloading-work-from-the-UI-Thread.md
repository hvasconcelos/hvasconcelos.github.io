---
layout: post
title: Offloading work from the UI Thread on Android
description: "Offloading work from the UI Thread on Android"
category: articles
tags: [Android,Java,Asynchronous,RxJava,Looper,Handler,JobScheduler,IntentService]
comments: true
---

In this article, I will present the most common asynchronous techniques techniques used on Android to develop an application, that using the multicore CPUs available on the most recent devices, is able to deliver up to date results quickly, respond to user interactions immediately and produce smooth transitions between the different UI states without draining the device battery.
Several reports have shown that an efficient application that provides a great user experience have better review ratings, higher user engagement and are able to achieve higher revenues.

## Why do we need Asynchronous Programming on Android?
The Android system, by default, executes the UI rendering pipeline, base components (Activity, Fragment, Service, BroadcastReceiver, ...) lifecycle callback handling and UI interaction processing on a single thread, sometimes known as UI thread or main thread.

The main thread handles his work sequentially collecting its work from a queue of tasks (Message Queue) that are queued to be processed by a particular application component.  When any unit of work, such as a I/O operation, takes a significant period of time to complete, it will block and prevent the main thread from handling the next tasks waiting on the main thread queue to processed.

![main_thread_queue]({{ site.url }}/images/main_thread_queue.png)

Besides that, most Android devices refresh the screen 60 times per second, so every 16 milliseconds (1s/60 frames) a UI rendering task has to be processed by the main thread in order to draw and update the device screen. The next figure shows up a typical main thread timeline that runs the UI rendering task every 16ms.

![main_thread_rendering]({{ site.url }}/images/main_thread_rendering.png)

When a long lasting operation, prevents the main thread from executing frame rendering in time, the current frame drawing is deferred or some frame drawings are missed generating a UI glitch noticeable to the application user.

A typical long lasting operation could be:

* Network data communication
* HTTP REST Request
* SOAP Service Access
* File Upload or Backup
* Reading or writing of files to the filesystem
* Shared Preferences Files
* File Cache Access
* Internal Database reading or writing
* Camera, Image, Video, Binary file processing.

A user Interface glitch produced by dropped frames on Android is known on Android as jank. The Android SDK command systrace (https://developer.android.com/studio/profile/systrace.html)
 comes with the ability to measure the performance of UI rendering and then diagnose and identify problems that may arrive from various threads that are running on the application process.
In the next image we illustrate a typical main thread timeline when a blocking operation dominates the main thread for more than 2 frame rendering windows.

![blocking_operation]({{ site.url }}/images/blocking_operation.png)


As you can perceive, 2 frames are dropped and the 1 UI Rendering frame was deferred because our blocking operation took approximately 35ms, to finish.

When the long running operation that runs on the main thread does not complete within 5 seconds, the Android System displays an “Application not Responding” (ANR) dialog to the user giving him the option to close the application.

Hence, in order to execute compute-intensive or blocking I/O operations without dropping a UI frame, generate UI glitches or degrade the application responsiveness, we have to offload the task execution to a background thread, with less priority, that runs concurrently and asynchronously in an independent line of execution, like shown on the following picture.

![async_processing]({{ site.url }}/images/async_processing.png)

Although, the use of asynchronous and multithreaded techniques always introduces complexity to the application, Android SDK and some well known open source libraries provide high level asynchronous constructs that allow us to perform reliable asynchronous work that relieve the main thread from the hard work.
Each asynchronous construct has advantages and disadvantages and by choosing the right construct for your requirements can make your code more reliable, simpler, easier to maintain and less error prone.

Let’s enumerate the most common techniques that are covered in detail in the “Asynchronous Android Programming” book.

## AsyncTask
AsyncTask is simple construct available on the Android platform since Android Cupcake (API Level 3) and is the most widely used asynchronous construct. The AsyncTask was designed to run short-background operations that once finished update the UI.
The AsyncTask construct performs the time consuming operation, defined on the doInBackground function, on a global static thread pool of background threads. Once doInBackground terminates with success, the AsyncTask construct switches back the processing to the main thread (onPostExecute) delivering the operation result for further processing.

```
public class DownloadImageTask extends AsyncTask<URL, Integer, Bitmap> {

  protected Long doInBackground(URL... urls) {}
  protected void onProgressUpdate(Integer... progress) {}
  protected void onPostExecute(Bitmap image) {}
}
```

This technique if it is not used properly can lead to memory leaks or inconsistent results.

## HandlerThread
The HandlerThread is a Threat that incorporates a message queue and an Android Looper that runs continuously waiting for incoming operations to execute. To submit new work to the Thread we have to instantiate a Handler that is attached to HandlerThread Looper.

```
HandlerThread handlerThread = new HandlerThread("MyHandlerThread");
handlerThread.start();
Looper looper = handlerThread.getLooper();
Handler handler = new Handler(looper);
handler.post(new Runnable(){
        @Override
        public void run() {}
    });
```

The Handler interface allow us to submit a Message or a Runnable subclass object that could aggregate data and a chunk of work to be executed.

## Loader
The Loader construct allow us to run asynchronous operations that load content from a content provider or a data source, such as an Internal Database or a HTTP service. The API can load data asynchronously, detect data changes, cache data and is aware of the Fragment and Activity lifecycle. The Loader API was introduced to the Android platform at API level 11, but are available for backwards compatibility through the Android Support libraries.

```
public static class TextLoader extends AsyncTaskLoader<String> {

    @Override public String loadInBackground() {
        // Background work
    }

    @Override public void deliverResult(String text) {}
    @Override protected void onStartLoading() {}
    @Override protected void onStopLoading() {}
    @Override public void onCanceled(String text) {}
    @Override protected void onReset() {}
}
```

## IntentService
The IntentService class is a specialized subclass of Service that implements a background work queue using a single HandlerThread. When work is submitted to an IntentService, it is queued for processing by a HandlerThread, and processed in order of submission.

```
public class BackupService extends IntentService {

    @Override
    protected void onHandleIntent(Intent workIntent) {
        // Background Work
    }
}
```

## JobScheduler
The JobScheduler API allow us to execute jobs in background when several prerequisites are fulfilled and taking into the account the energy and network context of the device. This technique allows us to defer and batch job executions until the device is charging or an unmetered network is available.

```
JobScheduler scheduler = (JobScheduler) getSystemService(
    Context.JOB_SCHEDULER_SERVICE );
JobInfo.Builder builder = new JobInfo.Builder(JOB_ID,serviceComponent);
builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED);
builder.setRequiresCharging(true);
scheduler.schedule(builder.build());
```

## RxJava
RxJava is an implementation of the Reactive Extensions (ReactiveX) on the JVM, that was developed by Netflix, used to compose asynchronous event processing that react to an observable source of events.
The framework extends the Observer pattern by allowing us to create a stream of events, that could be intercepted by operators (input/output) that modify the original stream of events and deliver the result or an error to a final Observer.
The RxJava Schedulers allow us to control in which thread our Observable will begin operating on and in which thread the event is delivered to the final Observer or Subscriber.

```
Subscription subscription = getPostsFromNetwork()
    .map(new Func1<Post, Post>() {
        @Override
        public Post call(Post Post) {
            ...
            return result;
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Observer<Post>() {
        @Override
        public void onCompleted() {}

        @Override
        public void onError() {}

        @Override
        public void onNext(Post post) {
            // Process the result
        }
    });
```

## Summary
As you have seen, there are several high level asynchronous constructs available to offload the long computing or blocking tasks from the UI thread and it is up to developer to choose right construct for each situation because there is not an ideal choice that could be applied everywhere.
Asynchronous multithreaded programming, that produces reliable results, is difficult and error prone task so using a high level technique tends to simplify the application source code and the multithreading processing logic required to scale the application over the available CPU cores.
Remember that keeping your application responsive and smooth is essential to delight your users and increase your chances to create a notorious mobile application.

The techniques and asynchronous constructs summarized on the previous paragraphs are covered in detail in the “Asynchronous Android Programming” book published by Packt Publishing.

* [Packt Publishing](https://www.packtpub.com/application-development/asynchronous-android-programming-second-edition)


