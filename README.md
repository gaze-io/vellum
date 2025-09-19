# ![vellum](docs/logo.png) vellum

[![vellumCI](https://github.com/gaze-io/vellum/actions/workflows/build.yml/badge.svg)](https://github.com/gaze-io/vellum/actions/workflows/build.yml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

#### **IMPORTANT:** The original repository is [here](https://github.com/blevesearch/vellum).

A Go library implementing an FST (finite state transducer) capable of:
  - mapping between keys ([]byte) and a value (uint64)
  - enumerating keys in lexicographic order

Some additional goals of this implementation:
 - bounded memory use while building the FST
 - streaming out FST data while building
 - mmap FST runtime to support very large FTSs (optional)

## Usage

### Building an FST

To build an FST, create a new builder using the `New()` method.  This method takes an `io.Writer` as an argument.  As the FST is being built, data will be streamed to the writer as soon as possible.  With this builder you **MUST** insert keys in lexicographic order.  Inserting keys out of order will result in an error.  After inserting the last key into the builder, you **MUST** call `Close()` on the builder.  This will flush all remaining data to the underlying writer.

In memory:
```go
  var buf bytes.Buffer
  builder, err := vellum.New(&buf, nil)
  if err != nil {
    log.Fatal(err)
  }
```

To disk:
```go
  f, err := os.Create("/tmp/vellum.fst")
  if err != nil {
    log.Fatal(err)
  }
  builder, err := vellum.New(f, nil)
  if err != nil {
    log.Fatal(err)
  }
```

**MUST** insert keys in lexicographic order:
```go
err = builder.Insert([]byte("cat"), 1)
if err != nil {
  log.Fatal(err)
}

err = builder.Insert([]byte("dog"), 2)
if err != nil {
  log.Fatal(err)
}

err = builder.Insert([]byte("fish"), 3)
if err != nil {
  log.Fatal(err)
}

err = builder.Close()
if err != nil {
  log.Fatal(err)
}
```

### Using an FST

After closing the builder, the data can be used to instantiate an FST.  If the data was written to disk, you can use the `Open()` method to mmap the file.  If the data is already in memory, or you wish to load/mmap the data yourself, you can instantiate the FST with the `Load()` method.

Load in memory:
```go
  fst, err := vellum.Load(buf.Bytes())
  if err != nil {
    log.Fatal(err)
  }
```

Open from disk:
```go
  fst, err := vellum.Open("/tmp/vellum.fst")
  if err != nil {
    log.Fatal(err)
  }
```

Get key/value:
```go
  val, exists, err = fst.Get([]byte("dog"))
  if err != nil {
    log.Fatal(err)
  }
  if exists {
    fmt.Printf("contains dog with val: %d\n", val)
  } else {
    fmt.Printf("does not contain dog")
  }
```

Iterate key/values:
```go
  itr, err := fst.Iterator(startKeyInclusive, endKeyExclusive)
  for err == nil {
    key, val := itr.Current()
    fmt.Printf("contains key: %s val: %d", key, val)
    err = itr.Next()
  }
  if err != nil {
    log.Fatal(err)
  }
```

### How does the FST get built?

A full example of the implementation is beyond the scope of this README, but let's consider a small example where we want to insert 3 key/value pairs.

First we insert "are" with the value 4.

![step1](docs/demo1.png)

Next, we insert "ate" with the value 2.

![step2](docs/demo2.png)

Notice how the values associated with the transitions were adjusted so that by summing them while traversing we still get the expected value.

At this point, we see that state 5 looks like state 3, and state 4 looks like state 2.  But, we cannot yet combine them because future inserts could change this.

Now, we insert "see" with value 3.  Once it has been added, we now know that states 5 and 4 can longer change.  Since they are identical to 3 and 2, we replace them.

![step3](docs/demo3.png)

Again, we see that states 7 and 8 appear to be identical to 2 and 3.

Having inserted our last key, we call `Close()` on the builder.

![step4](docs/demo4.png)

Now, states 7 and 8 can safely be replaced with 2 and 3.

For additional information, see the references at the bottom of this document.

### What does the serialized format look like?

We've broken out a separate document on the [vellum disk format v1](docs/format.md).

### Can I use this with Unicode strings?

Yes, however this implementation is only aware of the byte representation you choose.  In order to find matches, you must work with some canonical byte representation of the string.  In the future, some encoding-aware traversals may be possible on top of the lower-level byte transitions.

## Related Work

 - [mafsa](https://github.com/smartystreets/mafsa)
 - [BurntSushi/fst](https://github.com/BurntSushi/fst)
