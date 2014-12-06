Buffer
==========

Documentation: http://godoc.org/github.com/djherbis/buffer

Usage
------------

The following buffers provide simple unique behviours which when composed can create complex buffering strategies. For use with github.com/djherbis/nio for Buffered io.Pipe and io.Copy implementations.

For example: 

```go
import (
  "github.com/djherbis/buffer"
  "github.com/djherbis/nio"
  
  "io/ioutil"
)

// Buffer 32KB to Memory, after that buffer to 100MB chunked files
buf := buffer.NewUnboundedBuffer(32*1024, 100*1024*1024)
nio.Copy(w, r, buf) // Reads from r, writes to buf, reads from buf writes to w (concurrently).

// Buffer 32KB to Memory, discard overflow
buf = buffer.NewSpill(32*1024, ioutil.Discard)
nio.Copy(w, r, buf)
```

Supported Buffers
------------

#### Bounded Buffers ####

Memory: Wrapper for bytes.Buffer

File: File-based buffering. The file never exceeds Cap() in length, no matter how many times its written/read from. It accomplishes this by "wrapping" around the fixed max-length file when the data gets too long but there is available freed space at the beginning of the file. The file deletes itself when empty.

```go
import (
  "github.com/djherbis/buffer"
)

// Create a File-based Buffer with max size 100MB
buf := buffer.NewFile(100*1024*1024)
```

Multi: A fixed length linked-list of buffers. Each buffer reads from the next buffer so that all the buffered data is shifted upwards in the list when reading. Writes are always written to the first buffer in the list whose Len() < Cap().

```go
import (
  "github.com/djherbis/buffer"
)

mem  := buffer.New(32*1024)
file := buffer.NewFile(100*1024*1024)

// Buffer composed of 32KB of memory, and 100MB of file.
buf := buffer.NewMulti(mem, file)
```

#### Unbounded Buffers ####

Partition: A queue of buffers. Writes always go to the first buffer in the queue which isn't full. If all buffers are full, a new buffer is "pushed" to the end of the queue (generated by a user-given function). Reads come from the first buffer, when the first buffer is emptied it is "poped" off the queue.

```go
import (
  "github.com/djherbis/buffer"
)

// Create 32 KB sized-chunks of memory as needed to expand/contract the buffer size.
buf := buffer.NewPartition(func() buffer.Buffer{
  return buffer.New(32*1024)
})

// Create 100 MB sized-chunks of files as needed to expand/contract the buffer size.
buf = buffer.NewPartition(func() buffer.Buffer{
  return buffer.NewFile(100*1024*1024)
})
```

Ring: A single buffer which begins overwriting the oldest buffered data when it reaches its capacity.

```go
import (
  "github.com/djherbis/buffer"
)

// Create a File-based Buffer with max size 100MB
file := buffer.NewFile(100*1024*1024)

// If buffered data exceeds 100MB, overwrite oldest data as new data comes in
buf := buffer.NewRing(file) // requires BufferAt interface.
```

Spill: A single buffer which when full, writes the overflow to a given io.Writer.
-> Note that it will actually "spill" whenever there is an error while writing, this should only be a "full" error.

```go
import (
  "github.com/djherbis/buffer"
  "github.com/djherbis/nio"
  
  "io/ioutil"
)

// Buffer 32KB to Memory, discard overflow
buf := buffer.NewSpill(32*1024, ioutil.Discard)
nio.Copy(w, r, buf)
```

#### Empty Buffer ####

Discard: Reads always return EOF, writes goto ioutil.Discard.

```go
import (
  "github.com/djherbis/buffer"
)

// Reads will return io.EOF, writes will return success (nil error, full write) but no data was written.
buf := buf.NewDiscard()
```

Installation
------------
```sh
go get github.com/djherbis/buffer
```
