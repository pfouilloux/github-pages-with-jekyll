---
title: "TDD in practice - Part 3: The test scenario"
date: 2023-06-16
classes: wide
header:
    image: /assets/images/tdd_banner.png
    thumb: /assets/images/tdd_thumb.png
    caption: Gopher by [Egon Elbre](github.com/egonelbre/gophers)
    credit: Gopher crash test dummy by Egon Elbre used under CC0 license
    credit-link: github.com/egonelbre/gophers
---

In [the second part of this series](/2023/05/11/tdd-in-practice-part2.html) we wrote a simple test harness to prove that the image cropping code we're about
to write does what we want it to.

We also learned that while writing tests up front may seem like it costs us time, it actually saves us time in multiple ways:
- Less manual testing is required to confirm that our code works and future maintainers will need less manual testing as well.
- Fewer bugs bubble up at the manual testing phase, because we'll have ironed those out during development.
- We validate our programming decisions early and often, avoiding big rewrites when we find a bug.
- It forces us to write modular code, which makes reuse and modification easier and quicker.

### Changing a programming decision

When thinking about how our cropping code would be used in a thumbnail generator we don't want to have to calculate and pass in the target width and height.
What we really want is a function to crop a square out of an image. It'll be up to the `Resize` function to change the dimensions to the desired size.
With that in mind I've gone ahead and renamed the `Crop` function to `Square`. Because we're nice and early in the process, this is a trivial change to make.
The biggest difference is that our test harness now looks like this:

`internal/img/square_test.go`
```go
func TestSquare(t *testing.T) {
    baseImg := newGradientImage(1000, 1000)

    scns := map[string]scenario{
        "should not crop image with same size as target": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),

            want: baseImg.SubImage(image.Rect(0, 0, 100, 100)),
        },
    }

    for name, scn := range scns {
        t.Run(name, scn.testSquare)
    }
}

func (scn scenario) testSquare(t *testing.T) {
    t.Parallel()

    got := img.Square(scn.src)

    scn.assertIsExpectedSize(t, got)
    scn.assertContainsExpectedPixels(t, got)
}
```

### Step 3 - Write a failing test

With the small refactoring out of the way we can start writing our first failing test. I like to start with happy cases first and keep failure cases for last but
any order is fine, just write that first test.

In our case, we'll start with a test that checks that given a portrait oriented image of
100x200 the `Square` function should return a 100x100 image taken from the center of the original image. For example, given:

![300x400 Photo by ZHENYU LUO on Unsplash]({{ site.url }}{{ site.baseurl }}/assets/images/tdd-in-practice/portrait-example.jpg)

We'd expect a 300x300 square like:

![300x300 Cropped image]({{ site.url }}{{ site.baseurl }}/assets/images/tdd-in-practice/portrait-cropped-example.jpg)

To achieve this we simply add a new scenario to our test suite. The new scenario will provide an image of 100x200 and expect an image with the top
50 pixels and bottom 50 pixels cropped out, resulting in a 100x100 square. If we've set up our test harness correctly we don't need to change it and we shouldn't need to write
any code either.

`internal/img/square_test.go`
```go
func TestCrop(t *testing.T) {
    baseImg := newGradientImage(1000, 1000)

    scns := map[string]scenario{
        "should not crop image with same size as target": {
            src: baseImg.SubImage(image.Rect(0, 0, 100, 100)),

            want: baseImg.SubImage(image.Rect(0, 0, 100, 100)),
        },
        "should crop portrait image towards the center": {
            src: baseImg.SubImage(image.Rect(0, 0, 100, 200)),

            want: baseImg.SubImage(image.Rect(0, 50, 100, 150)),
        },
    }

    for name, scn := range scns {
        t.Run(name, scn.testCrop)
    }
}
```

### Step 4 - Make it pass

When we run our test we get some errors. This is good!
```
square_test.go:50: expected an 100x100 image but got an 100x200 image
square_test.go:56: the cropped image was not the same as the expected subimage
```

We can now write some code to get our test to pass. It doesn't matter if the code is ugly, we just want our test to pass. Also don't handle errors nicely just yet,
I like to put a panic & a TODO and move on. We'll be building test scenarios for error cases later, let's not get distracted.

- I like to start with a happy case but a failure case can work too
- The only thing you should be writing here is a test case, we shouldn't need to write any code or fiddle with the test harness
- Don't care if the code is ugly just make that test pass
- Don't handle errors yet, but if you see places where we should be handling errors, rejecting bad input or putting in conditional logic just put a
TODO there and keep going

# ! Step 5 - Write another test

`internal/img/crop_test.go`
```go
func TestCrop(t *testing.T) {
    baseImg := newGradientImage(1000, 1000)

    scns := map[string]scenario{
        "should not crop image with same size as target": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),
            targetWidth:  100,
            targetHeight: 100,

            want: baseImg.SubImage(image.Rect(0, 0, 100, 100)),
        },
        "should crop square image towards the center": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),
            targetWidth:  50,
            targetHeight: 50,

            want: baseImg.SubImage(image.Rect(25, 25, 75, 75)),
        },
        "should crop vertical image towards the center": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 200)),
            targetWidth:  50,
            targetHeight: 50,

            want: baseImg.SubImage(image.Rect(25, 75, 75, 125)),
        },
    }

    for name, scn := range scns {
        t.Run(name, scn.testCrop)
    }
}
```

# ! Step 4 - Make it pass

- Same story as before

# ! Step 6 - Rinse and repeat until the test scenario is complete

`internal/img/crop_test.go`
```go
func TestCrop(t *testing.T) {
    baseImg := newGradientImage(1000, 1000)

    scns := map[string]scenario{
        "should not crop image with same size as target": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),
            targetWidth:  100,
            targetHeight: 100,

            want: baseImg.SubImage(image.Rect(0, 0, 100, 100)),
        },
        "should crop square image towards the center": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 100)),
            targetWidth:  50,
            targetHeight: 50,

            want: baseImg.SubImage(image.Rect(25, 25, 75, 75)),
        },
        "should crop vertical image towards the center": {
            src:          baseImg.SubImage(image.Rect(0, 0, 100, 200)),
            targetWidth:  50,
            targetHeight: 50,

            want: baseImg.SubImage(image.Rect(25, 75, 75, 125)),
        },
        "should crop horizontal image towards the center": {
            src:          baseImg.SubImage(image.Rect(0, 0, 200, 100)),
            targetWidth:  50,
            targetHeight: 50,

            want: baseImg.SubImage(image.Rect(75, 25, 125, 75)),
        },
    }

    for name, scn := range scns {
        t.Run(name, scn.testCrop)
    }
}
```

- The test scenario here is the group of tests we are working on to exercise our API
- Golang Table Test is a nice unit of scenario, harder in other languages...