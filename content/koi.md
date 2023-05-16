+++
title = "Koi, the kinda okay image format"
description = "Creating a new lossless image format to learn about image compression and some notes on performance optimization"
date = 2023-04-02
updated = 2023-05-15

[taxonomies]
tags = ["koi", "image"]
+++

{% quote (class="info")%}
This post has been updated to reflect changes in the koi file format. The original post can be found [here](https://github.com/explodingcamera/blog/blob/8a3c81d81a251b4ac64714c4e5f60a2c07376551/content/koi.md).
{% end %}

I've been working on a new image format called [koi](https://github.com/explodingcamera/koi) - **the kinda okay image format**  
and I wanted to share some details.

It's a lossless image format based on ideas from [qoi](https://phoboslab.org/log/2021/11/qoi-fast-lossless-image-compression) and [qoir](https://nigeltao.github.io/blog/2022/qoir.html) that is designed to use a small amount of memory and be fast to decode. I have exactly zero experience with image formats, so this is a learning experience for me. As it currently stands, it outperforms many other image formats in terms of compression ratio and decoding speed on images with a lot of flat colors, but more on that later.

# The basics

To start, let's look at how the file format is structured.

To identify the file format, the first 8 bytes must be `KOI `. Following that is a header with metadata about the image. The header is a series of key-value pairs, encoded using Binary JSON (BSON). The following keys are supported:

- `v` (version): The file format version
- `w` (width): The image width in pixels. Must be greater than zero. (unsigned 64-bit integer)
- `h` (height): The image height in pixels. Must be greater than zero. (unsigned 64-bit integer)
- `c` (channels): The number of channels. Must be `1`, `2`, `3` or `4`.
- `x` (compression): The compression algorithm. Must be `0` (none) or `1` (LZ4).
- `s` (color space): The color space. Must be `0` (sRGB with linear alpha), `1` (all channels linear)
  There's also two optional keys:

- `e` (exif): The EXIF data as a byte array.
- `b` (block size): The size of blocks, smaller blocks allow for parallel processing, larger blocks allow for better compression. Used by the encoder for the maximum size of uncompressed blocks. Can be up to 199992 bytes. (unsigned 32-bit integer)

The header is followed by a series of blocks, each of which is a chunk of compressed image data. Each block is prefixed with a small header that contains the length of the block and the number of pixels it contains. These are stored as 4-byte unsigned integers, encoded in little endian order.

Koi supports 1-4 channels (grayscale, rgba, and both with alpha) and stores 8 bits per channel.

The image data is stored as a series of chunks, each of which is one to five bytes long and only corresponds to a single pixel. This is a pretty big departure from most image formats, which store data in blocks of 4x4 or 8x8 pixels. The reason for this is that it allows for very fast decoding, since the decoder only needs to keep track of the last pixel value. Initially, koi also had support for run-length encoding and a cache of recently used pixel values, but I removed those features because they didn't improve the compression ratio in my benchmark suite and made decoding and encoding a lot slower.

The following chunk types are currently supported, with space for more in the future:

<style>
  .chunk-table {
    /* overflow: auto;
    display: flex;
    padding: 0;
    /* border: none; */ */
  }
  .chunk-table pre {
    line-height: 1.2;
    display: flex;
    justify-content: center;
  }
  .chunk-table tbody {
    display: flex;
    flex-direction: column;
  }
  .chunk-table tr {
    display: flex;
    flex-wrap: wrap;
  }
  .chunk-table td:first-child {
    flex: 1;
  }
  .chunk-table td:last-child {
    flex: 999;
    min-width: 15rem;
  }
</style>

<table class="chunk-table">
  <tr>
    <td>
      <pre><code>┌─ OP_DIFF ───────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────────────────│
│  0  0 │      diff       │
└───────┴─────────────────┘</code></pre></td>
    <td><p>
      This chunk can store the color difference between the current pixel and the previous pixel in the image more efficiently when the difference is small. The difference is stored as a unsigned integer with a bias of 2. Similar to QOI's <a href="https://qoiformat.org/qoi-specification.pdf">OP_DIFF</a>
      <br/>
    </p></td>
  </tr>
   <tr>
    <td>
      <pre><code>┌─ OP_LUMA ───────────────┬────────────────────────┐
│         Byte[0]         │      Byte[1]           │
│  7  6  5  4  3  2  1  0 │ 7  6  5  4  3  2  1  0 │
│───────┼─────────────────│───────────┼────────────│
│  0  0 │       dg        │  dr - dg  │  db - dg   │
└───────┴─────────────────┴───────────┴────────────┘</code></pre></td>
    <td><p>
      Similar to OP_DIFF, but stores the difference between the current pixel and the previous pixel's luma value.
      <br/>
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_DIFF_ALPHA ─────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────────────────│
│  1  1 │      diff       │
└───────┴─────────────────┘</code></pre></td>
    <td><p>
      The difference between the current pixel's alpha value and the previous pixel's alpha value. Stored as unsigned integers with a bias of 2. Lengths above 59 are illegal since they are used by other chunk types.
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_SAME ───────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│─────────────────────────│
│  1  0  0  0  0  0  0  0 │
└─────────────────────────┘</code></pre></td>
    <td><p>
      Ignore the current pixel, and use the previous pixel instead, this is useful for images with large areas of the same color and is easily compressible.
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_GRAY ───────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│─────────────────────────│
│  1  1  1  1  1  1  0  0 │
└─────────────────────────┘</code></pre></td>
    <td><p>
      Grayscale pixel, can also be also be used in images with more than one channel if the color values are the same across all channels. Followed by a single byte containing the grayscale value.
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_GRAY_ALPHA ─────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│─────────────────────────│
│  1  1  1  1  1  1  0  1 │
└─────────────────────────┘</code></pre></td>
    <td><p>
      Grayscale pixel with alpha, can also be also be used in images with more than one channel if the pixel is the same in all channels. Followed by two bytes containing the grayscale value and alpha value respectively.
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_RGB ────────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│─────────────────────────│
│  1  1  1  1  1  1  1  0 │
└─────────────────────────┘</code></pre></td>
    <td><p>
      RGB pixel, followed by three bytes containing the red, green and blue values.
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_RGBA ───────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│─────────────────────────│
│  1  1  1  1  1  1  1  1 │
└─────────────────────────┘</code></pre></td>
    <td><p>
      RGBA pixel, followed by four bytes containing the red, green, blue and alpha values.
    </p></td>
  </tr>
  <tr>
    <td>
      <pre><code>┌─ OP_SAME ───────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│─────────────────────────│
│  0  0  0  0  0  0  0  0 │
└─────────────────────────┘</code></pre></td>
    <td><p>
      The current pixel is the same as the previous pixel.
    </p></td>
  </tr>
</table>

While developing the format, I also experimented with a few other chunk types, but more often than not they ended up being less compressible and slower to decode than the ones listed above.

These chunks are compressed using [lz4](https://lz4.github.io/lz4/), which is, while not the most efficient compression algorithm, very fast to decode and encode. When encoding, the reference implementation allows choosing between `lz4-flex`, a pure rust implementation, and `lz4`/`lz4-hc`, which are bindings to the C implementation and specify the compression level. When decoding, the reference implementation uses `lz4-flex`, since it is faster than the C implementation in my benchmarks.

The last block is followed by 4 zero bytes to indicate the end of the stream, so block sizes are not allowed to be 0.

# Performance

To compare the performance of koi to other image formats, I wrote a simple benchmark which is available [here](https://github.com/explodingcamera/koi-rs/tree/main/koi-bench). The benchmark decodes a series of images and measures the time it takes to decode them. The images are taken from the [Qoi Benchmark Suite](https://qoiformat.org/benchmark/). My results were achieved on a Ryzen 7 5800X with 32GB of RAM and images are decoded directly from and into memory.

```
┌─────────────────────────────────────┐
│ Overall                             │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │ 152.64s │   6.29s │  0.25 │
│ PngFast │   3.73s │   5.32s │  0.30 │
│ Koi     │  26.20s │   4.21s │  0.26 │
│ KoiFast │   7.80s │   4.11s │  0.28 │
│ Qoi     │   4.65s │   3.39s │  0.28 │
└─────────┴─────────┴─────────┴───────┘
```

When looking at the overall benchmark results, Koi sits somewhere in the middle between the different png profiles. Koi actually outperforms the other formats in a lot of categories, except
for photos with a lot of detail, where it is still pretty competetive considering the simple architecture of the format.

```
Koi's worst results
┌─────────────────────────────────────┐
│ images/photo_tecnick                │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │ 27133ms │  1151ms │  0.55 │
│ PngFast │   551ms │   799ms │  0.58 │
│ Koi     │  5065ms │   709ms │  0.60 │
│ KoiFast │  1246ms │   675ms │  0.62 │
│ Qoi     │   829ms │   630ms │  0.60 │
└─────────┴─────────┴─────────┴───────┘
```

The best case scenario for Koi is with images with a lot of transparency and grey colors, like in the icon_512 suite:

```
┌─────────────────────────────────────┐
│ images/icon_512                     │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │  2005ms │   122ms │  0.05 │
│ PngFast │   102ms │   149ms │  0.10 │
│ Koi     │   439ms │   118ms │  0.05 │
│ KoiFast │   174ms │   123ms │  0.06 │
│ Qoi     │   109ms │    74ms │  0.08 │
└─────────┴─────────┴─────────┴───────┘
```

And for screenshots, Koi can also hold its own:

```
┌─────────────────────────────────────┐
│ images/screenshot_web               │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │  4593ms │   277ms │  0.08 │
│ PngFast │   172ms │   274ms │  0.12 │
│ Koi     │   886ms │   182ms │  0.07 │
│ KoiFast │   359ms │   185ms │  0.08 │
│ Qoi     │   209ms │   146ms │  0.08 │
└─────────┴─────────┴─────────┴───────┘
```

# Safety

I'm assuming untrustworthy input, so the decoder is written in a way that it can't panic or cause undefined behavior. All inputs are checked for a maximum size, so the decoder can't be tricked into allocating a huge amount of memory.

There are a few places where I use `unsafe` code, due to some rust features still being unstable, but I'm trying to keep it to a minimum.

# Performance optimizations

Having a benchmark suite has been very useful for optimizing the format, but profiling the encoder and decoder directly using [perf](https://perf.wiki.kernel.org/index.php/Main_Page) and [hotspot](https://github.com/KDAB/hotspot) also helped to pinpoint bottlenecks in the code.

By comparing the results after small, incremental changes, I was able to improve the performance of both the encoder and decoder by more than 200%. The biggest improvements were made by rearchitecting the hot paths, where a lot of time was spend on the `Write` and `Read` traits. Here, I now instead use mutable slices to write and read data directly into the output buffer without having to keep track of the current position, this alone improved the performance by 50% over the previous version where I kept track of the current position using a local variable.

```rust
// Old version (simplified)
let data = [0; 1024];

// Using Cursor instead of a local variable was even slower, at least in this case
let mut pos = 0;

for chunk in chunks {
  match chunk {
    OP_RGB => {
      let px = Pixel::from(data[pos..pos + 3])
      pos += 3;
    }
  }
}

// New version
let mut data = [0; 1024];

for chunk in chunks {
  let px: Pixel;
  (data, px) = match chunk {
    [OP_RGBA, r, g, b, a, rest @ ..] => (rest, Pixel::<C>::from([*r, *g, *b, *a]))
  }
}
```

In some places, writing ideomatic Rust code also had a big impact on the performance. While the Rust compiler is very good at optimizing code, it can't do magic and sometimes it's necessary to help it out a bit. For example, in the main loop of the decoder, I was using something like `chunks.iter().fold(|chunk| ...).collect()` , which even when refactored to not allocate a new vector for every chunk, was still a lot slower than using loops directly. The promise of zero-cost abstractions didn't hold up here, and while I normally go for readability over performance, in this case it was necessary to make the code a bit more verbose to get the performance I wanted.

A different case where the Rust compiler was able to optimize the code very well was when I started to use `#[inline]` on some of my functions. Here, more often then not, using `#[inline]` decreased the performance of the code, so this is something that should be used with caution and only after profiling.

# Conclusion

Creating a new image format is actually a lot of fun, and It's totally possible to create something that works well for special use cases like icons or game assets. When taking the right shortcuts, it's possible to implement something usable in a few days when utilizing existing compression algorithms and metadata formats, without needing to utilize more advanced compression algorithms. Switching to a more advanced compression algorithm like Zstd or Brotli also enables the format to be purpose built for a specific decompression budget.

I'm happy with the results so far and as a next step I want to add Rust `no_std` support so I can use it in the kernel project I'm working on to test displaying images on the screen.

Some other things I want to look into is adding support for more color spaces, like YCbCr and CMYK, and maybe even support for animation. I also want to look into adding support for more compression algorithms, like LZ4 or Zstd, and maybe even support for lossy compression. Also, I'm not happy with BSON as a metadata format, so I want to look into using something like [MessagePack](https://msgpack.org/index.html) instead. And lastly, there's still a lot of room for improvement on the performance side, and it should be possible to get the performance even faster than for Qoi, at least for the fast profile.

The source code for the encoder, decoder and benchmark is available on [GitHub](https://github.com/explodingcamera/koi-rs) and licensed under the ISC license.
