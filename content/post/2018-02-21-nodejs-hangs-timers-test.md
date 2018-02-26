---
title: "Mocha tests hanging with Timers"
date: 2018-02-21
---

When setting up unit tests for a specific JS class, I noticed that even though the 
mocha based unit tests was always passing, the test process would never actually exit.

Here is what the class looked like, in it's pared down form.

``` js
class Blah {
	constructor() {
		this.stopCleanup = false;
		const interval = 1000;
		const run = () => {
			this.dothings();
            setTimeout(run, interval);
		};
		setTimeout(run, interval);
	}
	
	methodUnderTest(){
	    // do some other work
	}

	dothings() {
		console.log('doing things');
	} 
}
```

There was a timer running in the class that would clean up some resources in 
regular intervals.


Here is the mocha test that was passing, but also never exiting

```
describe('mocha hang', () => {
	let b;
	beforeEach(() => {
		b = new Blah();
	}); 

	it('testing blahs method', () => { 
		b.methodUnderTest();
		expect(something).to.be.true; 
	});
});
```

This issue snuck up on me because I added a timer at some later 
state

Node stays alive while [there are timers still running](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
or if there is any pending asynchronous I/O

In fact, [mocha even mentions this](https://mochajs.org/) on their main page

> “Hanging” most often manifests itself if a server is still 
listening on a port, or a socket is still open, etc. 
It can also be something like a runaway setInterval(), 
or even an errant Promise that never fulfilled.

There are a couple of ways to fix this.

One way would be to mock time by using [sinon to fake time](http://sinonjs.org/releases/v4.4.2/fake-timers/).

```
describe('mocha hang', () => {
	let b;
	beforeEach(() => {
		b = new Blah();
		this.clock = sinon.useFakeTimers();		
	});

	afterEach(() => {
		// this.clock.restore();
	});

	it('testing blahs method', () => { 
		b.methodUnderTest();
		expect(something).to.be.true; 
	});
});
```

The advantage of this is that you can now control how much time
is passing in the test and actually control and unit test the 
timers.


Another solution would be to enable/disable the timer specifically 
before and after each test.
This might nnot be the best solution. Now you have to expose a way 
to start and stop the timer. You are also dependant on the timer interval.
If you start a timer with a long interval, the test wont exit until
the interval completes.

A third solution is one i learnt about recently. You can tell the Nodejs 
event loop to not stay alive for a timer by using [timeout.unref()](https://nodejs.org/api/timers.html#timers_timeout_unref).
Again, I don't recommend this solution. It's too invasive and mostly 
likely will force you to refactor your code some more.
Not to mention the note in the documentation for that function 
that warns of the performance impact of overusing this function.
