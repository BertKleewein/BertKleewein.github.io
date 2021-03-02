# The Problem with Callbacks

I was reviewing some of a co-worker's code and I saw a problematic pattern that I see over and over.
It's one of those things that looks perfectly fine at first, but causes all kinds of difficult-to-tackle problems that you only see at the worst possible times.

This is a perfect example of what Kent Beck calls ["code smell"](https://en.wikipedia.org/wiki/Code_smell).
The code is not technically incorrect, but the pattern indicates that there might be a deeper underlying problem.
I like the term because saying "The code looks fine, but I just don't like the way this part _smells_." is a perfect analogue to the way I feel sometimes.

This happened to be in node.js (actually typescript), but I've, uh, seen people (\<cough\>, \<cough\>, not me) run into this in C, C#, and Python.

# This code is offensive.  Smelling.  I mean, it smells bad.

(Paraphrased from [The Blues Brothers](https://www.imdb.com/title/tt0080455/)

OK, I'm not going to copy the code exactly, but this is enough to get the point across.
This is code for an IoT Device that listens on some network resource for messages and responds to those messages.
The code, nice and simple and elegant, looks like this:

```typescript
    // register a function to listen for network messages
    this.iot_client.on("message", (msg) => {
        if (msg["command"] == "start") {
            logger.info("handling start command");
            this.handle_start_command(msg);
        } else if (msg["command"] == "stop") {
            logger.info("handling stop command");
            this.handle_stop_command(msg);
        } else {
            logger.info("handling unknown command");
            this.handle_unknown_command(msg);
        }
    })
```

This seems pretty simple, right?
When the `iot_client` object receives a message, it emits an event named `message`.

```typescript
    this.emit("message", msg);
```

The code in the anonymous function (starting with `(msg) => {`) gets called every time a `message` event is emitted with the contents of the message.
That anonymous function looks at the message, decides what to do with it, and does it.

So, what's the problem?
Well, I look at this code, and I immediately see, er, smell three big problems:
1. Context
2. Reentrancy
3. Failure handling

Let's talk about them one at a time.

## But, first, a word

Before I go any further, I need get specific on terminology.
I'm calling this anonymous function a "callback".
This might not be exactly the same way you use the word "callback."

Now, I'm not talking about anyone in particular, but if you're pedantic you might say:

    A callback is a function that gets called when an operation is completed.
    It gets called once and then never again.
    This thing that you have here can get called multiple times.
    It's not a callback, it's an event handler.

And, in response, I would say:
    You are correct, sir.
    It can be called multiple times.
    I understand the distinction, and it's an important one.
    I'm still going to call it a callback.
    (In this article at least)


## The first big problem: It's not your context, dude

This problem is easier to understand in language that uses threads like Python, but applies to Node as well.
The reason this is a problem is because the callback gets called by somebody else.
A different library.
That you probably didn't write.

In order to see this problem, you need to think about the code that's calling your callback.
* What kind of state is it in?
* What does it expect?
* If your language has threads, what thread is running?
* Does the library keep anything locked when they call into you?

The simple answer is "you don't know".

Sure, you can look up the code for `this.iot_client` and find the line of code that calls `this.emit('message)`.
That's all fine and dandy.  That's what open source is all about, right?
But, let's be honest, that's cheating.
You can see what `this.iot_client` looks like _right now_, but right now is a really short period of time.
You never know if or when the code behind `this.iot_client` is going to change.
Maybe the library owners make a major change, or maybe some developer in your shop decides to "borrow" your code for a different project that replaces `this.iot_client` with an object that looks almost exactly the same.

Without a doubt, your code is going to more robust if you:
1. Assume that code calling into you is the most bug-filled, worst-smelling, poor-behaving piece of code you've ever seen.
2. Even if that code isn't that bad, I'm sure it still has bugs.
3. Even if it doesn't have bugs, did they make the same assumptions when they were writing the code that you did when you started using it.

No matter what, you need to be defensive.

Saying "be defensive" is a bit too vague.
What, exactly, do I mean by that?
If I had to make it as vague as possible, but still _less_ vague than I was before, I would say that it comes down to 2 things:
1. Always assume that everything you do in one of these callbacks will affect your caller.
2. Always assume that your caller is not prepared for anything unexpected.

I'm thinking about a few specific problems that could happen here:

### 1. If your code throws an exception, what happens to the caller?

Pretend `handle_start_command` throws an exception?  What happens then?
* In the best case, the library wrapped the `this.emit` call in a `try/catch` block where it logs the error and moves on.
* In the next best case, the exception bubbles up and you cause some bizarre and hard-to-diagnose failure deep in the bowels of the library.
* In the worst case, the exception causes a failure so drastic that the entire process stops running.

I had this happen once in a Python app.
My exception caused the network thread to die.
The library wasn't prepared to handle this.
The thread died and the library never noticed it.
I didn't even see anything in the log.
The app just stopped responding to the network and I had no idea why.

### 2. If your code takes a long time to run, what are you preventing?

Surely the library is planning to do something after it calls you.
It can't do that until you're done.
You can't guess what it wants to do.
Maybe it's important.
Maybe it's not.

In my Python example above, I was getting called on the network thread.
This thread was responsible for picking up incoming bytes and sending out outgoing bytes.
If it's busy running my callback code, it can't do those things.
Without knowing it, I was stalling the network every time my callback got called.
Most of the time, this was no big deal, but sometimes, if I was really slow, or if the network was really busy, I would cause the incoming buffer to overflow and drop bytes.

### 3. What happens inside the code that you're calling?

Look back at the code above.
Notice how it calls `logger.info`.
Do you know what that does?
I mean _really_ know.
* Does it directly to the filesystem?
* Does it put the message into a buffer?
* Does it wait for the buffer to fill and then send it to a logging server?

I'm not sure.
That call looks innocent, but it might not be.
If it needs to send the message to a logging server and there's a network glitch, your callback might end up taking 3 precious seconds away from the library as it waits patiently for your callback to complete.

The `logger.info` is an example of unexpected side effects.
It's certainly a common example, but any outside function could have this problem.
To be totally safe, you need to do the least amount of work possible in the callback.
The safest thing is to stash the `msg` object somewhere and return.
Use a different thread (or coroutine or timer) to deal with it.

(If you've ever written a device driver, you know exactly what I'm talking about here and I've just wasted 3 minutes of _your_ precious time explaining something you already knew.)

## The second big problem: the big word that sounds like it shouldn't be a word: "reentrancy"

This one is a little harder to visualize until you have an example.
When I say "reentrancy", I'm talking about calling into some library while it's calling into you.
Your code is already inside the library (because the library is calling you), and you're "re-entering" the library that you're already in.

Let's say our example above included a function that started like this:


```typescript
    this.iot_client.on("message", (msg) => {
        if (msg["command"] == "start") {
            this.iot_client.send_message({currentState: "starting"});
    // ... and so on
}
```

Now when a message arrives:
1. The `iot_client` object emits a `Message` event to call into us.
2. While handling the event, we call back into the `iot_client` object to send a response message back.
That is reentrancy.

### So what?  Why is this a problem?

Well, again, we don't know what state the library is in, and we don't know if it's prepared to send a message while it's in the middle of receiving a message.
Maybe it's OK, or maybe we have a classical deadlock situation.
Can the library send and receive at the same time?

To make a somewhat contrived example, what if the library had code that looks like this:
```typescript
    // Block until we get the critical section
    this.critical_section.enter();

    // Emit our event
    this.emit("message", msg);

    // Release the critical section
    this.critical_section.leave();
```

and also had code that looks like this.

```typescript
function send_message(msg) {
    // Block until we get the critical section
    this.critical_section.enter();

    // Emit our event
    this.send_packet(msg)

    // Release the critical section
    this.critical_section.leave();
}
```

It looks like the library developers were trying to make their code thread safe somehow.
They didn't want to email an Event while they were the middle of a `send_packet()` call.
Why?  I don't know.
I've seen it happen more than once.

This is what happens when our code receives a message:
1. The library function enters the critical section and calls our callback.
2. Our callback tries to send a message in response
3. The `send_message` function blocks waiting for the critical section to become available.
4. The critical section will _never_ become available because it doesn't get released until the callback is complete.
5. The callback can't complete until after `send_message` returns.

We have a good old deadlock.
Neither piece of code can execute and they get stuck forever.

## An easy solution:

In Node.js, we have an easy way to fix the first two problems easily enough.
Just do something like this:
```typescript
    this.iot_client.on("message", (msg) => {
      process.nextTick(() => {
        if (msg["command"] == "start") {
          this.handle_start_command();
        }
      });
    });
```

In other languages, you would put the `msg` object into some sort of list and return.
Some other thread would take the `msg` object out of the list and handle it.

No matter what language we use, we don't want to do anything with the `msg` object until after we've returned control to the library.
This seems like an easy fix, and it is, but there's still one more thing to worry about.

## The third big problem: What about, <gasp> handling failures?

You also need to decide how to handle failures inside callbacks.
This isn't always obvious.
We know that our code can raise an exception, but what do we do about it?
The answer here might be obvious or it might not.
The important part is to plan for failure.

### Option 1: Do nothing (NOT RECOMMENDED)

This is the easiest thing to do, and unfortunately very common.

```typescript
    this.iot_client.on("message", (msg) => {
      process.nextTick(() => {
        if (msg["command"] == "start") {
          this.handle_start_command();
        }
      });
    });
```

If `handle_start_command()` raises an exception, we let the code that called into us handle it.
Maybe they ignore it, maybe they log it, maybe they crash.
In this case, the exception causes the app to exit.
As my 6-year-old would say: NOT GOOD.


### Option 2: Log the failure and move on (Better, but not great)

Instead of letting the caller handle our exception, we can handle it.
If we're lazy, we assume that this will never happen.
But it might, so we add a little bit of logging so some future developer can see what happened and fix the bug.

```typescript
    this.iot_client.on("message", (msg) => {
      process.nextTick(() => {
        try {
          if (msg["command"] == "start") {
            this.handle_start_command();
          }
        } catch(error) {
          console.error(error);
        }
      });
    });
```

(Notice how I put my `try`/`catch` block around the entire function instead of just wrapping the call to `handle_start_command`?  I did this because `if (msg["command"] == "start")` could also raise an exception.  I want to catch _everything_.)

Now that I've added this code, I think we can all agree that I am awesome!
I probably saved that future developer a few hours of tracking down this bug.
They're probably glad that I were so proactive.
Or at least they're like me more than the chump who chose Option #1 and let the app crash.

But, there is an even better option.

### Option 3: Try to recover

This is the hardest option because it requires us to think a out failure recovery.
What do we do if the `handle_start_command()` function raises an exception?

1. We could send a message back to the server to say that it failed.  If such a message exists.  Maybe it doesn't.  [Murphy's Law](https://en.wikipedia.org/wiki/Murphy%27s_law) says that it definitely doesn't exist.
2. We could exit our app.  Or crash.  Or restart.  Or reboot.

One of these things is bound to fix the problem, just as long as we don't get stuck in a loop where the server keeps sending a `start` command and we keep crashing.
Do we need to worry about `handle_start_command()` crashing twice in a row?
I don't know.
Only you can answer this.

The only thing I can say is that you don't want to do this:
```typescript
        } catch(error) {
          this.recoverFromFailedStart()
        }
```

Because `recoverFromFailedStart()` could raise an exception, so we'd better do this:
```typescript
        } catch(error) {
          try {
            this.recoverFromFailedStart();
          } catch(innerError) {
            console.error(innerError);
          }
        }
```

And we probably want to recover from that crash instead of just logging it, so add this:
```typescript
        } catch(error) {
          try {
            this.recoverFromFailedStart();
          } catch(innerError) {
            this.recoverFromFailedRecovery();
          }
        }
```

And, at this point, we wonder if `recoverFromFailedRecovery` might fail.
We're stuck inside the movie _Inception_, wondering how we're going to get out.

## And, so, in conclusion...

Hopefully I've just spent the last 342 lines of markdown telling you things that you already knew.
In that case, as my account likes to say, good on you.
If not, then, hopefully I've given you something new to watch out for.





