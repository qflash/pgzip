pgzip
=====

Go parallel gzip compression/decompression. This is a fully gzip compatible drop in replacement for "compress/gzip".

This will split compression into blocks that are compressed in parallel. This can be useful for compressing big amounts of data. The output is a standard gzip file.

The gzip decompression is modified so it decompresses ahead of the current reader. This means that reads will be non-blocking if the decompressor can keep ahead of your code reading from it. CRC calculation also takes place in a separate goroutine.

You should only use this if you are (de)compressing big amounts of data, say **more than 1MB** at the time, otherwise you will not see any benefit, and it will likely be faster to use the internal gzip library.

It is important to note that this library creates and reads *standard gzip files*. You do not have to match the compressor/decompressor to get the described speedups, and the gzip files are fully compatible with other gzip readers/writers.

A golang variant of this is [bgzf](http://godoc.org/code.google.com/p/biogo.bam/bgzf), which has the same feature, as well as seeking in the resulting file. The only drawback is a slightly bigger overhead compared to this and pure gzip. See a comparison below.

[![GoDoc][1]][2] [![Build Status][3]][4]

[1]: https://godoc.org/github.com/klauspost/pgzip?status.svg
[2]: https://godoc.org/github.com/klauspost/pgzip
[3]: https://travis-ci.org/klauspost/pgzip.svg
[4]: https://travis-ci.org/klauspost/pgzip

Installation
====
```go get github.com/klauspost/pgzip```

Usage
====
[Godoc Doumentation](https://godoc.org/github.com/klauspost/pgzip)

To use as a replacement for gzip, exchange 

```import "compress/gzip"``` 
with 
```import gzip "github.com/klauspost/pgzip"```.

## Compression
To change the block size, use the added (*pgzip.Writer).SetConcurrency(blockSize, blocks int) function. With this you can control the approximate size of your blocks, as well as how many you want to be processing in parallel. Default values for this is SetConcurrency(250000, 16), meaning blocks are split at 250000 bytes and up to 16 blocks can be processing at once before the writer blocks.


Example:
```
var b bytes.Buffer
w := gzip.NewWriter(&b)
w.SetConcurrency(100000, 10)
w.Write([]byte("hello, world\n"))
w.Close()
```

To get any performance gains, you should at least be compressing more than 1 megabyte of data at the time.

You should at least have a block size of 100k and at least a number of blocks that match the number of cores your would like to utilize, but about twice the number of blocks would be the best.

Another side effect of this is, that it is likely to speed up your other code, since writes to the compressor only blocks if the compressor is already compressing the number of blocks you have specified. This also means you don't have worry about buffering input to the compressor.

## Decompression

Decompression works similar to compression. That means that you simply call pgzip the same way as you would call [gzip](http://golang.org/pkg/encoding/json/). 

The only difference is that if you want to specify your own readahead, you have to use `pgzip.NewReaderN(r io.Reader, blockSize, blocks int)` to get a reader with your custom blocksizes. The `blockSize` is the size of each block decoded, and `blocks` is the maximum number of blocks that is decoded ahead.

See [Example on playground](http://play.golang.org/p/uHv1B5NbDh)

Performance
====
## Compression
Compression cost is usually about 0.2% with default settings with a block size of 250k.

Example with GOMAXPROC set to 4 (dual core with 2 hyperthreads)

Compressor  | MB/sec   | speedup | size | size overhead
------------|----------|---------|------|---------
[gzip](http://golang.org/pkg/encoding/json/) (golang) | 15.082MB/s | 1.0x | 6.405.193 | 0%
[pgzip](https://github.com/klauspost/pgzip) (golang) | 26.736MB/s|1.8x | 6.421.585 | 0.2%
[bgzf](http://godoc.org/code.google.com/p/biogo.bam/bgzf) (golang) | 29.525MB/s | 1.9x | 6.875.913 | 7.3%

## Decompression

The decompression speedup is there because it allows you to do other work while the decompression is taking place.

In the example above, the numbers are as follows on a 4 CPU machine:

Decompressor | Time | Speedup
-------------|------|--------
[gzip](http://golang.org/pkg/encoding/json/) (golang) | 1m28.85s | 0%
[pgzip](https://github.com/klauspost/pgzip) (golang) | 43.48s | 104%

But wait, since gzip decompression is inherently singlethreaded (aside from CRC calculation) how can it be more than 100% faster?  Because pgzip due to its design also acts as a buffer. When using ubuffered gzip, you are also waiting for io when you are decompressing. If the gzip decoder can keep up, it will always have data ready for your reader, and you will not be waiting for input to the gzip decompressor to complete.

This is pretty much an optimal situation for pgzip, but it reflects most common usecases for CPU intensive gzip usage.
