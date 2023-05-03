---
title: "TDD in practice - Part 2: The first test"
date: 2022-02-07
classes: wide
---

# ! Step 1 - Write a test harness

- Start as far in as you can. Smallest unit first. If you can't think of one that's okay. Time for a spike :P
- Write calls to your api before it exits.
- As I'm writing code I'll be figuring out what kind of inputs my tests need, etc
- This step is important.
- This is where we design our api from the perspective of the caller <- This is good.

# ! Step 2 - Make it compile

- Fill out the test inputs
- Stub out the functions and methods the test relies on
- No code in this step, except maybe a constructor.

# ! Step 3 - Write a test

- I like to start with a happy case but a failure case can work too
- The only thing you should be writing here is a test case, we shouldn't need to write any code or fiddle with the test harness

# ! Step 4 - Make it pass

- Don't care if the code is ugly just make that test pass
- Don't handle errors yet, but if you see places where we should be handling errors, rejecting bad input or putting in conditional logic just put a
  TODO there and keep going