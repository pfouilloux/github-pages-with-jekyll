---
title: "TDD in practice - Part 3: The test scenario"
date: 2023-05-19
classes: wide
---


# ! Step 2 - Write a test

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


- I like to start with a happy case but a failure case can work too
- The only thing you should be writing here is a test case, we shouldn't need to write any code or fiddle with the test harness
- Don't care if the code is ugly just make that test pass
- Don't handle errors yet, but if you see places where we should be handling errors, rejecting bad input or putting in conditional logic just put a
TODO there and keep going

# ! Step 5 - Write another test

- It can be handy to refer to the TODOs, or maybe there's another case we already have in mind

# ! Step 4 - Make it pass

- Same story as before

# ! Step 6 - Rinse and repeat until the test scenario is complete

- The test scenario here is the group of tests we are working on to exercise our API
- Golang Table Test is a nice unit of scenario, harder in other languages...