# RxJava 简介

### 这篇文章简介的目的
让 Flipboard 北京团队快速上手 RxJava， 美国人已经用熟了。

### RxJava 粗定义
一种扩展的观察者模式。代码长这样：

```java
Observable.subscribe(observer);
```

### RxJava 中的一些概念
* Observable: 被观察对象。担当发送事件的角色。在 Retrofit 中，请求被包装进一个 Observable 中，并在请求结束后回调 `Observer.onNext()` 或`Observable.onError()`
* Observer: 观察者。担当接收事件的角色。通过 `Observable.subscribe()` ，Observer 将接收到它订阅的 Observable 的事件。
* subscribe: 订阅（动词）。即将一个 Observable 和一个 Observer 绑定。
* Subscription: 订阅（名词）。在 `Observable.subscribe()` 中返回。作用只有一点：通过 `Subscription.unsubscribe()` 来解除 Observer 和 Observable 的订阅关系。
* Scheduler: 调度器。担当线程分配的角色。通过 `Observable.subscribeOn()` 和 `Observable.observeOn()` 分配。

### Flipboard 目前使用 RxJava 的场景
一共主要有3种：

1. 配合 Retrofit 使用，实现比 Callback 更加灵活的 HTTP 请求和回调；
2. RxAndroid 的基本使用，实现便捷的多线程控制；
3. 用 RxBus 来实现 EventBus ，替代 Otto。

下面分别对这三类进行说明。

### 场景一： Retrofit （从 Retrofit 开始理解最容易）
#### 1 基本使用——观察者模式：

传统 Callback:
```java
@GET("/user/login")
void login(@Query("username") String username, @Query("password") String password, Callback<User> callback);

...

client.login(username, password, new Callback<User>() {
    @Override
    public void success(User user, Response response){
        userTextView.setText(username);
    }
    
    @Override
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
        @Override
        public void onNext(User user) {
            userTextView.setText(user.getName());
        }
        
        @Override
        public void onError(Throwable error) {
            // handle error
        }
        
        @Override
        public void onComplete() {
            // 对于 Retrofit 没用
        }
    });
```

> 这里用到了 RxJava 的最基本概念： Observable 、 Observer 、 subscribe (订阅，和注册同理)。

>> *为什么是`onNext<T>(T t)` 和 `onComplete()`，而不是`onSuccess<T>(T t)`?（我要开始胡扯了）:*

>> *Rx 本来用于服务端，基于响应式编程的思想，将所有请求变成基于事件订阅的机制，目的是把 MVC 拆得更散。RxJava 并非是为了 Retrofit 而创造的， Retrofit 只是把它拿来用。*

#### 2 转换：
RxJava 允许将数据流中的对象进行多种形式的转换后再由 Observer 进行处理。Retrofit 最常会用到的有两个转换方法： `map()`, `flatMap()`

##### `map()`
```java
client.login(username, password)
    .map(new Fun1<User, String>() { // map(): 将对象转成指定类型
        @Override
        public String call(User user) {
            return user.getName();
        }
    .subscribe(new Observer<String>() { // 然后这里就由 User 变成了 String
        @Override
        public void onNext(String username) {
            userTextView.setText(username);
        }
      
        @Override
        public void onError(Throwable error) {
            // handle error
        }
      
        @Override
        public void onComplete() {
        }
    });
```

##### `flatMap()`
```java
client.getAccessToken(param1, param2)
    // flatMap(): 由响应数据生成新的Observable，常用于请求嵌套，比多层包裹的 Callback 清晰些
    .flatMap(new Fun1<String, Observable<User>>() {
        @Override
        public Observable<User> call(String token) {
            return client.login(username, password, token);
        }
    })
    .map(new Func1<User, String>() {
        @Override
        public String call(User user) {
            return user.getName();
        }
    .subscribe(new Observer<String>() {
        @Override
        public void onNext(String username) {
            userTextView.setText(username);
        }
        
        @Override
        public void onError(Throwable error) {
            // handle error
        }
        
        @Override
        public void onComplete() {
        }
    });
```

#### 3 多线程分配：
```java
client.login(username, password)
    .doOnNext(new Action1(User) {
        public void call(User user) {
            saveUserToDatabase(user); // 读写数据库属于耗时操作，放在后台不会阻塞主线程
        }) 
    .map(new Function<User, String>() {
        public String call(User user) {
            return user.getName();
        }
    .subscribeOn(Schedulers.io()) // subscribeOn() 负责指定工作（这一行之前的代码）所发生的线程
    .observeOn(AndroidSchedulers.mainThread()) // observeOn() 负责指定回调所发生的线程
    .subscribe(new Observer<String>() {
        @Override
        public void onNext(String username) {
            userTextView.setText(username);
        }
        
        @Override
        public void onError(Throwable error) {
            // handle error
        }
        
        @Override
        public void onComplete() {
        }
    });
```

### 场景二： RxAndroid
RxAndroid 是 RxJava 的 Android 扩展，增加了一些 Android 的便利方法。目前我看到我们在用的有：

* `ViewObservable.clicks()`: 设置点击监听器，可以使用 `throttleFirst()` 来添加防手抖间隔。 
```java
ViewObservable.clicks(flipButton)
    .throttleFirst(500, TimeUnit.MILLISECONDS)
    .subscribe(new Action1<OnClickEvent>() {
    @Override
    public void call(OnClickEvent onClickEvent) {
        // do something
    }
});
```

* `WidgetObservable.text()`: 设置文字监听器。
```java
WidgetObservable.text(editTitle)
    .filter(new Func1<OnTextChangeEvent, Boolean>() {
        @Override
        public Boolean call(OnTextChangeEvent onTextChangeEvent) {
            return onTextChangeEvent.text() != null && onTextChangeEvent.text().length() > 0;
        }
    })
    .subscribe(new SubscriberAdapter<OnTextChangeEvent>() {
        @Override
        public void onError(Throwable e) {
            throw new RuntimeException(e);
        }

        @Override
        public void onNext(OnTextChangeEvent onTextChangeEvent) {
            // do something
        }
    });
```

### Observable 的创建和 subscribe 的细节

#### 创建 Observable
最常见的初始化 Observable 的方法有：

* `Observable.create(OnSubscribe)`: 手动创建 Observable 对象，自由控制 Observer 的回调。
* `Observable.just(T...)`: 将参数逐个通过调用 observer 的 `onNext()`发送出去，并在最后调用一次 `onComplete()`表示结束。
* `Observable.from(T[])`: 将数组拆开，逐个发送。

> 使用 Retrofit 的时候，不需要手动创建 Observable 。RxAndroid 也提供了一些便捷的绑定方法，例如上面出现过的点击监听和文字监听。

#### subscribe
* `Observable.subscribe(Observer)`: 将传入的 Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Action1 onNext)`: 将传入的 onNext 组装成 Observer 后，将组装的Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Action1 onNext, Action1 onError)`: 将传入的 onNext 和 onError 组装成 Observer 后，将组装的Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Action1 onNext, Action1 onError, Action0 onComplete)`: 将传入的 onNext, onError, onComplete 组装成 Observer 后，将组装的Observer 和 Observable 绑定，并激活 Observable。
* `Observable.subscribe(Subscriber)`: 和 `subscribe(Observer)` 一样。Subscriber 是 Observer 接口的一个实现，它同时实现了 Subscription 接口。实质上， `subscribe(Observer observer)` 的内部实现是用 observer 实现一个 Subscriber 后调用 `subscribe(Subscriber)`。

#### unsubscribe
unsubscribe 的原因有两种情况：

1. 不再需要。方式： `Subsription.unsubscribe()` (可以先通过 `Subscription.isUnsubscribed()` 检查一下)。
2. 防止内存泄露。方式：在合适的地方 (`Activity.onDestroy()`, `View.onDetachToWindow()`中）调用 `Subscription.subscribe()`， 或者使用** Flipboard 的 toolbox 中的便捷方法：`BindTransformer` ([示例](https://github.com/Flipboard/android/blob/cacf09a0b45b1682a9d79a7990f5ed3443af87b4/apps/flipboard/src/main/java/flipboard/gui/section/SectionFragment.java#L1282))**实现自动 unsubscribe.

### RxBus
RxBus 不是一个库，而是一种 RxJava 的使用方式。具体可以在[这里](https://github.com/Flipboard/android/issues/2302#issuecomment-118561719)看到 Zac 的介绍。
