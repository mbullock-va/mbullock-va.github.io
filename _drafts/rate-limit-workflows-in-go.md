---
layout: default
title:  "Rate Limiting Workflows in Go"
date:   2017-07-29 11:38:22 -0600
categories: jekyll update
---
# Rate Limiting Workflows in Go

The simplicity of Go's concurrency paradigms contribute to making it an extremely powerful language. But with great power comes great responsibility, right? Spinning up to many go routines to quickly can easily overwhelm your upstream resources which may result in catastrophic outages.

Consider the following example:
```
for i := range items {
    go doWork(items[i])
}
```

This loop iterates over all the items in a slice and does some work for each item concurrently. For a short slice, with a reasonable implementation of `doWork` this is unlikely to be a problem, but as the slice gets longer there is a greater risk of overwhelming upstream resources. For example, if the items were URLs and the `doWork` function does a URL fetch, next thing you know you have inadvertently DOS'd a website by doing thousands of fetches concurrently. Something similar could happen to your database if `doWork` was querying for data rather than doing a URL fetch. Let's go though some examples that show how we can effectively scale back workflows in Go.

## No Rate Limiting
First things first, let's establish a starting point with code that actually runs. You will find Go Playground links for the full implementation that you can run in your browser below each code snippet.

Here is code without any rate limiting:
```
package main

import (
	"fmt"
	"time"
	"sync/atomic"
	"sync"
)

var numConcurrent int64 = 0

func doWork(wg *sync.WaitGroup) {
	routineStarting()
	defer routineDone(wg)
	processTime(time.Second*3)
}

func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)

	for i := 0; i<20; i++ {
		wg.Add(1)
		go doWork(&wg)
	}
}

// ----------------------
// Helper functions below
// ----------------------

func mainDone(start time.Time, wg *sync.WaitGroup) {
	wg.Wait()
	took := time.Since(start)
	fmt.Printf("Finished, took: %v\n", took)
}

func mainStarting() time.Time {
	fmt.Printf("Main is starting\n")
	return time.Now()
}

func routineStarting() {
	atomic.AddInt64(&numConcurrent, 1)
	fmt.Printf("Active Goroutines: %d\n", numConcurrent)
}

func routineDone(wg *sync.WaitGroup) {
	wg.Done()
	atomic.AddInt64(&numConcurrent, -1)
	fmt.Printf("Active Goroutines: %d\n", numConcurrent)
}

func processTime(d time.Duration) {
	time.Sleep(d)
}
```
Run the code here: [Go Playground](https://play.golang.org/p/wv5QRUtNzZ)

This snippet loops 20 times, and fires off a go routine to do some work, we call this function `doWork` and in this case it does 3 seconds of "processing". This code is unlikely to be a concern for any production server for any reasonable implementation of `doWork`. But change 20 to 1000000 and upstream dependencies like a database could crumble. Note that these are simplified examples, `doWork` does not take any arguments to actually do work. Also, the wait group is simply there to prevent the main function from finishing before all the "work" is done. It is not used to rate limit the work.

Let's get to solving the problem.

## Throttling with Worker Pools
This first example helps reduce the throughput by using a pool of worker go routines. This strategy ensures that only a certain number of `doWork` calls can be happening concurrently.

```
func worker(wg *sync.WaitGroup, jobs chan struct{}) {
	for range jobs {
		doWork(wg)
	}
}

func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)

	jobs := make(chan struct{}, 3)
	defer close(jobs)

	// start 3 workers
	for i := 0; i<3; i++ {
		go worker(&wg, jobs)
	}

	for i := 0; i<20; i++ {
		wg.Add(1)
		jobs <- struct{}{}
	}
}
```
Run the code here: [Go Playground](https://play.golang.org/p/Cs_kZ1POJ3)

The code's main function starts by defining a channel called `jobs`. This channel will coordinate the work that the main function needs to be done (written to the channel), for the workers do (read from the channel). Next we have a loop that starts 3 go routines, each run the `worker` function to do the work. The `worker` function ranges over the jobs channel, pulling a job off the channel then does work for that job, it will continue to do so until the jobs channel is closed. This call happens synchronously so the worker doesn't pull another job from the channel until work is done for it's current job. The main function then loops 20 times as it did before, but this time it simply writes to the jobs channel rather than starting a `doWork` go routine.

Pretty slick, right? This solution reduces the throughput by only allowing a maximum of 3 go routines to do work concurrently. But, this is only sufficient if your needs are coarse grained. In some cases we will want more control to deal with unknowns, such as how many `doWork` calls will there be per second? If our work involves database queries is 3 concurrent fetches still too fast to avoid taking down the database? What about 1?

Let's try something else.

## Rate Limiting with a Ticker
```
func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)

	rate := time.Second / 2
	limiter := time.Tick(rate)

	for i := 0; i<20; i++ {
		<-limiter
		wg.Add(1)
		go doWork(&wg)
	}
}
```
Run the code here: [Go Playground](https://play.golang.org/p/tKVFV-ZjQx)

This next code snippet utilizes a `Ticker` which is a truer implementation of rate limiting than the worker pool implemented above. `time.Tick` returns an unbuffered channel that is written to at a desired frequency. In this case, we configure it to write to the channel at a rate of 2 per second. This provides rate limiting because reading from a channel blocks if there are no values in the channel, as a result, our for loop can read a maximum of 2 values every second, so we only fire off 2 go routines to do work per second. Boom done!

Not so fast. This would ensure that we only do 2 queries per second. But, what happens if these two queries start to backup the upstream resources? (Perhaps a number higher than 2, would be more reasonable in terms of backing up a database, but I leave this for simplicity). Responses to our 2 queries will take longer and longer, but we are still sending them at the same rate! Next thing you know you have way too many concurrent go routines active and you've taken down your upstream dependencies again.

So, if our dependencies can't keep up we do our best to ease off.

## Rate Limiting with a Ticker and Worker Pool

```
func worker(wg *sync.WaitGroup, jobs chan struct{}) {
	for range jobs {
		doWork(wg)
	}
}

func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)
	rate := time.Second /2
	limiter := time.Tick(rate)
	jobs := make(chan struct{}, 3)
	defer close(jobs)

	// start 3 workers
	for i := 0; i<3; i++ {
		go worker(&wg, jobs)
	}

	for i := 0; i<20; i++ {
		<-limiter
		wg.Add(1)
		jobs <- struct{}{}
	}
}
```
Run the code here: [Go Playground](https://play.golang.org/p/mWJut9x-kZ)

In the above code we've put the previous two ideas together. We set up a Ticker to write to our limiter channel at a desired rate of 2 per second, so we aim for 2 queries per second, but we've also set up a pool of worker go routines to ensure that only a maximum of 3 can be run at a single time. Pretty fancy.

## Rate Limiting with a Ticker and Semaphore
```
func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)

	rate := time.Second / 2
	limiter := time.Tick(rate)

	semaphore := make(chan struct{}, 3)
	defer close(semaphore)

	for i := 0; i<20; i++ {
		<-limiter
		semaphore<-struct{}{}
		wg.Add(1)
		go func(){
			defer func(){<-semaphore}()
			doWork(&wg)
		}()
	}
}
```
Run the code here: [Go Playground](https://play.golang.org/p/pyAr0ZIACR)

In this code example, rather than using a pool of worker go routines we've implemented another channel to act as a semaphore. To acquire the semaphore you write to the semaphore channel. Then use a closure, binding the semaphore to a go routine that is running the `doWork` call. Once, that call is done, the go routine reads from the semaphore channel effectively releasing the semaphore. This semaphore works because writing to a channel blocks if the channel's buffer is full, preventing go routines from being started. This solution removes the overhead of needing to setup a worker pool, but incurs extra overhead for starting up each go routines (but remember, go routines are cheap).

## Rate Limiting with a Buffered Ticker

If you'd like to ramp up a little faster you can buffer the limiter channel:

```
func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)

	rate := time.Second / 2
	maxBurst := 3
	limiter := make(chan struct{}, maxBurst)
	defer close(limiter)
	tick := time.NewTicker(rate)
	defer tick.Stop()


	for i := 0; i< maxBurst; i++ {
		limiter <-struct{}{}
	}

	// write the tick to the limiter channel manually
	go func() {
  		for range tick.C {
			select {
      			case limiter <- struct{}{}:
      default:
			}
  		}
	}()

	for i := 0; i<20; i++ {
		<-limiter
		wg.Add(1)
		go doWork(&wg)
	}
}
```
Run the code here: [Go Playground](https://play.golang.org/p/O63l5sBE38)

The code starts by initializing the ticker and the channel manually. In this example we cannot rely on `time.Tick` to return a channel for us, because we want our limiter channel to be buffered. There is an additional benefit to creating the ticker yourself, using `time.Tick` does not give the developer access to the ticker, which means it cannot be stopped when it is no longer needed. By creating the ticker and channel ourselves we can ensure that there are no memory leaks by stopping the ticker when we are done with it. Next we fill the limiter channels buffer to allow 3 items to be read immediately. Then we start a go routine that writes to the limiter on each tick, and skips the tick if the channel is full. Now we have a bursty limiter, the semaphore or worker pool can be added to this as we have with our other implementations to throttle the throughput if the upstream dependencies are not responding quickly enough.

## Rate Limiting with the Go Rate Subpackage

The ticker implementation of rate limiting works quite well, but it does not scale to massive workloads. If we want hundreds or thousands of ticks per second it is recommended to use a token bucket approach to rate limiting found here: [Docs](https://godoc.org/golang.org/x/time/rate#Limiter

```
package main

import (
	"context"
	"fmt"
	"golang.org/x/time/rate"
	"sync"
	"sync/atomic"
	"time"
)

var numConcurrent int64 = 0

// doWork tracks the number of concurrently running processes and runs for a given processing time.
func doWork(wg *sync.WaitGroup) {
	routineStarting()
	defer routineDone(wg)
	processTime(time.Second * 3)
}

func main() {
	var wg sync.WaitGroup
	startTime := mainStarting()
	defer mainDone(startTime, &wg)

	limiter := rate.NewLimiter(2, 1)

	for i := 0; i < 20; i++ {
		wg.Add(1)
		limiter.Wait(context.Background())
		go doWork(&wg)
	}
}

// ----------------------
// Helper functions below
// ----------------------

func mainDone(start time.Time, wg *sync.WaitGroup) {
	wg.Wait()
	took := time.Since(start)
	fmt.Printf("Finished, took: %v\n", took)
}

func mainStarting() time.Time {
	fmt.Printf("Main is starting\n")
	return time.Now()
}

func routineStarting() {
	atomic.AddInt64(&numConcurrent, 1)
	fmt.Printf("Active Goroutines: %d\n", numConcurrent)
}

func routineDone(wg *sync.WaitGroup) {
	wg.Done()
	atomic.AddInt64(&numConcurrent, -1)
	fmt.Printf("Active Goroutines: %d\n", numConcurrent)
}

func processTime(d time.Duration) {
	time.Sleep(d)
}
```

Go Sub packages are not supported in Go Playground. But you can run this code on your machine with `go run`.

In this example we have created a limiter from the rate package that will make 2 requests per second and allow a burst of 1 event. This package provides other interesting functionality, but let's leave that for a future post.

There you have it! We've gone through several examples for implementing rate limiting strategies in Go. Keep these in mind for your next Go application.
