---
title: Go Gotchas
date: 2020-06-05 00:00:00 +05:00
modified: 2020-06-05 00:00:00 +05:00
tags: [go, golang]
description: Some gotchas or surprising features of the Go langauge
image: "https://golang.org/lib/godoc/images/go-logo-blue.svg"
---

I have had the good fortune of being able to use Go in my current job. I like the language a lot, it is pretty straightforward. I like that it is typed for safety. I like that the language features are mostly orthogonal. There are however, a few things I find inconsistent, confusing, or surprising. I would like to document these here.

#### Calling a method on a nil pointer

```go
package main

import "fmt"

type Godzilla struct {
  fireballs int
}

func (g *Godzilla) BlowFireball() {
  g.fireballs--
}

func (g *Godzilla) DoNothing() {
  fmt.Println("I'm doing nothing")
}

func main() {
  var godzilla *Godzilla
  godzilla.DoNothing()
  godzilla.BlowFireball()
}
```

What do you think the result of running this code is? I think it's intuitive to think "there will be a runtime error when `godzilla.DoNothing()` is called, because godzilla is nil". It's true, godzilla is nil, but the surprising thing to me is that Go doesn't care about that. We don't encounter an error until `g.fireballs--` is called. It seems that methods are just some syntactic sugar on top of plain old functions. Think about it this way: if you pass in a nil pointer to a function, it doesn't fail at the time you call the function, it fails only once it tries to do something with that nil value. In the example above you can see

#### Copying structs that contain pointers

```go
package main

import "fmt"

type Lawnmower struct {
  handsChopped int
  lastHandChopped *string
}

func main() {

  bill := "bill"
  husqvarna := Lawnmower{
    handsChopped:    5,
    lastHandChopped: &bill,
  }

  snapper := husqvarna
  snapper.handsChopped = 10

  fmt.Println("Husqvarna hands chopped:", husqvarna.handsChopped)
  fmt.Println("Snapper hands chopped:", snapper.handsChopped)

  bill = "hank"

  fmt.Println("Husqvarna last hand chopped:", *husqvarna.lastHandChopped)
  fmt.Println("Snapper last hand chopped:", *snapper.lastHandChopped)

  buck := "buck"
  snapper.lastHandChopped = &buck

  fmt.Println("Husqvarna last hand chopped:", *husqvarna.lastHandChopped)
  fmt.Println("Snapper last hand chopped:", *snapper.lastHandChopped)
}
```

It is a simple mistake to think "I copy this struct, and then everything is different so I can modify one without modifying the other". Not so fast! Structs may contain pointer, and when they're copied, they do not generate a new pointer and a new thing that pointer is pointing at, they just copy the value of the pointer. So unless you change the pointer itself, you may have an issue modifying one struct and inadvertently changing the other too.

In the example above, modifying the number of hands chopped by each lawnmower is no problem, they are just values, and they don't interfere with each other. However, when we get to the line `bill = "hank"` both pointers are still pointing at the location of bill. Modifying the bill variable will change whose hand was last chopped in both lawnmowers.

If you have to modify what is in one struct but not the other, you need to declare a new variable and get a pointer to that.

This is a trivial example, but you can probably easily imagine how this can get really complicated when you want to copy a struct with other structs nested inside, any of which might contain pointers.

#### Using len() on strings that contain high unicode characters

```go
package main

import (
	"fmt"
)

func main() {
	foo := "Föö" // ö is a 2-byte character in unicode
	
	fmt.Println(len(foo)) // prints 5, because this is counting bytes
	
	fmt.Println(len([]rune(foo))) // prints 3, what you probably want, because it is 3 discrete runes
	
	fmt.Println(foo[1:2]) // prints � because you just tried to cut a 2 byte character in half
}
```

Something that is definitely unintuitive about len() is that when you use it on a string it doens't actually count the characters, it counts the bytes. It just so happens that many commonly used characters are one byte so this doesn't matter, and it is likely to slip by the programmer unnoticed.