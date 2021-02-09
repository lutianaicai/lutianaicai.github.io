---
title: PromiseKit 源码分析
date: 2019-03-31
categories: iOS
tags: 源码分析
---

面对回调地狱 [PromiseKit](https://github.com/mxcl/PromiseKit) 提供了一种简洁易用的异步编程模式。让你可以编写出更加易读，更加专注结果的代码。本文意在探寻简介背后的逻辑。

## 从最小类型谈去
搞清复杂封装的源码中最基本的类型就好比搞清一篇英文文章中所有看不懂的生词的意思。再想看懂文章，只需要把已知信息串联起来就行了。

### Box
`box` 是 `PromiseKit` 中基本的类，在介绍它之前，先得说一下同样定义在Box.swift中的另外两个更小的类型 `Sealant` 和 `Handlers`

```swift
enum Sealant<R> {
    case pending(Handlers<R>)
    case resolved(R)
}

final class Handlers<R> {
    var bodies: [(R) -> Void] = []
    func append(_ item: @escaping(R) -> Void) { bodies.append(item) }
}
```

`Handlers<R>` 用于保存 `Promise` 的结果,`bodies` 属性记录了所有可能会接受 `Promise` 返回值的方法，例如 `.then` `.done`。
`Sealant` 则表征Box的状态，`.pending` 表示还在等待结果，`.resolved` 表示已经接受了结果

说完这两个类型，下面进入正题，开始介绍 `Box`

#### 基类Box

首先从 `Box` 基类讲起

```swift
class Box<T> {
  func inspect() -> Sealant<T> { fatalError() }
  func inspect(_: (Sealant<T>) -> Void) { fatalError() }
  func seal(_: T) {}
}
```

它接受一个范型参数，约束了三个方法，但是其实这三个方法都近乎空实现

```swift
func inspect() -> Sealant<T> { fatalError() }
```

我们根据返回值可以查询目前 `box` 的状态，是 `.pending` 还是 `.resolved`

```swift
func inspect(_: (Sealant<T>) -> Void) { fatalError() }
```

则是根据状态的不同选择不同的逻辑

```swift
func seal(_: T) {}
```

是把值放入箱子然后封上。具体实现我们还得看 `Box` 的两个派生类 `SealedBox` 和 `EmptyBox`

#### SealedBox

翻译过来就是 `封上了的箱子`，看代码

```swift
final class SealedBox<T>: Box<T> {
  let value: T

  init(value: T) {
    self.value = value
  }

  override func inspect() -> Sealant<T> {
    return .resolved(value)
  }
}
```

可以看出，创建封好的箱子时就要往箱子里放入一个 `Promise` 的结果，而且放过以后 `Box` 的状态就为 `.resolved` 了且值也不能修改了。

#### EmptyBox

空箱 `EmptyBox` 是 `Box` 类中最复杂也是最重要的派生类，先看属性

```swift
private var sealant = Sealant<T>.pending(.init())
private let barrier = DispatchQueue(label: "org.promisekit.barrier", attributes: .concurrent)
```

其中：
* `sealant` 表示 `Box` 的状态，初始值是 `.pending`，关联一个空的 `Handler`
* `barrier` 是一个异步队列。内部通过这个队列尽心状态与读写的同步。

接下来看获取箱子状态的 `inspect` 方法

```swift
override func inspect() -> Sealant<T> {
        var rv: Sealant<T>!
        barrier.sync {
            rv = self.sealant
        }
        return rv
    }
```

很简单，`barrier` 中获取这个`sealant` 并返回

再看设置状态的 `inspect` 方法

```swift
override func inspect(_ body: (Sealant<T>) -> Void) {
        var sealed = false
        barrier.sync(flags: .barrier) {
            switch sealant {
            case .pending:
                // body will append to handlers, so we must stay barrier’d
                body(sealant)
            case .resolved:
                sealed = true
            }
        }
        if sealed {
            // we do this outside the barrier to prevent potential deadlocks
            // it's safe because we never transition away from this state
            body(sealant)
        }
    }
```

它同步判断了箱子的状态，如果是 `.pending` 就调用 `body` 参数，如果是 `.resolved` 就把箱子的密封状态设置成 `true` 然后再屌用 `body` 参数

最后再看封箱方法 `seal`

```swift
override func seal(_ value: T) {
        var handlers: Handlers<T>!
        barrier.sync(flags: .barrier) {
            guard case .pending(let _handlers) = self.sealant else {
                return  
            }
            handlers = _handlers
            self.sealant = .resolved(value)
        }

        if let handlers = handlers {
            handlers.bodies.forEach{ $0(value) }
        }

    }
```

首先在 `barries` 中同步判断，箱子如果封上了就直接顺序调用所有 `handler`，如果还是 `.pending` 状态就修改状态至 `.resolved` 并且修改箱子里的值为`value`,然后再顺序调用所有的 `handler`

### Resolver

`resolver` 用于处理 `promise` 中`closure` 中的结果，先看基础类型 `Result`

```swift
public enum Result<T> {
    case fulfilled(T)
    case rejected(Error)
}
```

很简单，promise 结果达成就 fulfilled 结果，发生错误就 rejected错误。这个作法跟 `Box` 中的 `sealant` 很像。

接着看 Resolver 实现

```swift
public final class Resolver<T> {
    let box: Box<Result<T>>

    init(_ box: Box<Result<T>>) {
        self.box = box
    }

    deinit {
        if case .pending = box.inspect() {
            conf.logHandler(.pendingPromiseDeallocated)
        }
    }
}
```

这里发现，它包含一个 `Box` 属性，在 `init` 时会注入一个封装了 `Result` 的 `Box`，在 `deinit` 时如果注入的 `box` 状态是 `.pending` 就会警告 `Resolver` 被提前释放了。

#### Extension

`resolver` 结构简单，功能基本是`extension` 实现

```swift
public extension Resolver {
    /// Fulfills the promise with the provided value
    func fulfill(_ value: T) {
        box.seal(.fulfilled(value))
    }

    /// Rejects the promise with the provided error
    func reject(_ error: Error) {
        box.seal(.rejected(error))
    }

    /// Resolves the promise with the provided result
    func resolve(_ result: Result<T>) {
        box.seal(result)
    }

    /// Resolves the promise with the provided value or error
    func resolve(_ obj: T?, _ error: Error?) {
        if let error = error {
            reject(error)
        } else if let obj = obj {
            fulfill(obj)
        } else {
            reject(PMKError.invalidCallingConvention)
        }
    }

    /// Fulfills the promise with the provided value unless the provided error is non-nil
    func resolve(_ obj: T, _ error: Error?) {
        if let error = error {
            reject(error)
        } else {
            fulfill(obj)
        }
    }

    /// Resolves the promise, provided for non-conventional value-error ordered completion handlers.
    func resolve(_ error: Error?, _ obj: T?) {
        resolve(obj, error)
    }
}
```

可以看出，无论是接收 value、error 还是 Result 都是给内部的 box 赋值。注释很清晰，不过多赘述。

## Promise

之前的分析就把 PromiseKit 中的基本类型了解了。“单词”我们已经掌握了，下面试着翻译翻译“段落”。

首先看 Promise 的初始化方法的实现：

```swift
public final class Promise<T>: Thenable, CatchMixin {
    let box: Box<Result<T>>

    fileprivate init(box: SealedBox<Result<T>>) {
        self.box = box
    }
    
    public class func value(_ value: T) -> Promise<T> {
        return Promise(box: SealedBox(value: .fulfilled(value)))
    }

    /// Initialize a new rejected promise.
    public init(error: Error) {
        box = SealedBox(value: .rejected(error))
    }

    /// Initialize a new promise bound to the provided `Thenable`.
    public init<U: Thenable>(_ bridge: U) where U.T == T {
        box = EmptyBox()
        bridge.pipe(to: box.seal)
    }

    /// Initialize a new promise that can be resolved with the provided `Resolver`.
    public init(resolver body: (Resolver<T>) throws -> Void) {
        box = EmptyBox()
        let resolver = Resolver(box)
        do {
            try body(resolver)
        } catch {
            resolver.reject(error)
        }
    }
}
```

首先是 `Promise` 也持有一个 `box` 属性，第一个 `init` 方法是一个文件私有方法，为属性赋值。后两个方法是接收 `value` 就封箱然后 `fulfilled` 出去，接收 `error` 就 `rejected` 出去没有啥好说的。

第三个绑定 `Thenable` 的一会再说，第四个方法注入了一个异步过程。`body` 参数支持我们分别调用 `fulfill` 和 `reject`，然后进行 `box.seal` 方法去通知所有关注这个值的 handlers。

说完初始化过程，再来说说 Promises 是如何链接的。

## Thenable

首先 Theanble 是一个协议，它约定了以下三个内容：

```swift
public protocol Thenable: class {
    /// The type of the wrapped value
    associatedtype T

    /// `pipe` is immediately executed when this `Thenable` is resolved
    func pipe(to: @escaping(Result<T>) -> Void)

    /// The resolved result or nil if pending.
    var result: Result<T>? { get }
}
```

我们由浅入深来看，首先是关联类型 `T`，在协议被采用时才会被指定，也就是 `Result` 中包含的类型。

其次是 `result`，这个对应实现在 `Promise` 里：

```swift
public var result: Result<T>? {
        switch box.inspect() {
        case .pending:
            return nil
        case .resolved(let result):
            return result
        }
    }
```

其实就是从封好的箱子里取值，如果还是 `.pending` 状态就返回 `nil`

### pipe

先看 `pipe` 在 `Promise` 中的实现：

```swift
public func pipe(to: @escaping(Result<T>) -> Void) {
        switch box.inspect() {
        case .pending:
            box.inspect {
                switch $0 {
                case .pending(let handlers):
                    handlers.append(to)
                case .resolved(let value):
                    to(value)
                }
            }
        case .resolved(let value):
            to(value)
        }
    }
```

可以看到先是读取了当前 Promise 盒子的状态，如果封上了就把值传给 pipe 中的 closure，如果是 .pending 状态就加入到 box 中的 handlers

再看 then 中的 pipe 到底做了什么

```swift
func then<U: Thenable>(on: DispatchQueue? = conf.Q.map, flags: DispatchWorkItemFlags? = nil, _ body: @escaping(T) throws -> U) -> Promise<U.T> {
        let rp = Promise<U.T>(.pending)
        pipe {
            switch $0 {
            case .fulfilled(let value):
                on.async(flags: flags) {
                    do {
                        let rv = try body(value)
                        guard rv !== rp else { throw PMKError.returnedSelf }
                        rv.pipe(to: rp.box.seal)
                    } catch {
                        rp.box.seal(.rejected(error))
                    }
                }
            case .rejected(let error):
                rp.box.seal(.rejected(error))
            }
        }
        return rp
    }
```

首先关注一下这个 `body` 参数，他是在 `Promise` 对象之后串联的 `closure`，这个 `closure` 一封装的值为参数返回一个新的遵循 `Thenable` 的对象。

了解这个以后，那就能够了解用作返回值的 `Promise<U.T>` 了

```swift
let rp = Promise<U.T>(.pending)
```

然后检查 `Result` 中是否有错误，有就封装 `rejected` 否则就异步执行一段 `closure`，而这里的关键就是

```swift
rv.pipe(to: rp.box.seal)
```

是啥意思呢，就是把 `then` 中 `closure`

执行成功封装的值交给返回值 `rp` 的 `box`，这也是异步过程能够链式调用的关键。

## 总结

从头梳理一遍 `Promise` 的工作流程。

首先调用 `Promise.init` 方法：

```swift
/// Initialize a new promise that can be resolved with the provided `Resolver`.
    public init(resolver body: (Resolver<T>) throws -> Void) {
        box = EmptyBox()
        let resolver = Resolver(box)
        do {
            try body(resolver)
        } catch {
            resolver.reject(error)
        }
    }
```

在 `init` 方法里，创建了一个 `EmptyBox`。这个 `EmptyBox` 自带一个关联的 `handlers` 数组，一个同步队列。

用 `EmptyBox` 创建一个 `Resolver` 用于处理 `closure` 中的结果。

```swift
try body(resolver)
```

调用 `Promise` 中的 `closure`，如果有异常就调用 `resolver.reject`。`closure` 中的方法成功就 `fulfill`，错误就 `reject`

无论是 `fulfill` 还是 `reject` 其实都是把 `value` 或 `error` 封装进 `box`

然后调用 seal 方法

```swift
override func seal(_ value: T) {
        var handlers: Handlers<T>!
        barrier.sync(flags: .barrier) {
            guard case .pending(let _handlers) = self.sealant else {
                return  
            }
            handlers = _handlers
            self.sealant = .resolved(value)
        }

        if let handlers = handlers {
            handlers.bodies.forEach{ $0(value) }
        }
    }
```

如果盒子还没封上就把值封装进去然后依次调用处理这个期望值的 `handlers`，这个就是 `Promise` 工作的大概流程。

本文分析思路有借鉴 [泊学](https://boxueio.com/) 的地方，在此附上官网链接。

如果你有发现任何错误，也欢迎您的赐教。


