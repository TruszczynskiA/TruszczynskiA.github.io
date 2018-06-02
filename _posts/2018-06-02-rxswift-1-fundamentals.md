---
layout: post
title: "RxSwift #1 - Fundamentals"
description: "A brief article about RxSwift fundamentals"
tags: [swift, rxswift]
---

We, as Swift developers, are spectators of a major change in IT world. Object-oriented programming (or OOP in short) is increasingly less popular nowadays. An advent of protocol-oriented and functional programming is just around the corner. Apple on their WWDC conference in 2016 said that Swift from version 3.0 will be a protocol-oriented language which will also allow to use functional programming methods. Almost 1.5 year earlier, ReactiveX started working on a Swift implementation of Rx. Rx is a reactive abstraction layer based on signals which emit changes to subscribers. Subscribers are notified about all changes so they can __react__ properly. Combining these two worlds - Reactive programming and functional programming resulted in functional reactive programming which is RxSwift right now. Interesting? I hope, I got your attention.😁 However, before you start coding using RxSwift, you must change your mindset about variables and functions. To help you to do this, I will explain the Rx fundamentals.

## Fundamentals

#### 1. Sequences

Sequences are the core of Rx. A single sequence is just a bundle of __events__ on a timeline. The sequences can be:

- terminated normally - e.g. after counting from 0 to 9:

{% highlight plain %}
--0-1-2-3-4-5-6-7-8-9--|
{% endhighlight %}

- terminated with an error - e.g. when the user enters a letter and you expected a digit:

{% highlight plain %}
--0-1-5-8-2-3-2-9-9-1--X
{% endhighlight %}

- infinite - e.g. a current `text` value in `UITextField`

{% highlight plain %}
--h-he-hel-hell-hello-->
{% endhighlight %}

Now, if you know how variables are handled in Rx, I can show you how to propagate it.

#### 2. ObservableTypes and Events

Sequences would be totally useless if nobody could track the changes on their timelines. Observable is a generic type which informs subscribers about changes/__events__. All Observables in RxSwift implement an `ObservableType` protocol which gives them access to various functionalities (which will be presented in a future blog post).

An event is an enum, which is a generic wrapper. Events are propagated by `ObservableTypes` and inform subscribers about one of three possible events:

- next - this event type contains a new value on a sequence.
- error - informs about the termination of sequence with an error. It contains an `Error` which can be handled by a subscriber.
- completion - informs about the termination of a sequence

`ObservableType` also informs subscriber(s) about the dispose state. More about disposing later.

There are plenty of types of `ObservableType` objects in RxSwift and supplementary frameworks (like RxCocoa or RxDataSources). However, I'll describe them in a separate blog post. Stay tuned!

#### 3. Subscription

Subscription is visible and the easiest to understand part of a reactive chain. Subscribers are entities which observe (listen) on a certain `ObservableType` and react to events which come to them. `ObservableType` has a variety of `subscribe` methods which are informative callback closures. Subscription is an easier to read, maintain, and more straight-forward method to pass data asynchronicity than its direct "rivals" (notifications, KVO, delegates). You can quickly check where the data comes from and who is the subscriber. It's a great way to make your code cleaner.

#### 4. Disposables and Dispose Bags

You Probably have some concerns about memory management. How is RxSwift able to handle retention between the observable and the subscriber? It's very easy. Every `subscribe` method returns a `Disposable` protocol type. This protocol has only one method `dispose()` which is implemented by every observer type in the framework. It contains a functionality which handles the whole deinitialization process. But how does each observer know when it's time to dispose itself? It's also very easy. A `dispose()` method will be called immediately after completion or an error event. But what if the subscriber will want to be deallocated before the sequence ends or an observer is connected to an infinite sequence? RxSwift provides us with a class called `DisposeBag` which has only two tasks. Firstly, it gathers references to disposables in an internal array. The second is to call the `dispose()` method on every disposable which is held in that array when the dispose bag is deallocated. In almost all cases the dispose bag is an internal variable of a subscriber and it's deallocated with it. To make a long story short, `DisposeBag` removes all connections when an object which holds this object is deallocated.

#### 5. Example

Now, after all the Rx fundamentals have been explained, let's create a simple demo which shows how all these things works.

First, lets create a view model and a view controller:

{% highlight swift %}
final class FundamentalsViewModel {

}
{% endhighlight %}

{% highlight swift %}
final class FundamentalsViewController: UIViewController {

    private let viewModel = FundamentalsViewModel()
}
{% endhighlight %}

Secondly, let's add some sequence in the view model which we will use in our example. We will put it temporarily into the `sequence()` method

{% highlight swift %}
private var value = 0

func sequence() {

    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] timer in

        guard let `self` = self else { return }

        self.value += 1
        guard self.value == 9 else { return }
        timer.invalidate()
    }
}
{% endhighlight %}

As you can see, this sequence will "count" from 0 to 9 with a second time interval. Ok, time for some meat. Let's wrap this sequence into an `ObservableType`. For this example, I used the simplest generic observable object called `Observable`. It's a read-only observer so it can't be manipulated by any external process.

{% highlight swift %}
final class FundamentalsViewModel {

    var counter: Observable<Int> {

        return Observable.create({ observer in

            Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] timer in

                guard let `self` = self else { return }

                self.value += 1
                observer.onNext(self.value)

                guard self.value == 9 else { return }

                timer.invalidate()
                observer.onCompleted()
            }

            return Disposables.create()
        })
    }

    private var value = 0
}
{% endhighlight %}

It's time to connect a subscriber. In our example, the view controller will take on this role. It will be using this value to simply print it on a debug console.

{% highlight swift %}
final class FundamentalsViewController: UIViewController {

    private let viewModel = FundamentalsViewModel()

    override func viewDidLoad() {
        super.viewDidLoad()

        viewModel.counter
            .subscribe(onNext: { value in
                print("New value -> \(value)")
            }, onCompleted: {
                print("Sequence completed 🎉")
            })
    }
}
{% endhighlight %}

Huzza, it's alive!. However, we must also add this subscription to the dispose bag to remove the connection with the view model when the view controller will be deallocated and the sequence will still be alive.

{% highlight swift %}
final class FundamentalsViewController: UIViewController {

    private let viewModel = FundamentalsViewModel()
    private let disposeBag = DisposeBag()

    override func viewDidLoad() {
        super.viewDidLoad()

        viewModel.counter
            .subscribe(onNext: { value in
                print("New value -> \(value)")
            }, onCompleted: {
                print("Sequence completed 🎉")
            })
            .disposed(by: disposeBag)
    }
}
{% endhighlight %}

That's all. We have created our first observable-observer(subscriber) pair. 🎉

Complete project is available [here](https://github.com/TruszczynskiA/RxSwiftExample).

## Afterword

As you can see, RxSwift is quite an easy way to simplify data management. However, it isn't a silver bullet and this convenience came at a cost. RxSwift is a very invasive framework which will be uneasy to remove from a project if you decide to do this.

I hope that this short article will help you understand the basics of RxSwift. Next time I will show you how the different types of `ObservableType` works.

Till then, keep coding! 👋🏻
