## Using Message Passing to Transfer Data Between Threads

One increasingly popular approach to ensuring safe concurrency is _message
passing_, where threads or actors communicate by sending each other messages
containing data. Here’s the idea in a slogan from [the Go language documentation](https://golang.org/doc/effective_go.html#concurrency):
“Do not communicate by sharing memory; instead, share memory by communicating.”

To accomplish message-sending concurrency, Rust's standard library provides an
implementation of channels. A _channel_ is a general programming concept by
which data is sent from one thread to another.

You can imagine a channel in programming as being like a directional channel of
water, such as a stream or a river. If you put something like a rubber duck
into a river, it will travel downstream to the end of the waterway.

A channel has two halves: a transmitter and a receiver. The transmitter half is
the upstream location where you put the rubber duck into the river, and the
receiver half is where the rubber duck ends up downstream. One part of your
code calls methods on the transmitter with the data you want to send, and
another part checks the receiving end for arriving messages. A channel is said
to be _closed_ if either the transmitter or receiver half is dropped.

Here, we’ll work up to a program that has one thread to generate values and
send them down a channel, and another thread that will receive the values and
print them out. We’ll be sending simple values between threads using a channel
to illustrate the feature. Once you’re familiar with the technique, you could
use channels for any threads that need to communicate with each other, such as
a chat system or a system where many threads perform parts of a calculation and
send the parts to one thread that aggregates the results.

First, in Listing 16-6, we’ll create a channel but not do anything with it.
Note that this won’t compile yet because Rust can’t tell what type of values we
want to send over the channel.

<Listing number="16-6" file-name="src/main.rs" caption="Creating a channel and assigning the two halves to `tx` and `rx`">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch16-fearless-concurrency/listing-16-06/src/main.rs}}
```

</Listing>

We create a new channel using the `mpsc::channel` function; `mpsc` stands for
_multiple producer, single consumer_. In short, the way Rust’s standard library
implements channels means a channel can have multiple _sending_ ends that
produce values but only one _receiving_ end that consumes those values. Imagine
multiple streams flowing together into one big river: everything sent down any
of the streams will end up in one river at the end. We’ll start with a single
producer for now, but we’ll add multiple producers when we get this example
working.

The `mpsc::channel` function returns a tuple, the first element of which is the
sending end—the transmitter—and the second element of which is the receiving
end—the receiver. The abbreviations `tx` and `rx` are traditionally used in many
fields for _transmitter_ and _receiver_, respectively, so we name our variables
as such to indicate each end. We’re using a `let` statement with a pattern that
destructures the tuples; we’ll discuss the use of patterns in `let` statements
and destructuring in Chapter 19. For now, know that using a `let` statement this
way is a convenient approach to extract the pieces of the tuple returned by
`mpsc::channel`.

Let’s move the transmitting end into a spawned thread and have it send one
string so the spawned thread is communicating with the main thread, as shown in
Listing 16-7. This is like putting a rubber duck in the river upstream or
sending a chat message from one thread to another.

<Listing number="16-7" file-name="src/main.rs" caption='Moving `tx` to a spawned thread and sending `"hi"`'>

```rust
{{#rustdoc_include ../listings/ch16-fearless-concurrency/listing-16-07/src/main.rs}}
```

</Listing>

Again, we’re using `thread::spawn` to create a new thread and then using `move`
to move `tx` into the closure so the spawned thread owns `tx`. The spawned
thread needs to own the transmitter to be able to send messages through the
channel.

The transmitter has a `send` method that takes the value we want to send. The
`send` method returns a `Result<T, E>` type, so if the receiver has already
been dropped and there’s nowhere to send a value, the send operation will
return an error. In this example, we’re calling `unwrap` to panic in case of an
error. But in a real application, we would handle it properly: return to
Chapter 9 to review strategies for proper error handling.

In Listing 16-8, we’ll get the value from the receiver in the main thread. This
is like retrieving the rubber duck from the water at the end of the river or
receiving a chat message.

<Listing number="16-8" file-name="src/main.rs" caption='Receiving the value `"hi"` in the main thread and printing it'>

```rust
{{#rustdoc_include ../listings/ch16-fearless-concurrency/listing-16-08/src/main.rs}}
```

</Listing>

The receiver has two useful methods: `recv` and `try_recv`. We’re using `recv`,
short for _receive_, which will block the main thread’s execution and wait
until a value is sent down the channel. Once a value is sent, `recv` will
return it in a `Result<T, E>`. When the transmitter closes, `recv` will return
an error to signal that no more values will be coming.

The `try_recv` method doesn’t block, but will instead return a `Result<T, E>`
immediately: an `Ok` value holding a message if one is available and an `Err`
value if there aren’t any messages this time. Using `try_recv` is useful if
this thread has other work to do while waiting for messages: we could write a
loop that calls `try_recv` every so often, handles a message if one is
available, and otherwise does other work for a little while until checking
again.

We’ve used `recv` in this example for simplicity; we don’t have any other work
for the main thread to do other than wait for messages, so blocking the main
thread is appropriate.

When we run the code in Listing 16-8, we’ll see the value printed from the main
thread:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Got: hi
```

Perfect!

### Channels and Ownership Transference

The ownership rules play a vital role in message sending because they help you
write safe, concurrent code. Preventing errors in concurrent programming is the
advantage of thinking about ownership throughout your Rust programs. Let’s do
an experiment to show how channels and ownership work together to prevent
problems: we’ll try to use a `val` value in the spawned thread _after_ we’ve
sent it down the channel. Try compiling the code in Listing 16-9 to see why
this code isn’t allowed.

<Listing number="16-9" file-name="src/main.rs" caption="Attempting to use `val` after we’ve sent it down the channel">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch16-fearless-concurrency/listing-16-09/src/main.rs}}
```

</Listing>

Here, we try to print `val` after we’ve sent it down the channel via `tx.send`.
Allowing this would be a bad idea: once the value has been sent to another
thread, that thread could modify or drop it before we try to use the value
again. Potentially, the other thread’s modifications could cause errors or
unexpected results due to inconsistent or nonexistent data. However, Rust gives
us an error if we try to compile the code in Listing 16-9:

```console
{{#include ../listings/ch16-fearless-concurrency/listing-16-09/output.txt}}
```

Our concurrency mistake has caused a compile time error. The `send` function
takes ownership of its parameter, and when the value is moved, the receiver
takes ownership of it. This stops us from accidentally using the value again
after sending it; the ownership system checks that everything is okay.

### Sending Multiple Values and Seeing the Receiver Waiting

The code in Listing 16-8 compiled and ran, but it didn’t clearly show us that
two separate threads were talking to each other over the channel. In Listing
16-10 we’ve made some modifications that will prove the code in Listing 16-8 is
running concurrently: the spawned thread will now send multiple messages and
pause for a second between each message.

<Listing number="16-10" file-name="src/main.rs" caption="Sending multiple messages and pausing between each one">

```rust,noplayground
{{#rustdoc_include ../listings/ch16-fearless-concurrency/listing-16-10/src/main.rs}}
```

</Listing>

This time, the spawned thread has a vector of strings that we want to send to
the main thread. We iterate over them, sending each individually, and pause
between each by calling the `thread::sleep` function with a `Duration` value of
one second.

In the main thread, we’re not calling the `recv` function explicitly anymore:
instead, we’re treating `rx` as an iterator. For each value received, we’re
printing it. When the channel is closed, iteration will end.

When running the code in Listing 16-10, you should see the following output
with a one-second pause in between each line:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Got: hi
Got: from
Got: the
Got: thread
```

Because we don’t have any code that pauses or delays in the `for` loop in the
main thread, we can tell that the main thread is waiting to receive values from
the spawned thread.

### Creating Multiple Producers by Cloning the Transmitter

Earlier we mentioned that `mpsc` was an acronym for _multiple producer,
single consumer_. Let’s put `mpsc` to use and expand the code in Listing 16-10
to create multiple threads that all send values to the same receiver. We can do
so by cloning the transmitter, as shown in Listing 16-11.

<Listing number="16-11" file-name="src/main.rs" caption="Sending multiple messages from multiple producers">

```rust,noplayground
{{#rustdoc_include ../listings/ch16-fearless-concurrency/listing-16-11/src/main.rs:here}}
```

</Listing>

This time, before we create the first spawned thread, we call `clone` on the
transmitter. This will give us a new transmitter we can pass to the first
spawned thread. We pass the original transmitter to a second spawned thread.
This gives us two threads, each sending different messages to the one receiver.

When you run the code, your output should look something like this:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Got: hi
Got: more
Got: from
Got: messages
Got: for
Got: the
Got: thread
Got: you
```

You might see the values in another order, depending on your system. This is
what makes concurrency interesting as well as difficult. If you experiment with
`thread::sleep`, giving it various values in the different threads, each run
will be more nondeterministic and create different output each time.

Now that we’ve looked at how channels work, let’s look at a different method of
concurrency.

{{#quiz ../quizzes/ch16-02-message-passing.toml}}