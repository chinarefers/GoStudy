# Fundamentals of concurrent programming

  [    ![bouncing balls](https://www.nada.kth.se/%7Esnilsson/concurrency/bouncing-balls.jpg)  ](http://www.flickr.com/photos/un_photo/5853737946/)

This is an introduction to concurrent programming with examples in [Go](http://golang.org). The text covers

- concurrent threads of execution (goroutines),
- basic synchronization techniques (channels and locks),
- basic concurrency patterns in Go,
- deadlock and data races,
- parallel computation.

Before you start, you need to know how to write basic Go programs.If you are already familiar with a language such as C/C++, Java, or Python,[A Tour of Go](http://tour.golang.org/) will give you all the background you need.You may also want to take a look at either [Go for C++ programmers](http://code.google.com/p/go-wiki/wiki/GoForCPPProgrammers) or [Go for Java programmers](http://www.nada.kth.se/%7Esnilsson/go_for_java_programmers/).

## 1. Threads of execution

Go permits starting a new thread of execution,a [goroutine](http://golang.org/ref/spec#Go_statements),using the `go` statement.It runs a function in a different, newly created, goroutine.All goroutines in a single program share the same address space.

Goroutines are lightweight,costing little more than the allocation of stack space.The stacks start small and grow by allocating and freeing heap storage as required. Internally goroutines are multiplexed onto multiple operating system threads.If one goroutine blocks an OS thread, for example waiting for input,other goroutines in this thread will migrate so that they may continue running.You do not have to worry about these details.

The following program will print `"Hello from main goroutine"`. It *might* print `"Hello from another goroutine"`, depending on which of the two goroutines finish first.

```
func main() {
    go fmt.Println("Hello from another goroutine")
    fmt.Println("Hello from main goroutine")

    // At this point the program execution stops and all
    // active goroutines are killed.
}

```

[goroutine1.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/goroutine1.go)

The next program will, most likely,print both `"Hello from main goroutine"`and `"Hello from another goroutine"`. They might be printed in any order. Yet another possibility is that thesecond goroutine is extremely slow and doesn’t printits message before the program ends.

```
func main() {
    go fmt.Println("Hello from another goroutine")
    fmt.Println("Hello from main goroutine")

    time.Sleep(time.Second) // wait 1 sec for other goroutine to finish
}

```

[goroutine2.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/goroutine2.go)

Here is a somewhat more realistic example, where we define a function thatuses concurrency to postpone an event.

```
// Publish prints text to stdout after the given time has expired.
// It doesn’t block but returns right away.
func Publish(text string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println("BREAKING NEWS:", text)
    }() // Note the parentheses. We must call the anonymous function.
}

```

[publish1.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/publish1.go)

This is how you might use the `Publish` function.

```
func main() {
    Publish("A goroutine starts a new thread of execution.", 5*time.Second)
    fmt.Println("Let’s hope the news will published before I leave.")

    // Wait for the news to be published.
    time.Sleep(10 * time.Second)

    fmt.Println("Ten seconds later: I’m leaving now.")
}

```

[publish1.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/publish1.go)

The program will, most likely, print the following three lines,in the given order and with a five second break in between each line.

```
$ go run publish1.go
Let’s hope the news will published before I leave.
BREAKING NEWS: A goroutine starts a new thread of execution.
Ten seconds later: I’m leaving now.

```

In general it’s not possible to arrange for threads to wait for each other by sleeping. In the next section we’ll introduce one of Go’s mechanisms for synchronization, *channels*, and thenwe’ll demonstrate how to use a channel to make one goroutine wait for another.

## 2. Channels

  [    ![Sushi conveyor belt](https://www.nada.kth.se/%7Esnilsson/concurrency/sushi-conveyor-belt.jpg)  ](http://www.flickr.com/photos/erikjaeger/35008017/)      Sushi conveyor belt  

A [channel](http://golang.org/ref/spec#Channel_types) is a Go language construct that provides a mechanismfor two goroutines to synchronize execution and communicate bypassing a value of a specified element type.The `<-` operator specifies the channel direction,send or receive. If no direction is given, the channel is bi-directional.

```
chan Sushi      // can be used to send and receive values of type Sushi
chan<- float64  // can only be used to send float64s
<-chan int      // can only be used to receive ints

```

Channels are a reference type and are allocated with make.

```
ic := make(chan int)        // unbuffered channel of ints
wc := make(chan *Work, 10)  // buffered channel of pointers to Work

```

To send a value on a channel,use `<-` as a binary operator. To receive a value on a channel, use it as a unary operator.

```
ic <- 3       // Send 3 on the channel.
work := <-wc  // Receive a pointer to Work from the channel.

```

If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value. Receivers block until there is data to receive.

### Close

The [`close`](http://golang.org/ref/spec#Close)function records that no more values will be sent on a channel. After calling `close`,and after any previously sent values have been received,receive operations will return a zero value without blocking.A multi-valued receive operation additionally returns a boolean indicating whether the value was delivered by a send operation.

```
ch := make(chan string)
go func() {
    ch <- "Hello!"
    close(ch)
}()
fmt.Println(<-ch)  // prints "Hello!"
fmt.Println(<-ch)  // prints the zero value "" without blocking
fmt.Println(<-ch)  // once again prints ""
v, ok := <-ch      // v is "", ok is false

```

A `for` statement with a `range` clause reads successive values sent on a channel until the channel is closed.

```
func main() {
    var ch <-chan Sushi = Producer()
    for s := range ch {
        fmt.Println("Consumed", s)
    }
}

func Producer() <-chan Sushi {
    ch := make(chan Sushi)
    go func() {
        ch <- Sushi("海老握り")  // Ebi nigiri
        ch <- Sushi("鮪とろ握り") // Toro nigiri
        close(ch)
    }()
    return ch
}

```

[sushi.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/sushi.go)

## 3. Synchronization

In the next example we let the `Publish` function return a channel, which is used to broadcast a message when the text has been published.

```
// Publish prints text to stdout after the given time has expired.
// It closes the wait channel when the text has been published.
func Publish(text string, delay time.Duration) (wait <-chan struct{}) {
    ch := make(chan struct{})
    go func() {
        time.Sleep(delay)
        fmt.Println("BREAKING NEWS:", text)
        close(ch) // broadcast – a closed channel sends a zero value forever
    }()
    return ch
}

```

[publish2.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/publish2.go)

Notice that we use a channel of empty structs: `struct{}`.This clearly indicates that the channel will only be used for signalling,not for passing data.

This is how you might use the function.

```
func main() {
    wait := Publish("Channels let goroutines communicate.", 5*time.Second)
    fmt.Println("Waiting for the news...")
    <-wait
    fmt.Println("The news is out, time to leave.")
}

```

[publish2.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/publish2.go)

The program will print the following three lines in the given order.The final line is printed immediately after the news is out.

```
$ go run publish2.go
Waiting for the news...
BREAKING NEWS: Channels let goroutines communicate.
The news is out, time to leave.

```

## 4. Deadlock

  [    ![traffic jam](https://www.nada.kth.se/%7Esnilsson/concurrency/traffic-jam.jpg)  ](http://www.flickr.com/photos/lasgalletas/263909727/)

Let’s introduce a bug in the `Publish` function:

```
func Publish(text string, delay time.Duration) (wait <-chan struct{}) {
    ch := make(chan struct{})
    go func() {
        time.Sleep(delay)
        fmt.Println("BREAKING NEWS:", text)
        //close(ch)
    }()
    return ch
}

```

The main program starts like before: it prints the first line and then waits for five seconds. At this point the goroutine started by the`Publish` function will print the breaking news and then exit leaving the main goroutine waiting.

```
func main() {
    wait := Publish("Channels let goroutines communicate.", 5*time.Second)
    fmt.Println("Waiting for the news...")
    <-wait
    fmt.Println("The news is out, time to leave.")
}

```

The program will not be able to make any progress beyond this point.This condition is known as a deadlock.

> A *deadlock* is a situation in which threads arewaiting for each other and none of them is able to proceed.

Go has good support for deadlock detection at runtime.In a situation where no goroutine is able to make progress,a Go program will often provide a detailed error message.Here is the output from our broken program:

```
Waiting for the news...
BREAKING NEWS: Channels let goroutines communicate.
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
    .../goroutineStop.go:11 +0xf6

goroutine 2 [syscall]:
created by runtime.main
    .../go/src/pkg/runtime/proc.c:225

goroutine 4 [timer goroutine (idle)]:
created by addtimer
    .../go/src/pkg/runtime/ztime_linux_amd64.c:73

```

In most cases it’s easy to figure out what caused a deadlock in a Go program and then it’s just a matter of fixing the bug.

## 5. Data races

A deadlock may sound bad, but the truly disastrous errors thatcome with concurrent programming are data races.They are quite common and can be very hard to debug.

> A *data race* occurs when two threads access the samevariable concurrently and at least one of the accesses is a write.

This function has a data race and it’s behavior is undefined.It may, for example, print the number 1. Try to figure out how that can happen – one possible explanation comes after the code.

```
func race() {
    wait := make(chan struct{})
    n := 0
    go func() {
        n++ // one access: read, increment, write
        close(wait)
    }()
    n++ // another conflicting access
    <-wait
    fmt.Println(n) // Output: UNSPECIFIED
}

```

[datarace.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/datarace.go)

The two goroutines, `g1` and `g2`,participate in a race and there is no way to know in which order the operationswill take place. The following is one out of many possible outcomes.

- `g1` reads the value `0` from `n`.
- `g2` reads the value `0` from `n`.
- `g1` increments its value from `0` to `1`.
- `g1` writes `1` to `n`.
- `g2` increments its value from `0` to `1`.
- `g2` writes `1` to `n`.
- The programs prints the value of n, which is now `1`.

The name ”data race” is somewhat misleading.Not only is the ordering of operations undefined;there are *no guarantees whatsoever*. Both compilers and hardware frequently turn code upside-down and inside-outto achieve better performance. If you look at a thread in mid-action,you might see pretty much anything:

  [    ![mid action](https://www.nada.kth.se/%7Esnilsson/concurrency/mid-action.jpg)  ](http://www.flickr.com/photos/brandoncwarren/2953838847/)

The only way to avoid data races is to synchronize access toall mutable data that is shared between threads. There are several ways toachieve this. In Go, you would normally use a channel or a lock.(Lower-lever mechanisms are available in the[`sync`](http://golang.org/pkg/sync/) and[`sync/atomic`](http://golang.org/pkg/sync/atomic/) packages,but are not discussed in this text.)

The preferred way to handle concurrent data access in Go is touse a channel to pass the actual data from one goroutine to the next.The motto is: ”Don’t communicate by sharing memory; share memory by communicating.”

```
func sharingIsCaring() {
    ch := make(chan int)
    go func() {
        n := 0 // A local variable is only visible to one goroutine.
        n++
        ch <- n // The data leaves one goroutine...
    }()
    n := <-ch   // ...and arrives safely in another goroutine.
    n++
    fmt.Println(n) // Output: 2
}
```

[datarace.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/datarace.go)

In this code the channel does double duty. It passes the datafrom one goroutine to another and it acts as a point of synchronization:the sending goroutine will wait for the other goroutine to receive the dataand the receiving goroutine will wait for the other goroutine to send the data.

[The Go memory model](http://golang.org/ref/mem) –the conditions under which reads of a variable in one goroutinecan be guaranteed to observe values produced by writes to the same variablein a different goroutine –is quite complicated,but as long as you share all mutable data between goroutinesthrough channels you are safe from data races. 

## 6. Mutual exclusion lock

  [    ![lock](https://www.nada.kth.se/%7Esnilsson/concurrency/lock.jpg)  ](http://www.flickr.com/photos/dzarro72/7187334179/)

Sometimes it’s more convenient to synchronize data access by explicit locking instead of using channels.The Go standard library offers a mutual exclusion lock,[sync.Mutex](http://golang.org/pkg/sync/#Mutex),for this purpose.

For this type of locking to work, it’s crucial that all accesses to the shared data, both reads and writes, are performed only when a goroutine holds the lock. One mistake by a single goroutine is enough to break the program and introduce a data race.

Because of this you should consider designing a custom data structurewith a clean API and make sure that all the synchronization is done internally. In this example we build a safe and easy-to-use concurrent data structure, `AtomicInt`, that stores a single integer.Any number of goroutines can safely access this number through the`Add` and `Value` methods.

```
// AtomicInt is a concurrent data structure that holds an int.
// Its zero value is 0.
type AtomicInt struct {
    mu sync.Mutex // A lock than can be held by just one goroutine at a time.
    n  int
}

// Add adds n to the AtomicInt as a single atomic operation.
func (a *AtomicInt) Add(n int) {
    a.mu.Lock() // Wait for the lock to be free and then take it.
    a.n += n
    a.mu.Unlock() // Release the lock.
}

// Value returns the value of a.
func (a *AtomicInt) Value() int {
    a.mu.Lock()
    n := a.n
    a.mu.Unlock()
    return n
}

func lockItUp() {
    wait := make(chan struct{})
    var n AtomicInt
    go func() {
        n.Add(1) // one access
        close(wait)
    }()
    n.Add(1) // another concurrent access
    <-wait
    fmt.Println(n.Value()) // Output: 2
}

```

[datarace.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/datarace.go)

## 7. Detecting data races

Races can sometimes be hard to detect.This function has a data race and when I executedthe program it printed `55555`.Try it out, you may well get a different result.(The [`sync.WaitGroup`](http://golang.org/pkg/sync/#WaitGroup)is part of Go’s standard library;it waits for a collection of goroutines to finish.)

```
func race() {
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Print(i) // The variable i is shared by six (6) goroutines.
            wg.Done()
        }()
    }
    wg.Wait() // Wait for all five goroutines to finish.
    fmt.Println()
}

```

[raceClosure.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/raceClosure.go)

A plausible explanation for the `55555` outputis that the goroutine that executes `i++` managed todo this five times before any of the other goroutines executed their print statements.The fact that the updated value of `i` was visibleto the other goroutines is purely coincidental.

A simple solution is to use a local variable and pass the numberas a parameter when starting the goroutine.

```
func correct() {
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go func(n int) { // Use a local variable.
            fmt.Print(n)
            wg.Done()
        }(i)
    }
    wg.Wait()
    fmt.Println()
}

```

[raceClosure.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/raceClosure.go)

This code is correct and the program prints an expected result,such as `24031`.Recall that the order of execution between goroutines is unspecified and may vary.

It’s also possible to avoid this data race while still using a closure,but then we must take care to use a unique variable for each goroutine.

```
func alsoCorrect() {
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < 5; i++ {
        n := i // Create a unique variable for each closure.
        go func() {
            fmt.Print(n)
            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println()
}

```

[raceClosure.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/raceClosure.go)

### Automatic data race detection

In general, it’s not possible to automatically detect all data races,but Go (starting with version 1.1) has a powerful [data race detector](http://tip.golang.org/doc/articles/race_detector.html).

The tool is simple to use:just add the `-race` flag to the `go` command.Running the program above with the detector turned on gives the following clear and informative output.

```
$ go run -race raceClosure.go 
Race: 
==================
WARNING: DATA RACE
Read by goroutine 2:
  main.func·001()
      ../raceClosure.go:22 +0x65

Previous write by goroutine 0:
  main.race()
      ../raceClosure.go:20 +0x19b
  main.main()
      ../raceClosure.go:10 +0x29
  runtime.main()
      ../go/src/pkg/runtime/proc.c:248 +0x91

Goroutine 2 (running) created at:
  main.race()
      ../raceClosure.go:24 +0x18b
  main.main()
      ../raceClosure.go:10 +0x29
  runtime.main()
      ../go/src/pkg/runtime/proc.c:248 +0x91

==================
55555
Correct: 
01234
Also correct: 
01324
Found 1 data race(s)
exit status 66

```

The tool found a data race consisting of a write toa variable on line 20 in one goroutine,followed by an unsynchronized read from the same variable on line 22 in another goroutine.

Note that the race detector only finds data races that actually happen during execution.

## 8. Select statement

The [select statement](http://golang.org/ref/spec#Select_statements) is the final tool in Go’s concurrency toolkit.It chooses which of a set of possible communications will proceed.If any of the communications can proceed, one of them is randomly chosen and the corresponding statements are executed.Otherwise, if there is no default case,the statement blocks until one of the communications can complete.

Here is a toy example showing how the select statement canbe used to implement a random number generator.

```
// RandomBits returns a channel that produces a random sequence of bits.
func RandomBits() <-chan int {
    ch := make(chan int)
    go func() {
        for {
            select {
            case ch <- 0: // note: no statement
            case ch <- 1:
            }
        }
    }()
    return ch
}

```

[randBits.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/randBits.go)

Somewhat more realistically, here is how a select statementcould be used to set a time limit on an operation.The code will either print the news or the time-out message,depending on which of the two receive statements that can proceed first.

```
select {
case news := <-NewsAgency:
    fmt.Println(news)
case <-time.After(time.Minute):
    fmt.Println("Time out: no news in one minute.")
}

```

The function [`time.After`](http://golang.org/pkg/time/#After)is part of Go’s standard library;it waits for a specified time to elapse and then sends the current timeon the returned channel.

## 9. The mother of all concurrency examples

  [    ![couples](https://www.nada.kth.se/%7Esnilsson/concurrency/couples.jpg)  ](http://www.flickr.com/photos/julia_manzerova/4617019027/)

Take the time to study this example carefully. When you understandit fully, you will have a thorough grasp of how concurrency works in Go.

The programs demonstrates how a channel can be used for both sending andreceiving by any number of goroutines. It also shows how  the selectstatement can be used to choose one out of several communications.

```
func main() {
    people := []string{"Anna", "Bob", "Cody", "Dave", "Eva"}
    match := make(chan string, 1) // Make room for one unmatched send.
    wg := new(sync.WaitGroup)
    wg.Add(len(people))
    for _, name := range people {
        go Seek(name, match, wg)
    }
    wg.Wait()
    select {
    case name := <-match:
        fmt.Printf("No one received %s’s message.\n", name)
    default:
        // There was no pending send operation.
    }
}

// Seek either sends or receives, whichever possible, a name on the match
// channel and notifies the wait group when done.
func Seek(name string, match chan string, wg *sync.WaitGroup) {
    select {
    case peer := <-match:
        fmt.Printf("%s sent a message to %s.\n", peer, name)
    case match <- name:
        // Wait for someone to receive my message.
    }
    wg.Done()
}

```

[matching.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/matching.go)

Example output:

```
$ go run matching.go
Cody sent a message to Bob.
Anna sent a message to Eva.
No one received Dave’s message.

```

## 10. Parallel computation

  [    ![CPUs](https://www.nada.kth.se/%7Esnilsson/concurrency/cpus.jpg)  ](http://www.flickr.com/photos/somegeekintn/4819945812//)

One application of concurrency is to divide a large computationinto work units that can be scheduled for simultaneous computationon separate CPUs.

Distributing computations onto several CPUs is more of an artthan a science. Here are some rules of thumb.

- Each work unit should take about 100μs to 1ms to compute.If the units are too small, the administrative overhead of dividingthe problem and scheduling sub-problems might be too large.If the units are too big, the whole computation may have to waitfor a single slow work item to finish. This slowdown can happenfor many reasons, such as scheduling, interrupts from other processes,and unfortunate memory layout. (Note that the number of work unitsis independent of the number of CPUs.)
- Try to minimize the amount of data sharing.Concurrent writes can be very costly, particularly so if goroutinesexecute on separate CPUs. Sharing data for reading is often much lessof a problem.
- Strive for good locality when accessing data.If data can be kept in cache memory, data loading and storingwill be dramatically faster.Once again, this is particularly important for writing.

The following example shows how to divide a costly computation anddistribute it on all available CPUs.This is the code we want to optimize.

```
type Vector []float64

// Convolve computes w = u * v, where w[k] = Σ u[i]*v[j], i + j = k.
// Precondition: len(u) > 0, len(v) > 0.
func Convolve(u, v Vector) (w Vector) {
    n := len(u) + len(v) - 1
    w = make(Vector, n)

    for k := 0; k < n; k++ {
        w[k] = mul(u, v, k)
    }
    return
}

// mul returns Σ u[i]*v[j], i + j = k.
func mul(u, v Vector, k int) (res float64) {
    n := min(k+1, len(u))
    j := min(k, len(v)-1)
    for i := k - j; i < n; i, j = i+1, j-1 {
        res += u[i] * v[j]
    }
    return
}

```

The idea is simple:identify work units of suitable size and then run each work unitin a separate goroutine. Here is a concurrentversion of `Convolve`.

```
func Convolve(u, v Vector) (w Vector) {
    n := len(u) + len(v) - 1
    w = make(Vector, n)

    // Divide w into work units that take ~100μs-1ms to compute.
    size := max(1, 1<<20/n)

    wg := new(sync.WaitGroup)
    wg.Add(1 + (n-1)/size)
    for i := 0; i < n && i >= 0; i += size { // i < 0 after int overflow
        j := i + size
        if j > n || j < 0 { // j < 0 after int overflow
            j = n
        }
        // These goroutines share memory, but only for reading.
        go func(i, j int) {
            for k := i; k < j; k++ {
                w[k] = mul(u, v, k)
            }
            wg.Done()
        }(i, j)
    }
    wg.Wait()
    return
}

```

[convolution.go](https://www.nada.kth.se/%7Esnilsson/concurrency/src/convolution.go)

When the work units have been defined, it’s often best toleave the scheduling to the runtime and the operating system.However, with Go 1.* you may need to tell the runtime how manygoroutines you want executing code simultaneously.

```
func init() {
    numcpu := runtime.NumCPU()
    runtime.GOMAXPROCS(numcpu) // Try to use all available CPUs.
}

```

[Stefan Nilsson](https://plus.google.com/+StefanNilsson/about?rel=author)