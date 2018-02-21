---
title: "Mocha tests hanging with Timers"
date: 2018-02-21
draft: true
---

When setting up unit tests for a specific JS class, I noticed that even though the 
mocha based unit tests was always passing, the test process would never actually exit.

Here is what the class looked like, in it's pared down form.

There was a timer running in the class that would clean up some resources in 
regular intervals.

```
class Blah {
	constructor() {
		this.stopCleanup = false;
		const interval = 1000;
		const run = () => {
			this.dothings();
			if (!this.stopCleanup) {
				setTimeout(run, interval);
			}
		};
		setTimeout(run, interval);
	}

	dothings() {
		console.log('doing things');
	}

	cleanup() {
		console.log('cleanup was called');
		this.stopCleanup = true;
	}
}

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