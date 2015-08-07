# RxJava 简介

### 是什么
一种扩展的观察者模式。

```java
Observable.subscribe(observer);
```

### Retrofit 中使用 RxJava （从 Retrofit 开始理解最容易）
#### 基本使用：

Callback:
```java
@GET("/user/login")
void login(@Query("username") String username, @Query("password") String password, Callback<User> callback);

...

client.login(username, password, new Callback<User>() {
    public void success(User user, Response response){
        userTextView.setText(username);
    }
    
    public void failure(RetrofitError error) {
      // handle error
    }
};
```

RxJava:
```java
@GET("/user/login")
Observable<User> login(@Query("username") String username, @Query("password") String password);

...

client.login(username, password) // login() 时只会准备请求，不会执行请求
    .subscribe(new Observer<User>() { // subscribe() 时执行请求，并在请求返回后回调 onNext() 或 onError()
      public void onNext(User user) {
        userTextView.setText(user.getName());
      }
      
      public void onError(Throwable error) {
        // handle error
      }
      
      public void onComplete() {
        // 对于 Retrofit 没用
      }
    });
```

#### 转换：
Retrofit 最长会用到有两个： `map()`, `flatMap()`

##### `map()`
```java
client.login(username, password)
    .map(new Function<User, String>() { // map(): 将对象转成指定类型
      public String call(User user) {
        return user.getName();
      }
    .subscribe(new Observer<String>() {
      public void onNext(String username) {
        userTextView.setText(username);
      }
      
      public void onError(Throwable error) {
        // handle error
      }
      
      public void onComplete() {
      }
    });
```

##### `flatMap()`
```java
client.getAccessToken(param1, param2)
    .flatMap(new Fun1<String, Observable<User>>() { // flatMap(): 由响应的数据生成新的Observable，常用语请求嵌套
      return client.login(username, password);
    })
    .map(new Function<User, String>() {
      public String call(User user) {
        return user.getName();
      }
    .subscribe(new Observer<String>() {
      public void onNext(String username) {
        userTextView.setText(username);
      }
      
      public void onError(Throwable error) {
        // handle error
      }
      
      public void onComplete() {
      }
    });
```

#### 指定线程：
```java
client.login(username, password)
    .doOnNext(new Action1(User) {
      public void call(User user) {
        saveUserToDatabase(user); // 耗时操作
      }) 
    .map(new Function<User, String>() {
      public String call(User user) {
        return user.getName();
      }
    .subscribeOn(Schedulers.io()) // 指定工作（这一行之前）所在的线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定回调（Observer）所在的线程
    .subscribe(new Observer<String>() {
      public void onNext(String username) {
        userTextView.setText(username);
      }
      
      public void onError(Throwable error) {
        // handle error
      }
      
      public void onComplete() {
      }
    });
```

> 为什么是`onNext<T>(T t)` 和 `onComplete()`，而不是`onSuccess<T>(T t)`?

> （我要开始胡扯了） Rx 本来用于服务端，基于响应式编程的思想，将所有请求变成基于事件订阅的机制，目的是把 MVC 拆得更散。RxJava 并非是为了 Retrofit 而创造的， Retrofit 只是把它拿来用。

### RxJava 中的一些概念（类或方法）
* Observable: 被观察对象。担当发送事件的角色。在 Retrofit 中，请求被包装进一个 Observable 中，并在请求结束后回调 `Observer.onNext()` 或`Observable.onError()`
* Observer: 观察者。担当接收事件的角色。通过 `Observable.subscribe()` ，Observer 将接收到它订阅的 Observable 的事件。
* Subscription: 订阅（名词）。在 `Observable.subscribe()` 中返回。作用只有一点：通过 `Subscription.unsubscribe()` 来解除 Observer 和 Observable 的订阅关系。
* Scheduler: 调度器。担当线程分配的角色。通过 `Observable.subscribeOn()` 和 `Observable.observeOn()` 分配。

### RxJava 基本使用
#### 创建 Observable
* `Observable.create(OnSubscribe)`: 完全手动创建 Observable 对象，并控制 Observer 的回调。
* `Observable.just(T...)`: 将参数逐个发送（调用 observer 的 `onNext()` 和 `onComplete()`。
* `Observable.from(T[])`: 将数组拆开，逐个发送。

#### 转换：
转换有两类：lift 和 compose。这个太复杂了等我读了源码再说吧。最常用的是两个 lift 方法：`map()` 和 `flatMap()`，前面已经提到过。

#### 线程分配：
前面提到过，两个方法：`subscribeOn()` 和 `observeOn()`，分别用来分配工作线程和监听线程。

#### 订阅
* `Observable.subscribe(Observer)`: 将传入的 Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Action1 onNext)`: 将传入的 onNext 组装成 Observer 后，将组装的Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Action1 onNext, Action1 onError)`: 将传入的 onNext 和 onError 组装成 Observer 后，将组装的Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Action1 onNext, Action1 onError, Action0 onComplete)`: 将传入的 onNext, onError, onComplete 组装成 Observer 后，将组装的Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Subscriber)`: 和 `subscribe(Observer)` 一样。Subscriber 是 Observer 接口的一个实现，它同时实现了 Subscription 接口。实质上， `subscribe(Observer observer)` 的内部实现是用 observer 实现一个 Subscriber 后调用 `subscribe(Subscriber)`。

#### 取消订阅
取消订阅的原因有两种情况：

1. 不再需要。方式： `Subsription.unsubscribe()` (可以先通过 `Subscription.isUnsubscribed()` 检查一下)。
2. 防止内存泄露。在合适的地方 (`Activity.onDestroy()`, `View.onDetachToWindow()`等)调 `Subscription.subscribe()`. 或者使用**Flipboard 的 toolbox 中的便捷方法：`BindTransformer` ([示例](https://github.com/Flipboard/android/blob/cacf09a0b45b1682a9d79a7990f5ed3443af87b4/apps/flipboard/src/main/java/flipboard/gui/section/SectionFragment.java#L1282))**.

#### RxAndroid
RxAndroid 是 RxJava 的 Android 扩展，增加了一些 Android 的便利方法。目前我看到我们在用的有：
* `ViewObservable.clicks()`: 设置点击监听器，可以使用 `throttleFirst()` 来添加防手抖间隔。 [示例](https://github.com/Flipboard/android/blob/17b1ca97d4b0640cd516c0b0ba6c786d52405448/apps/briefing/src/main/java/flipboard/boxer/gui/item/ItemViewHolder.java#L183-L203)
# `WidgetObservable.text()`: 设置文字监听器。[示例](https://github.com/Flipboard/android/blob/b95af6f0e3eb03e85b29694c633192fbe0f7439f/apps/flipboard/src/main/java/flipboard/activities/CreateMagazineActivity.java#L194-L213)
