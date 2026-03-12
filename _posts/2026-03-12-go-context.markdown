---
layout: post
title:  "Cancellation in Go: the implementation behind context.Context"
categories: Go Golang
---

When it comes to concurrency programming in Go, the `context` package is as important as goroutines and channels nowadays. If you have ever been curious about what happens under the ubiquitous `ctx context.Context`, this post is for you. I'll try to answer these questions:

1. What problems the `context` package is trying to solve
2. How the Go team implements the `context` package

This is not a post about the usage of the `context` package. Since many examples and tutorials have already talked about it, I won’t cover it here.

## The problem: goroutine cancellation

![goroutine-tree](/asset/images/goroutine_tree.png)

In a typical Go program, the main goroutine starts subroutines for some tasks, and subroutines may also start subroutines, so at runtime, all the goroutines form a tree which traced back to the main goroutine. 

```go
func main(){
	wg := sync.WaitGroup()
	go func(){
		wg.Add(3)
		for i:= range 3{
			go backgroudTask(wg)
		}
	}
		
	wg.Wait()
}
```

In the pseudo-code above, we starts goroutines to perform background task then wait for them to finish. How do we end these goroutines early in order to contiune the execution of `main` function? 

Well, the short answer is we can't. Because Go doesn't provide a language-level mechanism to manage goroutines: Once a goroutine is started, unless it exits by itself, **no other goroutine can force it to exit**. (For a more concrete example of this pattern, see this Go blog [Pipelines](https://go.dev/blog/pipelines).)

This issue is most pronounced in web server programming, since request handlers often start additional goroutines to access resources. When a request needs to be abandoned, we want to release those goroutines as soon as possible.

To solve the problem, we need these behaviors:

1. When a goroutine is canceled, all its child goroutines also need to be canceled.
2. Canceling one goroutine shouldn't affect any of its ancestor goroutines; cancellation only propagates down the tree.

## The implementation details

To help us better understand the core design, we'll look at the [release-branch.go1.19](https://github.com/golang/go/blob/release-branch.go1.19/src/context/context.go)  version of the `context` package.

The package provides the `Context` interface, which exposes the following methods:

```go
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

All the methods only give us information about a context value — **we can't mutate a `Context` through its methods**, even the `Done` channel is receive-only. This design prevents subroutines from accidentally changing the ctx and affecting its ancestor, since the ctx is usually passed down by the ancestor.

### The `emptyCtx`

The package provides two methods for creating root context: `Background()` and `TODO()`. Both return a non-nil, empty `Context` that is never canceled, has no values and no deadline. `emptyCtx` is the underlying implementation.

```go
type emptyCtx int  
  
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {  
    return  
}  
  
func (*emptyCtx) Done() <-chan struct{} {  
    return nil  
}  
  
func (*emptyCtx) Err() error {  
    return nil  
}  
  
func (*emptyCtx) Value(key any) any {  
    return nil  
}
```

### The `cancelCtx`

Now let's take a close look at the more complex `cancelCtx`, which implement the cancellation mechanism.

```go
type canceler interface {  
    cancel(removeFromParent bool, err error)  
    Done() <-chan struct{}  
}

type cancelCtx struct {  
    Context  
  
    mu       sync.Mutex            // protects following fields  
    done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call    
    children map[canceler]struct{} // set to nil by the first cancel call    
    err      error                 // set to non-nil by the first cancel call
}
```

A `cancelCtx` has the following fields:

- `Context`: stores the parent context.
- `mu`: protects its private fields because a context can be shared across many goroutines.
- `done`: the channel that is closed when the context is canceled; it is stored in an `atomic.Value` to reduce lock contention.
- `children`: a map that stores registered child `cancelCtx`

You may notice that the key type of `children` is actually an interface called `canceler`. But in the `go1.19` implementation, `cancelCtx` is the only concrete implementation of `canceler`. A `cancelCtx` is created by calling `WithCancel`.

```go
type CancelFunc func()

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {  
    if parent == nil {  
       panic("cannot create context from nil parent")  
    }  
    c := newCancelCtx(parent)  
    propagateCancel(parent, &c)  
    return &c, func() { c.cancel(true, Canceled) }  
}
```

`WithCancel` returns a `cancelCtx` and a `CancelFunc`. You can call the `CancelFunc` to cancel the context, or the context will be canceled when its parent is canceled.

But why does cancellation of a parent cause cancellation of a child? The answer is in `propagateCancel`.

### How the cancellation signal is propagated

```go
func propagateCancel(parent Context, child canceler) {  
    done := parent.Done()  
    if done == nil {  
       return // parent is never canceled  
    }  
  
    select {  
    case <-done:  
       // parent is already canceled  
       child.cancel(false, parent.Err())  
       return  
    default:  
    }  
  
    if p, ok := parentCancelCtx(parent); ok {  
       p.mu.Lock()  
       if p.err != nil {  
          // parent has already been canceled  
          child.cancel(false, p.err)  
       } else {  
          if p.children == nil {  
             p.children = make(map[canceler]struct{})  
          }  
          p.children[child] = struct{}{}  
       }  
       p.mu.Unlock()  
    } else {  
       atomic.AddInt32(&goroutines, +1)  
       go func() {  
          select {  
          case <-parent.Done():  
             child.cancel(false, parent.Err())  
          case <-child.Done():  
          }  
       }()  
    }  
}
```

This function does the following:

1. Checks whether the parent can be canceled and whether it is already canceled.
2. Tries to find an ancestor that is a `cancelCtx`; if found, it registers the current child in that ancestor's `children` map.
3. If none of the ancestor is of type `cancelCtx`, it spawns a goroutine that listens for the parent's `Done` channel and cancels the child when that is canceled.

In short: cancellation is propagated either by actively checking the `children` map of a `cancelCtx` ancestor, or passively by spawning a goroutine that listens on the parent's `Done` channel.

With that in mind, let's review the `cancel` function.

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {  
    if err == nil {  
       panic("context: internal error: missing cancel error")  
    }  
    c.mu.Lock()  
    if c.err != nil {  
       c.mu.Unlock()  
       return // already canceled  
    }  
    c.err = err  
    d, _ := c.done.Load().(chan struct{})  
    if d == nil {  
       c.done.Store(closedchan)  
    } else {  
       close(d)  
    }  
    for child := range c.children {  
       // NOTE: acquiring the child's lock while holding parent's lock.  
       child.cancel(false, err)  
    }  
    c.children = nil  
    c.mu.Unlock()  
  
    if removeFromParent {  
       removeChild(c.Context, c)  
    }  
}
```

The core steps are to close the `done` channel (or store the `closedchan`, a shared pre-closed channel), cancel its registered children one by one, and finally detach itself from its parent.

With `cancelCtx`, we can efficiently propagate a cancellation signal to any goroutine that listens on its—or its children's—`Done` channel.

### The `timerCtx`

`timerCtx` extends `cancelCtx` with deadline/timeout semantics.

```go
type timerCtx struct {  
    cancelCtx  
    timer *time.Timer // Under cancelCtx.mu.  
  
    deadline time.Time  
}
```

By embedding `cancelCtx`, `timerCtx` can reuse cancellation-propagation logic. It can be created by calling `WithDeadline`.(or `WithTimeout`, which is just a wrapper function.)

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {  
    if parent == nil {  
       panic("cannot create context from nil parent")  
    }  
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {  
       // The current deadline is already sooner than the new one.  
       return WithCancel(parent)  
    }  
    c := &timerCtx{  
       cancelCtx: newCancelCtx(parent),  
       deadline:  d,  
    }  
    propagateCancel(parent, c)  
    dur := time.Until(d)  
    if dur <= 0 {  
       c.cancel(true, DeadlineExceeded) // deadline has already passed  
       return c, func() { c.cancel(false, Canceled) }  
    }  
    c.mu.Lock()  
    defer c.mu.Unlock()  
    if c.err == nil {  
       c.timer = time.AfterFunc(dur, func() {  
          c.cancel(true, DeadlineExceeded)  
       })  
    }  
    return c, func() { c.cancel(true, Canceled) }  
}
```

Apart from timer-related bookkeeping, its implementation is very similar to `WithCancel`.

### The `valueCtx`

Another use case for `Context` is value passing. It seems natural to do so, since a context will be passed around goroutines. 

However, in my opinion, context values are best suited for request-scoped metadata (authentication tokens, trace IDs), and they should be used with care, so you won't shoot yourself in the foot (the hunt for why the value is missing can be frustrating). In most cases, explicit over implicit is still the rule to follow, i.e., passing the value by parameter.

```go
type valueCtx struct {  
    Context  
    key, val any  
}

func WithValue(parent Context, key, val any) Context {  
    if parent == nil {  
       panic("cannot create context from nil parent")  
    }  
    if key == nil {  
       panic("nil key")  
    }  
    if !reflectlite.TypeOf(key).Comparable() {  
       panic("key is not comparable")  
    }  
    return &valueCtx{parent, key, val}  
}

func (c *valueCtx) Value(key any) any {  
    if c.key == key {  
       return c.val  
    }  
    return value(c.Context, key)  
}
```

`valueCtx` is straightforward: it stores the parent context, the key, and the value the user passed. Retrieving a value walks up the parent chain: when a key isn't found in the current `valueCtx`, the lookup continues to the parent until the root is reached.

## Summary

To recap: the `context` package solves the problem of goroutine cancellation, and also used  to pass request-scoped values between goroutines. A function that takes `ctx context.Context` should listen to its `Done` channel and abandon execution when that channel is closed.

When a context is canceled, its registered children are canceled too. The implementation achieves this by either:
1. registering children in a `cancelCtx` ancestor's `children` map (active propagation)
2. spawning a helper goroutine to wait on a parent's `Done` channel (passive listening). 

Both approaches let cancellation flow downward without affecting ancestors.