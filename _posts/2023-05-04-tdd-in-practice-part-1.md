---
title: "TDD in practice - Part 1: Introduction"
date: 2023-05-03
classes: wide
---

I'm a big proponent of test-driven development because it helps me reflect on my code's behavior and approach programming tasks more effectively.
That's not to say that it's the only way to  write good code. But I've found it to be a very helpful way to approach programming tasks.
Even if you're not into TDD I'd encourage you to have a read and reach out in the comments with any questions or thoughts.

In my day job I work with [GoLang](https://go.dev/), and I've found that TDD is a great fit for Go. As such, I'll be using Go in my examples but these
principles should remain applicable to most languages. Please don't hesitate to reach out in the comments if you'd like to try some of this in another
language and would like a hand porting things over. I don't know all languages as well as Go but I'm conversant in several languages and happy to help.

## What's TDD anyway?

TDD stands for Test Driven Development. In brief, it's a development technique that encourages programmers to think of how the code is going to be used
while we are developing it. It does this by focusing on writing the test first, then writing some code to satisfy the test and finally refactoring the code
before starting the cycle again with the next test case. This is as far as we need to go describing TDD for this series but if you're keen to know more I recommend watching this
[excellent talk by Ian Cooper](https://youtu.be/EZ05e7EMOLM) and/or reading [Kent Beck's TDD By Example](https://www.pearson.com/store/p/test-driven-development-by-example).

## The spike

Before we dig into it, let's take a step back. I've found TDD works best when I've got an idea what I'm building and how I'm going to go about building it.
Ideally we want to start with the smallest component and work our way towards the finished product. In order to do this we need to know what components we 
need and have a general idea of how they are going to fit together. If I'm having trouble listing these components out, I like to write up a quick & dirty spike,
just to get an idea how things will fit together.

### What's a spike

A spike is some throwaway code we write early on to get familiar with a problem, a new framework, an unfamiliar library, etc. The purpose of a spike is to get a better
understanding of what we need to build. It is most emphatically **not production code**. There's a very important reason why it's not production code: speed. The idea is to
quickly get up to speed on a subject so we can effectively plan how to build something and accurately estimate it. Trying to do this in a way we can just copy into prod is
going to result in one of two outcomes. One: the spike takes a long time to build, in which case we may as well have tried to build the prod solution straight away. Two: we have
spike code in production and that will come back to bite us when we need to change it. Ideally a spike should take only a small percentage of the total time allocated for a task and will
make the whole task more predictable. These characteristics can be helpful selling the idea to a team/manager unfamiliar with the concept.

### What are we writing

We'll be writing a simple microservice to generate thumbnails from images. Our user's images are already stored in an s3 bucket by another service
Our service will be receiving a stream of events which are fired from the images bucket. The service is expected to generate a 100x100 square thumbnail
from the center of the image and save it in the same s3 bucket with the same file name suffixed with "_thumb"
I'll be keeping the code pretty basic as the idea is to work through a TDD example not to design a good thumbnail generator.

For this series I'll be focusing on unit testing. We'll leave out the bits about s3 buckets and event stream and spike out resizing an image to a 
100x100 square

> :information_source: Each post in this series will have its own tag in the github repository to make following along easier. This step is tagged with 
> [0_spike](https://github.com/pfouilloux/thumbs/tree/0_spike)

### Step 0 - Writing the spike

Once we've got our project set up the first thing to do is create a little main function so we can start running some code.
We'll set it up with simple logic to read an image from disk and write the result to the output.
You can download a copy of the [testdata/test.jpeg from github](https://github.com/pfouilloux/thumbs/blob/0_spike/testdata/test.jpeg)
`main.go`
```go
package main

import (
	"io"
	"os"
)

func main() {
	// we open the test file
	origFile, err := os.Open("testdata/test.jpeg")
	if err != nil {
		// We do want to handle errors because it's handy to see where things can go wrong, but no need to get fancy in a spike
		panic(err)
	}
	
	// we write to a file to see the results of our work
	thumbFile, err := os.Create("testout/test_thumb.jpeg")
	if err != nil {
		panic(err)
	}
	defer thumbFile.Close()
	
	// we just copy the original file into the new one for now
	if _, err := io.Copy(thumbFile, origFile); err != nil {
		panic(err)
	}
}
```
Running this code will copy the image in 'testdata/test.jpeg' to 'testout/test_thumb.jpg'


Next we add the image resizing code to create a real thumbnail.
`main.go`
```go
package main

import (
	"image"
	"image/jpeg"
	"os"

	"golang.org/x/image/draw"
)

func main() {
	origFile, err := os.Open("testdata/test.jpeg")
	if err != nil {
		// We do want to handle errors because it's handy to see where things can go wrong, but no need to get fancy in a spike
		panic(err)
	}

	orig, _, err := image.Decode(origFile) // We don't care about the image format for the spike
	if err != nil {
		panic(err)
	}

	// this next code block is pretty messy and there's a lot of unnecessary repetition, but that's okay in a spike, we'll fix that up in the prod code
	// in a nutshell this code crops the image into a square around the center of the original image
	square := orig
	if orig.Bounds().Dx() < orig.Bounds().Dy() {
		subImager := orig.(interface {
			SubImage(r image.Rectangle) image.Image
		})

		adj := (orig.Bounds().Dy() - orig.Bounds().Dx()) / 2
		square = subImager.SubImage(image.Rect(orig.Bounds().Min.X, orig.Bounds().Min.Y+adj, orig.Bounds().Max.X, orig.Bounds().Max.Y-adj))
	} else if orig.Bounds().Dx() > orig.Bounds().Dy() {
		subImager := orig.(interface {
			SubImage(r image.Rectangle) image.Image
		})

		adj := (orig.Bounds().Dx() - orig.Bounds().Dy()) / 2
		square = subImager.SubImage(image.Rect(orig.Bounds().Min.X, orig.Bounds().Min.Y+adj, orig.Bounds().Max.X, orig.Bounds().Max.Y-adj))
	}

	// Here we do the actual resizing
	thumb := image.NewRGBA(image.Rect(0, 0, 100, 100))
	draw.BiLinear.Scale(thumb, image.Rect(0, 0, 100, 100), square, square.Bounds(), draw.Src, nil)

	// we write to a file to see the results of our work
	thumbFile, err := os.Create("testout/test_thumb.jpeg")
	if err != nil {
		panic(err)
	}
	defer thumbFile.Close()

	// hardcoded to jpeg here for the spike but we'll want to keep the original encoding in the prod code
	if err := jpeg.Encode(thumbFile, thumb, nil); err != nil {
		panic(err)
	}
}
```
Running this code will create a thumbnail of the image in 'testdata/test.jpeg' in 'testout/test_thumb.jpg'.

#### Learnings and planning

Awesome! We're ready to push this to prod now right? No we're not, not even close.

There's a bunch we can learn from writing this code, not least how we can generate a thumbnail with go. But it's untested, messy and contains hardcoded values.
This is fine when we're just figuring things out but think back to how long it took you to grok the resizing code. I left that uncommented on purpose. If you haven't read it yet,
I'd encourage you to go back and have a go. It's pretty slow going isn't it? If you're anything like me a small headache starts growing behind your eyelids as you look at all the `orig.Bounds()`
and try to find what's different between the first if and else branches. And I wrote the thing!

What I'm trying to get at here is that this code will be a nightmare to maintain. Worse yet, it's entirely untested so whatever changes we make later could break the system in ways we haven't thought of.
There's also several valid edge cases that it doesn't consider. Can you think of some? We'll cover those I could think of in a future post.

So what do we do with it? We retire it. It's served it's purpose. We now know what steps are required to create a thumbnail, we know what apis we need to call and what information to give them.
We might even be able to accurately estimate how long it'll take to build our thumbnail functionality.

Informed by the spike, let's quickly go over what we need to resize an image into a thumbnail.
1. Load the original image
2. Decode the original image into an image object
3. Take a square sub-image at the center of the original image
4. Create a new go image object 
5. Draw the square sub-image onto the new image object
6. Create a new file to store the thumbnail
7. Encode the thumbnail image into the new file

We can even take a stab at a mental model of the operations we'll be building this with:
- `read` to load an image in and decode them into image object
- `write` to create the thumbnail file and encode the thumbnail image into it
- `crop` to extract a square from the original image
- `resize` to resize the cropped image into a thumbnail

Armed with this information we are in a much better position to write clear, consise code to handle this task.
Arguably we could have come up with this without writing a spike, especially if this is a problem we've solved before and have experience to back us up.
That's fair. But firstly this is a fairly trivial problem, as the complexity of the problem increases it becomes harder and harder to keep in our head.
Secondly this is a technique that can be used at any level, even a beginner can achieve good results by experimenting first and breaking things down before building.
Third it does tend to tease out most of the big hidden blockers early in the process, which can help to plan the required work and increase the accuracy of our estimates.

## Next steps

Next we'll dive into building the SubImager implementation using TDD techniques. We'll start with building out the tests to cover the common cases in the next post.
The following will focus on implementing the SubImager itself. In the final post we'll go over refactoring our implementation for clarity and maintainability.

Hope this was helpful and that you've learned something and/or this triggered an interesting train of thought. Once again please don't hesitate to share your thoughts, comments, etc in the comments section! 