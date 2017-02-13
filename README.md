# RxJava 详解 #

### RxJava是什么 ###
一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库

### Rxjava有点 ###
RxJava的好处就在于它的简洁性，逻辑简单的时候看不出RxJava的优势，想必大家都知道在调度过程比较复杂的情况下，异步代码经常会既难写也难被读懂。这时候RxJava的优势就来了，随着程序逻辑变得越来越复杂，它依然能够保持简洁。

### API 介绍和原理简析 ###
#### 1. 概念：扩展的观察者模式 ####
RxJava 的异步实现，是通过一种扩展的观察者模式来实现的。

**RxJava 的观察者模式**

RxJava 有四个基本概念：Observable (可观察者，即被观察者)、 Observer (观察者)、 subscribe (订阅)、事件。Observable 和 Observer 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Observer。

与传统观察者模式不同， RxJava 的事件回调方法除了普通事件 onNext() （相当于 onClick() / onEvent()）之外，还定义了两个特殊的事件：onCompleted() 和 onError()。

onCompleted(): 事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。RxJava 规定，当不会再有新的 onNext() 发出时，需要触发 onCompleted() 方法作为标志。
onError(): 事件队列异常。在事件处理过程中出异常时，onError() 会被触发，同时队列自动终止，不允许再有事件发出。
在一个正确运行的事件序列中, onCompleted() 和 onError() 有且只有一个，并且是事件序列中的最后一个。需要注意的是，onCompleted() 和 onError() 二者也是互斥的，即在队列中调用了其中一个，就不应该再调用另一个。

#### 2. 基本实现 ####
基于以上的概念， RxJava 的基本实现主要有三点：

**1) 创建 Observer**

Observer 即观察者，它决定事件触发的时候将有怎样的行为。 RxJava 中的 Observer 接口的实现方式：

    Observer<String> observer = new Observer<String>() {
        @Override
        public void onCompleted() {
            Log.i(TAG,"onCompleted");
        }
        @Override
        public void onError(Throwable e) {
            Log.i(TAG,"onError");
        }
        @Override
        public void onNext(String string) {
            Log.i(TAG,string);
        }
    };

除了 Observer 接口之外，RxJava 还内置了一个实现了 Observer 的抽象类：Subscriber。 Subscriber 对 Observer 接口进行了一些扩展，但他们的基本使用方式是完全一样的：

    Subscriber<String> subscriber = new Subscriber<String>() {
        @Override
        public void onCompleted() {
        }
        @Override
        public void onError(Throwable e) {
        }
        @Override
        public void onStart() {
            super.onStart();
        }
        @Override
        public void onNext(String o) {
        }
    };

不仅基本使用方式一样，实质上，在 RxJava 的 subscribe 过程中，Observer 也总是会先被转换成一个 Subscriber 再使用。所以如果你只想使用基本功能，选择 Observer 和 Subscriber 是完全一样的。它们的区别对于使用者来说主要有两点：

onStart(): 这是 Subscriber 增加的方法。它会在 subscribe 刚开始，而事件还未发送之前被调用，可以用于做一些准备工作，例如数据的清零或重置。这是一个可选方法，默认情况下它的实现为空。需要注意的是，如果对准备工作的线程有要求（例如弹出一个显示进度的对话框，这必须在主线程执行）， onStart() 就不适用了，因为它总是在 subscribe 所发生的线程被调用，而不能指定线程。要在指定的线程来做准备工作，可以使用 doOnSubscribe() 方法，具体可以在后面的文中看到。
unsubscribe(): 这是 Subscriber 所实现的另一个接口 Subscription 的方法，用于取消订阅。在这个方法被调用后，Subscriber 将不再接收事件。一般在这个方法调用前，可以使用 isUnsubscribed() 先判断一下状态。 unsubscribe() 这个方法很重要，因为在 subscribe() 之后， Observable 会持有 Subscriber 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：要在不再使用的时候尽快在合适的地方（例如 onPause() onStop() 等方法中）调用 unsubscribe() 来解除引用关系，以避免内存泄露的发生。


**2) 创建 Observable**

Observable 即被观察者，它决定什么时候触发事件以及触发怎样的事件。 RxJava 使用 create() 方法来创建一个 Observable ，并为它定义事件触发规则：

    Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            subscriber.onNext("Hello");
            subscriber.onNext("Hi");
            subscriber.onNext("Aloha");
            subscriber.onCompleted();
        }
    });

可以看到，这里传入了一个 OnSubscribe 对象作为参数。OnSubscribe 会被存储在返回的 Observable 对象中，它的作用相当于一个计划表，当 Observable 被订阅的时候，OnSubscribe 的 call() 方法会自动被调用，事件序列就会依照设定依次触发（对于上面的代码，就是观察者Subscriber 将会被调用三次 onNext() 和一次 onCompleted()）。这样，由被观察者调用了观察者的回调方法，就实现了由被观察者向观察者的事件传递，即观察者模式。

create() 方法是 RxJava 最基本的创造事件序列的方法。基于这个方法， RxJava 还提供了一些方法用来快捷创建事件队列，例如：



**just(T...): 将传入的参数依次发送出来**

    Observable observable = Observable.just("Hello", "Hi", "Aloha");
	// 将会依次调用：
	// onNext("Hello");
	// onNext("Hi");
	// onNext("Aloha");
	// onCompleted();

**from(T[]) / from(Iterable<? extends T>) : 将传入的数组或 Iterable 拆分成具体对象后，依次发送出来。**

    String[] words = {"Hello", "Hi", "Aloha"};
	Observable observable = Observable.from(words);
	// 将会依次调用：
	// onNext("Hello");
	// onNext("Hi");
	// onNext("Aloha");
	// onCompleted();

上面 just(T...) 的例子和 from(T[]) 的例子，都和之前的 create(OnSubscribe) 的例子是等价的。

**3) Subscribe (订阅)**

创建了 Observable 和 Observer 之后，再用 subscribe() 方法将它们联结起来，整条链子就可以工作了。代码形式很简单：

    observable.subscribe(observer);
	// 或者：
	observable.subscribe(subscriber);

> 有人可能会注意到， subscribe() 这个方法有点怪：它看起来是『observalbe 订阅了 observer / subscriber』而不是『observer / subscriber 订阅了 observalbe』，这看起来就像『杂志订阅了读者』一样颠倒了对象关系。这让人读起来有点别扭，不过如果把 API 设计成 observer.subscribe(observable) / subscriber.subscribe(observable) ，虽然更加符合思维逻辑，但对流式 API 的设计就造成影响了，比较起来明显是得不偿失的。

Observable.subscribe(Subscriber) 的内部实现是这样的（仅核心代码）：

    // 注意：这不是 subscribe() 的源码，而是将源码中与性能、兼容性、扩展性有关的代码剔除后的核心代码。
	// 如果需要看源码，可以去 RxJava 的 GitHub 仓库下载。
	public Subscription subscribe(Subscriber subscriber) {
    subscriber.onStart();
    onSubscribe.call(subscriber);
    return subscriber;
	}

可以看到，subscriber() 做了3件事：

1.调用 Subscriber.onStart() 。这个方法在前面已经介绍过，是一个可选的准备方法。

2.调用 Observable 中的 OnSubscribe.call(Subscriber) 。在这里，事件发送的逻辑开始运行。从这也可以看出，在 RxJava 中， Observable 并不是在创建的时候就立即开始发送事件，而是在它被订阅的时候，即当 subscribe() 方法执行的时候。

3.将传入的 Subscriber 作为 Subscription 返回。这是为了方便 unsubscribe().

**这部分的代码可以在工程中RxjavaDemoAccidenceFirst看到**

----------



除了 subscribe(Observer) 和 subscribe(Subscriber) ，subscribe() 还支持不完整定义的回调，RxJava 会自动根据定义创建出 Subscriber 。形式如下：

    Action1<String> onNextAction = new Action1<String>() {
	    // onNext()
	    @Override
	    public void call(String s) {
		        Log.d(tag, s);
		}
	};
	Action1<Throwable> onErrorAction = new Action1<Throwable>() {
	    // onError()
	    @Override
	    public void call(Throwable throwable) {
		        // Error handling
		}
	};
	Action0 onCompletedAction = new Action0() {
	    // onCompleted()
	    @Override
	    public void call() {
		        Log.d(tag, "completed");
		}
	};
	// 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
	observable.subscribe(onNextAction);
	// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
	observable.subscribe(onNextAction, onErrorAction);
	// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
	observable.subscribe(onNextAction, onErrorAction, onCompletedAction);

简单解释一下这段代码中出现的 Action1 和 Action0。 Action0 是 RxJava 的一个接口，它只有一个方法 call()，这个方法是无参无返回值的；由于 onCompleted() 方法也是无参无返回值的，因此 Action0 可以被当成一个包装对象，将 onCompleted() 的内容打包起来将自己作为一个参数传入 subscribe() 以实现不完整定义的回调。这样其实也可以看做将 onCompleted() 方法作为参数传进了 subscribe()，相当于其他某些语言中的『闭包』。 Action1 也是一个接口，它同样只有一个方法 call(T param)，这个方法也无返回值，但有一个参数；与 Action0 同理，由于 onNext(T obj) 和 onError(Throwable error) 也是单参数无返回值的，因此 Action1 可以将 onNext(obj) 和 onError(error) 打包起来传入 subscribe() 以实现不完整定义的回调。事实上，虽然 Action0 和 Action1 在 API 中使用最广泛，但 RxJava 是提供了多个 ActionX 形式的接口 (例如 Action2, Action3) 的，它们可以被用以包装不同的无返回值的方法。

> 简单的说可以把一个Action当作的是一个观察者（Observer）来看，Action1中的call方法因为有参数（如果参数是Throwable则相当与观察者的onError方法也就是只会执行onError方法，否则相当于观察者的onNext方法也只会执行onNext方法），Action0中的call方法没有参数可以看作的观察者的onCompleted方法。

**下面举两个例子：**

**a. 打印字符串数组**

将字符串数组 names 中的所有字符串依次打印出来：

    String[] names = {"Tom","Jack","Kobe"};
    Observable.from(names).subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
           Log.i(TAG,s);
           }
    });

**b. 由 id 取得图片并显示**

由指定的一个 drawable 文件 id drawableRes 取得图片，并显示在 ImageView 中，并在出现异常的时候打印 Toast 报错：

    	final int drawableRes = R.mipmap.ic_launcher;
        Observable.create(new Observable.OnSubscribe<Drawable>() {
            @Override
            public void call(Subscriber<? super Drawable> subscriber) {
                Drawable drawable = getTheme().getDrawable(drawableRes);
                subscriber.onNext(drawable);
                subscriber.onCompleted();
            }
            }).subscribe(new Observer<Drawable>() {
                @Override
                public void onCompleted() {
                }

                @Override
                public void onError(Throwable e) {
                    Toast.makeText(getActivity(), "Error!", Toast.LENGTH_SHORT).show();
                }

                @Override
                public void onNext(Drawable drawable) {
                    image_logo.setImageDrawable(drawable);
                }
            });

正如上面两个例子这样，创建出 Observable 和 Subscriber ，再用 subscribe() 将它们串起来，一次 RxJava 的基本使用就完成了。非常简单。

**这部分的代码可以在工程中RxjavaDemoAccidenceSecond中看到**

----------

#### 线程控制 —— Scheduler (一) ####

在不指定线程的情况下， RxJava 遵循的是线程不变的原则，即：在哪个线程调用 subscribe()，就在哪个线程生产事件；在哪个线程生产事件，就在哪个线程消费事件。如果需要切换线程，就需要用到 Scheduler （调度器）。

**1) Scheduler 的 API (一)**
在RxJava 中，Scheduler ——调度器，相当于线程控制器，RxJava 通过它来指定每一段代码应该运行在什么样的线程。RxJava 已经内置了几个 Scheduler ，它们已经适合大多数的使用场景：

- Schedulers.immediate(): 直接在当前线程运行，相当于不指定线程。这是默认的 Scheduler。
- Schedulers.newThread(): 总是启用新线程，并在新线程执行操作。
- Schedulers.io(): I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。
- Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
- 另外， Android 还有一个专用的 AndroidSchedulers.mainThread()，它指定的操作将在 Android 主线程运行。

有了这几个 Scheduler ，就可以使用 subscribeOn() 和 observeOn() 两个方法来对线程进行控制了。 * subscribeOn(): 指定 subscribe() 所发生的线程（即订阅时的线程），即 Observable.OnSubscribe 被激活时所处的线程。或者叫做事件产生的线程。 * observeOn(): 指定 Subscriber 所运行在的线程（即观察者的线程）。或者叫做事件消费的线程。

文字叙述总归难理解，看代码：

    Observable.just(1,2,3,4)
                        .subscribeOn(Schedulers.io())// 指定 subscribe() 发生在 IO 线程
                        .observeOn(AndroidSchedulers.mainThread())// 指定 Subscriber 的回调发生在主线程
                        .subscribe(new Action1<Integer>() {
                            @Override
                            public void call(Integer integer) {
                                Log.i(TAG,integer+"");//这里是主线程
                            }
                        });

上面这段代码中，由于 subscribeOn(Schedulers.io()) 的指定，被创建的事件的内容 1、2、3、4 将会在 IO 线程发出；而由于 observeOn(AndroidScheculers.mainThread()) 的指定，因此 subscriber 数字的打印将发生在主线程 。事实上，这种在 subscribe() 之前写上两句 subscribeOn(Scheduler.io()) 和 observeOn(AndroidSchedulers.mainThread()) 的使用方式非常常见，它适用于多数的 『后台线程取数据，主线程显示』的程序策略。

而前面提到的由图片 id 取得图片并显示的例子，如果也加上这两句：

    Observable.create(new Observable.OnSubscribe<Drawable>() {
                    @RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
                    @Override
                    public void call(Subscriber<? super Drawable> subscriber) {
                        //这部分会执行在子线程（IO线程）
                        Drawable drawable = getTheme().getDrawable(drawableRes);
                        subscriber.onNext(drawable);
                        subscriber.onCompleted();
                    }
                }) .subscribeOn(Schedulers.io())// 指定 subscribe() 发生在 IO 线程
                    .observeOn(AndroidSchedulers.mainThread())// 指定 Subscriber 的回调发生在主线程
                    .subscribe(new Subscriber<Drawable>() {
                        @Override
                        public void onCompleted() {
                            
                        }
                        @Override
                        public void onError(Throwable e) {
                            
                        }
                        @Override
                        public void onNext(Drawable drawable) {
                            image_scheduler.setImageDrawable(drawable);
                        }
                    });
那么，加载图片将会发生在 IO 线程，而设置图片则被设定在了主线程。这就意味着，即使加载图片耗费了几十甚至几百毫秒的时间，也不会造成丝毫界面的卡顿。

**2) Scheduler 的原理 (一)**



#### 4. 变换 ####
RxJava 提供了对事件序列进行变换的支持，这是它的核心功能之一，也是大多数人说『RxJava 真是太好用了』的最大原因。**所谓变换，就是将事件序列中的对象或整个序列进行加工处理，转换成不同的事件或事件序列。**概念说着总是模糊难懂的，来看 API。

**1) API**

首先假设这么一种需求：假设有一个数据结构『学生』，现在需要打印出一组学生的名字。实现方式很简单：

    Subscriber<String> subscriber = new Subscriber<String>() {
                    @Override
                    public void onCompleted() {
                    }

                    @Override
                    public void onError(Throwable e) {
                    }

                    @Override
                    public void onNext(String s) {
                        Log.i(TAG,s);
                    }
                };

                Observable.from(students)
                        .map(new Func1<Student, String>() {
                            @Override
                            public String call(Student student) {
                                return student.name;
                            }
                        })
                        .subscribe(subscriber);

这里出现了一个叫做 Func1 的类。它和 Action1 非常相似，也是 RxJava 的一个接口，用于包装含有一个参数的方法。 Func1 和 Action 的区别在于， Func1 包装的是有返回值的方法。另外，和 ActionX 一样， FuncX 也有多个，用于不同参数个数的方法。FuncX 和 ActionX 的区别在 FuncX 包装的是有返回值的方法。

可以看到，map() 方法将参数中的 Student 对象转换成一个 String 对象后返回，而在经过 map() 方法后，事件的参数类型也由 Student 转为了 String。这种直接变换对象并返回的，是最常见的也最容易理解的变换。不过 RxJava 的变换远不止这样，它不仅可以针对事件对象，还可以针对整个事件队列，这使得 RxJava 变得非常灵活。我列举几个常用的变换：

- flatMap(): 那么再假设：如果要打印出每个学生所需要修的所有课程的名称呢？（需求的区别在于，每个学生只有一个名字，但却有多个课程。）首先可以这样实现：


			Subscriber<Student> _subscriber = new Subscriber<Student>() {
                @Override
                public void onCompleted() {
                    Log.i(TAG, "onCompleted");
                }

                @Override
                public void onError(Throwable e) {
                }

                @Override
                public void onNext(Student student) {
                    List<Course> courses = student.courseList;
                    for (int i = 0; i < courses.size(); i++) {
                        Course course = courses.get(i);
                        Log.i(TAG, course.courseName);
                    }
                }
            };
            Observable.from(students).subscribe(_subscriber);

依然很简单。那么如果我不想在 Subscriber 中使用 for 循环，而是希望 Subscriber 中直接传入单个的 Course 对象呢（这对于代码复用很重要）？用 map() 显然是不行的，因为 map() 是一对一的转化，而我现在的要求是一对多的转化。那怎么才能把一个 Student 转化成多个 Course 呢？

这个时候，就需要用 flatMap() 了：

    Subscriber<Course> subscriber1 = new Subscriber<Course>() {
                    @Override
                    public void onNext(Course course) {
                        Log.i(TAG, course.courseName);
                    }
                    @Override
                    public void onCompleted() {
                        Log.i(TAG, "onCompleted");
                    }

                    @Override
                    public void onError(Throwable e) {
                    }
                };
                Observable.from(students)
                        .flatMap(new Func1<Student, Observable<Course>>() {
                            @Override
                            public Observable<Course> call(Student student) {
                                return Observable.from(student.courseList);
                            }
                        })
                        .subscribe(subscriber1);

从上面的代码可以看出， flatMap() 和 map() 有一个相同点：它也是把传入的参数转化之后返回另一个对象。但需要注意，和 map() 不同的是， flatMap() 中返回的是个 Observable 对象，并且这个 Observable 对象并不是被直接发送到了 Subscriber 的回调方法中。 flatMap() 的原理是这样的：

1. 使用传入的事件对象创建一个 Observable 对象；
2. 并不发送这个 Observable, 而是将它激活，于是它开始发送事件；
3. 每一个创建出来的 Observable 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 的回调方法。这三个步骤，把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 flatMap() 所谓的 flat。

