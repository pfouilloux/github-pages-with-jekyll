---
title: "TDD in practice - Part 2: The test harness"
date: 2023-05-11
classes: wide
header:
  image: /assets/images/tdd_banner.png
  thumb: /assets/images/tdd_thumb.png
  credit: Gopher crash test dummy by Egon Elbre used under CC0 license
  credit-link: github.com/egonelbre/gophers
---

In [the first part of this series](/2023/05/03/tdd-in-practice-part-1.html) we briefly went over what a spike is and why it can be a valuable tool.
We also wrote a spike and arrived at the following list of tasks:
- Load the original image
- Decode the original image into an image object
- Take a square sub-image at the center of the original image
- Create a new go image object
- Draw the square sub-image onto the new image object
- Create a new file to store the thumbnail
- Encode the thumbnail image into the new file

And a mental model of operations:
- `read` to load an image in and decode them into image object
- `write` to create the thumbnail file and encode the thumbnail image into it
- `crop` to extract the a small square from the original image to use as thumbnail
- `resize` to resize the cropped image into a thumbnail

For the rest of the series we'll focus on the `crop` operation because it's the one that has the most going on. The others are trivial to implement and I'll
leave that as an exercise for the reader.

## TDD: Where to start?

In pure TDD we write the tests first. Before any code at all. We then build just enough code for the test to compile, watch it fail, then build just enough code for it to pass,
then build the next test, etc. I recommend sticking to this when starting with TDD but as you gain experience and your mental muscles get stronger you'll find where you can take
some shortcuts.

## But why would I want to do that? Can't I just write some code and test it later?

That's true enough, you could just write some code and retrofit some tests onto it to prove that it works. However I posit that TDD will make your code better than if you just wrote it.

#### 1. The code is designed from the caller's perspective

This is a huge benefit of TDD. By working out the test first, it forces us to think about how our code will be used. How do we construct the expected parameters? What do we return?
How do we return errors? These are all questions that TDD forces us to think about up front, making for a thoughtfully designed API. Additionally we know straight away what should be public
and what should be hidden, making it easier to distill the API's public surface down to the minimum required. Future you and/or future maintainers will thank you for this.

#### 2. The code is inherently testable

It sounds kind of obvious but by writing the tests up front we make our code testable by default. All the little hidden dependencies like time.Now() and other seemingly innocuous static functions
are dealt with up front so we don't get surprised by them when writing our tests. This leads naturally to more coverage and a more complete test suite, which makes our code easier to modify
and maintain. Again future you and/or future maintainers will thank you for this.

#### 3. Edge cases are exposed before we write the code

While writing the tests we'll often come up with edge cases we hadn't thought about when designing the code. By dealing with these up front we save ourselves from having to rearchitect our code or
hack something into it later. Edge cases become first class citizens and are dealt with early and explicitly. This is also something the future will thank you for.

There's a bit of a pattern emerging here, we take a bit of a hit up front but we reap the benefits throughout all of the code's lifespan. Remember when we were writing the spike, we talked about
writing throwaway code? This is the opposite. TDD is a tool we can keep in our belt to help us write maintainable code. If we're writing a script that will be used once and forgotten,
don't use TDD. If we're writing a one shot app for something like an marketing event, TDD may not be the right tool. But if we're expecting people to read and work on our code months and years
from now, TDD can help make code that'll stand the test of time.

## Step 1 - Writing our test harness

First we'll start by writing a basic test harness. We want it to validate two things:
1. Is the cropped image the expected size?
2. Does the cropped image contain the expected pixels? Eg: if we crop towards the center we'd expect to get the center of the original image back.

Briefly, a test harness is a tool to exercise a bit of code and validate it does what is expected. Usually it starts by setting up the initial state of the system, then it executes the code under test from that
state and finally it asserts that the state of the system after the code under test has executed. In our case, this means:
- setting up a dummy image
- executing the crop operation on that dummy image
- asserting that the result of the crop is the expected size and contains the expected portion of the dummy image

This step is critical in the TDD process. This is where we design our public API from the perspective of someone calling it. As we're writing this test harness we're figuring out
what kind of inputs we'll need, how to supply dependencies, etc. I like to start with the smallest possible unit of code, as deep into my mental model as I can. Having the building blocks ready
to go makes it a lot easier to build tests for operations that compose them. If you're struggling to define a mental model of your code, spike it out and build the model from there like we did in [Part 1](/2023/05/03/tdd-in-practice-part-1.html)

> :information_source: This step is tagged with [2.1-TestHarness-Red](https://github.com/pfouilloux/thumbs/tree/2.1-TestHarness-Red)

This is the first draft of our test file:

`internal/img/crop_test.go`
```go
package img_test

import (
  "image"
  "image/color"
  "reflect"
  "testing"
  "thumbs/internal/img"
)

// keeping our test scenarios in a struct outside of the test function allows us to attach methods to it and make our code
// easier to read later on
type scenario struct {
  src                       image.Image
  targetWidth, targetHeight int

  want image.Image
}

func TestCrop(t *testing.T) {
  // first we set up a base image to work with in our tests. This code will create a 1000x1000 image filled with a gradient
  // it's more complex than setting up an empty image, but having it allows us to assert that we're cropping from the center
  // as the center will have different pixels than the sides.
  baseImg := image.NewRGBA(image.Rect(0, 0, 1000, 1000))
  size := baseImg.Bounds().Size()
  for x := 0; x < size.X; x++ {
    for y := 0; y < size.Y; y++ {
      clr := color.RGBA{R: uint8(255 * x / size.X), G: uint8(255 * y / size.Y), B: 55, A: 255}
      baseImg.Set(x, y, clr)
    }
  }

  // this is a table driven test function a common pattern in go. It's advised to use a map as the tests will run in a pseudorandom
  // order. This means they will fail if coupled to each other.
  scns := map[string]scenario{
    // the simplest case just to check that we can get a copy of the src
    "should not crop image with same size as target": {
      src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),
      targetWidth:  100,
      targetHeight: 100,

      want: baseImg.SubImage(image.Rect(0, 0, 100, 100)),
    },
  }

  // our tests run here
  for name, scn := range scns {
    t.Run(name, scn.testCrop)
  }
}

// this is the actual test harness
func (scn scenario) testCrop(t *testing.T) {
  t.Parallel() // runs the tests in parallel, usually a good practicee

  // the function under test
  got := img.Crop(scn.src, scn.targetWidth, scn.targetHeight)

  // we check the size of the images to tell us if we've cropped an image of the right size
  if scn.want.Bounds() != got.Bounds() {
    t.Errorf("expected an %dx%d image but got an %dx%d image", scn.want.Bounds().Dx(), scn.want.Bounds().Dy(), got.Bounds().Dx(), got.Bounds().Dx())
  }

  // finally we check that the content is the same as what we expected
  if !reflect.DeepEqual(scn.want, got) {
    t.Errorf("the cropped image was not the same as the expected subimage")
  }
}
```

Try running this and it will fail to compile. You will probably see something like:
```
❯ go test ./...
thumbs/internal/img: no non-test Go files in /Users/pfouilloux/code/thumbs/internal/img
?       thumbs  [no test files]
FAIL    thumbs/internal/img [build failed]
FAIL
```

> :information_source: This step is tagged with [2.2-TestHarness-Green](https://github.com/pfouilloux/thumbs/tree/2.2-TestHarness-Green)

We need this minimal code to make it compile & pass
`internal/img/crop.go`
```
package img

import "image"

func Crop(image image.Image, width, height int) image.Image {
  return image
}
```
And it passes
```
❯ go test ./...
?       thumbs  [no test files]
ok      thumbs/internal/img     (cached)
```

But there are some improvements we can make to the test harness to make it easier to read/work with in later steps.
[Red -> Green -> Refactor](http://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd) already! Test code is not excempt from refactoring.

> :information_source: This step is tagged with [2.3-TestHarness-Refactor](https://github.com/pfouilloux/thumbs/tree/2.3-TestHarness-Refactor)

`internal/img/crop_test.go`
```go
package img_test

import (
  "image"
  "image/color"
  "reflect"
  "testing"
  "thumbs/internal/img"
)

type scenario struct {
  src                       image.Image
  targetWidth, targetHeight int

  want image.Image
}

func TestCrop(t *testing.T) {
  baseImg := newGradientImage(1000, 1000)

  scns := map[string]scenario{
    "should not crop image with same size as target": {
      src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),
      targetWidth:  100,
      targetHeight: 100,

      want: baseImg.SubImage(image.Rect(0, 0, 100, 100)),
    },
  }

  for name, scn := range scns {
    t.Run(name, scn.testCrop)
  }
}

func (scn scenario) testCrop(t *testing.T) {
  t.Parallel()

  got := img.Crop(scn.src, scn.targetWidth, scn.targetHeight)

  scn.assertIsExpectedSize(t, got)
  scn.assertContainsExpectedPixels(t, got)
}

func (scn scenario) assertIsExpectedSize(t testing.TB, got image.Image) {
  if scn.want.Bounds().Min != got.Bounds().Min || scn.want.Bounds().Max != got.Bounds().Max {
    t.Errorf("expected an %dx%d image but got an %dx%d image", scn.want.Bounds().Dx(), scn.want.Bounds().Dy(), got.Bounds().Dx(), got.Bounds().Dx())
  }
}

func (scn scenario) assertContainsExpectedPixels(t testing.TB, got image.Image) {
  if !reflect.DeepEqual(scn.want, got) {
    t.Errorf("the cropped image was not the same as the expected subimage")
  }
}

func newGradientImage(w, h int) *image.RGBA {
  gradient := image.NewRGBA(image.Rect(0, 0, w, h))
  size := gradient.Bounds().Size()
  for x := 0; x < size.X; x++ {
    for y := 0; y < size.Y; y++ {
      gradient.Set(x, y, interpolateColour(size, x, y))
    }
  }

  return gradient
}

func interpolateColour(sz image.Point, x, y int) color.RGBA {
  return color.RGBA{
    R: uint8(255 * x / sz.X),
    G: uint8(255 * y / sz.Y),
    B: 55,
    A: 255,
  }
}
```

## Conclusion & next steps

Now that we have a basic test harness set up to prove that the image cropping code does what we want it to, we are ready to start building that code.
Having the ability to tell that our code does what it's supposed to up front saves us time in multiple ways:
- Less manual testing is required to confirm that our code works and future maintainers will need less manual testing as well.
- Fewer bugs bubble up at the manual testing phase, because we'll have ironed those out during development.
- We validate our programming decisions early and often, avoiding big rewrites when we find a bug.
- It forces us to write modular code, which makes reuse and modification easier and quicker.

Come back next week when we'll be writing some tests and building out the cropping functionality.

Hope this was helpful and that you've learned something and/or this triggered an interesting train of thought. Once again please don't hesitate to share your thoughts, comments, etc in the comments section!