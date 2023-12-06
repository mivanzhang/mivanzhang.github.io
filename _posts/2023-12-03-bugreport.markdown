---
layout: post
title:  "English"
date:   2023-12-03 22:26:12 +0800
categories: English
---

I  recently encountered a very interesting error. Stacktrace is below:

```java
Fatal Exception: java.lang.NullPointerException
Attempt to invoke interface method 'void android.view.ViewTreeObserver$OnGlobalLayoutListener.onGlobalLayout()' on a null object reference
android.view.ViewTreeObserver.dispatchOnGlobalLayout (ViewTreeObserver.java:1079)
android.view.ViewRootImpl.performTraversals (ViewRootImpl.java:3309)
android.view.ViewRootImpl.doTraversal (ViewRootImpl.java:2197)
android.view.ViewRootImpl$TraversalRunnable.run (ViewRootImpl.java:8673)
android.view.Choreographer$CallbackRecord.run (Choreographer.java:1031)
android.view.Choreographer.doCallbacks (Choreographer.java:849)
android.view.Choreographer.doFrame (Choreographer.java:779)
android.view.Choreographer$FrameDisplayEventReceiver.run (Choreographer.java:1016)
android.os.Handler.handleCallback (Handler.java:938)
android.os.Handler.dispatchMessage (Handler.java:99)
android.os.Looper.loop (Looper.java:257)
android.app.ActivityThread.main (ActivityThread.java:8192)
java.lang.reflect.Method.invoke (Method.java)
com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run (RuntimeInit.java:626)
com.android.internal.os.ZygoteInit.main (ZygoteInit.java:1015)

```
So I went to check the crash places, I was puzzled by the code, but why there is a NPE, I cannot understand, we are iterating the list only, how possible?:

```java
   final void dispatchOnComputeInternalInsets(InternalInsetsInfo inoutInfo) {
        // NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
        // perform the dispatching. The iterator is a safe guard against listeners that
        // could mutate the list by calling the various add/remove methods. This prevents
        // the array from being modified while we iterate it.
        final CopyOnWriteArray<OnComputeInternalInsetsListener> listeners =
                mOnComputeInternalInsetsListeners;
        if (listeners != null && listeners.size() > 0) {
            CopyOnWriteArray.Access<OnComputeInternalInsetsListener> access = listeners.start();
            try {
                int count = access.size();
                for (int i = 0; i < count; i++) {
                    access.get(i).onComputeInternalInsets(inoutInfo);
                }
            } finally {
                listeners.end();
            }
        }
    }
```

Maybe `access.get(i)` is set null by some other thread? Multi-thread issues? Investigation continues.

What's CopyOnWriteArray? There is CopyOnWriteList, is this similar? I knew that CopyOnWriteList is thread safe, but as for CopyOnWriteArray, I donot know, so check the source code:

```java

  static class CopyOnWriteArray<T> {
        private ArrayList<T> mData = new ArrayList<T>();
        private ArrayList<T> mDataCopy;

    //   ...
        private ArrayList<T> getArray() {
            if (mStart) {
                if (mDataCopy == null) mDataCopy = new ArrayList<T>(mData);
                return mDataCopy;
            }
            return mData;
        }

        Access<T> start() {
            if (mStart) throw new IllegalStateException("Iteration already started");
            mStart = true;
            mDataCopy = null;
            mAccess.mData = mData;
            mAccess.mSize = mData.size();
            return mAccess;
        }

 
        int size() {
            return getArray().size();
        }

        void add(T item) {
            getArray().add(item);
        }

        void addAll(CopyOnWriteArray<T> array) {
            getArray().addAll(array.mData);
        }

        void remove(T item) {
            getArray().remove(item);
        }

        void clear() {
            getArray().clear();
        }
    }

```
So this class trys to modify on one ArrayList, but iterates on another, its a good idea, seems the same with CopyOnWriteList, but, there is always a but, I remember there is a lock in CopyOnWriteList, but this one no!!!

After I read the code more carefully, I found CopyOnWriteArray is total not thread safe, he does modify on one ArrayList, but iterates on another, but copy and modification is not atomical, so they may be supsended while the other happens, so final there, kill the bug.