---
title: "Mocha tests hanging with Timers"
date: 2018-02-21
draft: true
---

When setting up unit tests for a specific JS class, I noticed that even though the 
mocha based unit tests was always passing, the test process would never actually exit.

Here is what the class looked like, in it's pared down form.

```
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
It can also be something like a runaway setInterval(), or even an errant Promise that never fulfilled.


```
describe('mocha hang', () => {
	let b;
	beforeEach(() => {
		// b = new Blah();
		// this.clock = sinon.useFakeTimers();		
	});

	afterEach(() => {
		// this.clock.restore();
		// if(b){
		// 	b.cleanup();
		// }
	});

	it('should time out after 500 ms', () => {
		let timedOut = false;

		const interval = 1000;
		const run = () => {
			console.log('im in the test');
			setTimeout(run, interval);
		};
		setTimeout(run, interval);
		expect(timedOut).be.false;
		// this.clock.tick(510);
		// expect(timedOut).be.true;
		// this.clock.tick(1500);
	});
});
```