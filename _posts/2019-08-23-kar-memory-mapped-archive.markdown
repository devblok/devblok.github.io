---
title:  "Kar - memory mapped archive"
date:   2019-09-01 11:00:00
categories: [go]
tags: [go,mmap,gamedev,archive,io]
---

There is a little game engine project I'm wokring on currently, written in Go(perhaps desirves a write up later). Game engines have always fascinated me in their complexity. I've always wondered what they do under the hood and how they're built. And also wanted to build my own ever since my mid teens. Obviously, you don't just build an engine, and not while as inexperienced and poor coder as I was at the time, I still tried though.

Anyways, fast-forward 10 years, I've decided to take a crack at it again, using a language I really want to master - Go. Among other things and pains of cgo, one of my goals is to implement resource-streaming from disk. I figured this is a good place to write a package, and practice how to do good io on golang.

I know I will want a way to memory-map an archive and do random-access and concurrent reads, that the engine will do whenever it's loading a particular model. The idea is to organise files much like a tar archive does, except that there's an index detailing where each file is located in memory and how big it is. Knowing that, would allow me to make a grab of that particular spot, decompress it, and immediately use it, everything on the fly.

Tar has a disadvantage though, and the name suggests it: it's a tape archive file. It's designed to be appended to the end, each file has a header, and it does not have an index. Can't know the locations of files without traversing the whole file is search of them. It's a possibility to do it at the start of the application, but that's not ideal, I want mine to work as fast as possible.

Introducing the kar package: a prebuilt, non-appendable and versionable archive that is well suited to be memory-mapped. The primary goals are pretty clear:
- Know the locations, sizes and names in advance
- Read a file in one go and decompress on the fly, with as little allocations as possible
- Build files in a non-appendable way, do not implement the simple Writer

Let's start with the builder:
```go
// io.go
type WriterTo interface {
	WriteTo(Writer) (n int64, err error)
}

// builder.go
type Builder struct {
	io.WriterTo
}

func (Builder) Add(string, []byte) {
	// ...
}

func (Build) WriteTo(w io.Writer) (n int64, err error) {
	// ...
}
```
The intent is to have the Builder finalize the build with a WriteTo, that will bundle all the files into the archive format and write it out to the given io.Writer. I did not spend too much time here, only made sure that files are safe to add concurrently, so as much of the machine could be used for compression work. All added files are initially compressed and put in a tmeporary folder as separate files, later, when WriteTo is executed, the archive header is written and all the separate compressed files are bundled into a single archive.

Next, implementing the reader. I took some more time figuring out how I want to lay this piece of code out, what interfaces should I use. The simple Reader is a must, others are optional, so I just went with that. Though, I also implemented a ReadAll, similar in function and appearance to ioutil.ReadAll. Gives some simple use cases with not much code.
```go
type Archive struct {
	reader io.ReaderAt
	header Header
}

func (a *Archive) GetFileInfo(name string) (IndexEntry, error) {
	// get info
}

func (a *Archive) ReadAll(name string) ([]byte, error) {
	// read entire file
}

func (a *Archive) Open(name string) (*Reader, error) {
	// make and return a section reader
}

type Reader struct {
	reader io.Reader
}

func (r *Reader) Read(p []byte) (int, error) {
	return r.reader.Read(p)
}
```
I managed to avoid implementing Read myself, by useing a SectionReader from the io package, I expected to write this part myself, but oh well. Getting separate readers for every file in the archive is a very useful touch. I can subtitute files with readers from a memory mapped archive now, allowing for simpler code elsewhere, but code that will perform great for loading things into that game engine. So how well does it actually perform at this point?

```
goos: linux
goarch: amd64
pkg: github.com/devblok/koru/utility/kar
BenchmarkReadFromMemoryMapped-4              300           5211593 ns/op
BenchmarkReadAllFromMemoryMapped-4           100          12930517 ns/op
PASS
ok      github.com/devblok/koru/utility/kar     3.459s
```
Quite clearly, calling Read operations is much more efficient than calling ReadAll. 5ms time to get the file out of the archive and into use (it will be read directly into gpu memory in best cases) is acceptable to me at this time. I will continue working on it as I need to, but that's it for now. A useful IO utility with just a few hundread lines of code!


