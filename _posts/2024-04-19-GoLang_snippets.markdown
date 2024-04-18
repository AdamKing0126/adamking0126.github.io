--- 
layout: post
title: "GoLang Snippets"
date: 2024-04-18 0:00:00
---
# Golang Snippets #

I'm a Go rookie, so I'm going to use this page to collect handy patterns/snippets.

### Pretty-print a JSON blob ###

Sometimes when you make a network request, you get a blob of JSON that you don't entirely know the shape of.  This is a bit awkward, because if you don't know the data's shape, you can't make a struct to deserialize the data to.

Coming from interpreted programming languages (Ruby, Python, JS), I'm used to just doing something like `pprint.pp(json.loads(data))`.  AFAICT, there's nothing like this in Go.  So I came up with this:

```
var indentedJSON bytes.Buffer

jsonBlob := data.([]uint8)
err := json.Indent(&indentedJSON, jsonBlob, "", "  ")
if err != nil {
    log.Fatalf("error indenting JSON: %v", err)
}

fmt.Println(indentedJSON.String())
```

Seems clunky.  Why do we have to do this?  Why is a JSON object represented as a byte slice, instead of a string?  I mean, at least you can look at the string and visually inspect its contents.  

Go uses byte slices as the standard way to handle data in I/O operations, regardless of the specific type of data.  It provides consistency.  Reading data in from a file should have the same mechanics as reading data in from a JSON blob, for example.  

Coming from using interpreted programming languages, this feels a bit unintuitive, but it's just part of adapting to a different design philosophy.  Speaking of philosophy, I've got some thoughts percolating about how the programming language is a teacher.

My programming style as a Python developer is very much a "poke at it and see what it does" philosophy.  Lots of use of the interactive debugger.  Printing values out, Parsing out json and examining the contents.  Etc.  The snippet above is very much in this camp.  But Go is about data contracts.  It is probably a better idea to learn how the data should be shaped by looking at the data contract, as opposed to looking at the data itself.
