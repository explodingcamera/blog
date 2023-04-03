+++
title = "building an image format for fun and profit"
date = 2023-04-02

[taxonomies]
tags = ["koi", "image"]
+++

I've been working on a new image format called [koi](https://github.com/explodingcamera/koi) and I wanted to share some details.

It's a lossless image format based on ideas from [qoi](https://phoboslab.org/log/2021/11/qoi-fast-lossless-image-compression) and [qoir](https://nigeltao.github.io/blog/2022/qoir.html) that is designed to use a small amount of memory and be fast to decode. I have exactly zero experience with image formats, so this is a learning experience for me. In the end, I managed to get it to a point where it's usable and competitive with PNG in certain situations.

# The basics

To identify the file format, the first 8 bytes must be `KOI \xF0\x9F\x99\x82`. The file header is encoded using BSON. All numbers are u32 in i32 fields. The following fields are required:

- `v` (version): The file format version. Currently always `0`.
- `e` (exif): The EXIF data as a byte array.
- `w` (width): The image width in pixels. Must be greater than zero.
- `h` (height): The image height in pixels. Must be greater than zero.
- `c` (channels): The number of channels. Must be `1`, `2`, `3` or `4`.
- `x` (compression): The compression algorithm. Must be `0` (none) or `1` (LZ4 frame).

The header is followed by a series of frames, which are compressed chunks of pixel data.

Koi supports 1-4 channels (grayscale, rgba, and both with alpha) and 8 bits per channel. Pixels are encoded in chunks that can be 1-4 bytes long, in which the first two bits of the first byte indicate the chunk type and length.

All chunk types of [QOI](https://phoboslab.org/log/2021/11/qoi-fast-lossless-image-compression) are supported, with the exception of run-length encoding, which is not used in koi. Instead, koi utilizes lz4 which does a better job of compressing runs of pixels through dictionary-matching.

The following chunk types are added additionally:

<table>
  <tr>
    <td style="width: min-content;">
      <pre><code>┌─ OP_INDEX ──────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────────────────│
│  0  0 │      index      │
└───────┴─────────────────┘</code></pre></td>
    <td><p>
      Index chunk, the last 6 bits are an index into a running list of the last pixel values, which are indexed by the following hash function:<br/>
      <code>(r * 3 + g * 5 + b * 7 + a * 11) % 62</code><br/><br/>
      Slightly modified from the hash function used in <a href="https://qoiformat.org/qoi-specification.pdf">qoi</a>
    </p></td>
  </tr>
  <tr>
    <td style="width: min-content;">
      <pre><code>┌─ OP_DIFF_ALPHA ─────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────────────────│
|  1  1 |      diff       |
└───────┴─────────────────┘</code></pre></td>
    <td><p>
      The difference between the current pixel's alpha value and the previous pixel's alpha value. Stored as unsigned integers with a bias of 2. Lengths above 59 are illegal since they are used by other chunk types.
    </p></td>
  </tr>
  <tr>
    <td style="width: min-content;">
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
    <td style="width: min-content;">
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
</table>

These chunks are then encoded using the [lz4](https://lz4.github.io/lz4/) compression algorithm in frames that can be up to 64kb in size, and include a checksum of the uncompressed data.

The last frame is followed by 8 special bytes `\x00\x00\x00\x00\xF0\x9F\x99\x82`.

# Performance

To compare the performance of koi to other image formats, I wrote a simple benchmark which is available [here](https://github.com/explodingcamera/koi-rs/tree/main/koi-bench). The benchmark decodes a series of images and measures the time it takes to decode them. The images are taken from the Qoi Benchmark Suite, which is a collection of images that are used to test the performance of qoi. The benchmark is run on a Ryzen 7 5800X with 32GB of RAM and images are decoded directly from and into memory.

```
┌─────────────────────────────────────┐
│ Overall                             │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │155382ms │  7096ms │  0.25 │
│ PngFast │  4046ms │  5724ms │  0.30 │
│ Koi     │ 21082ms │  6954ms │  0.27 │
└─────────┴─────────┴─────────┴───────┘
```

When looking at the overall benchmark results, Koi actually sits somewhere in the middle between the different png profiles. Koi actually outperforms the default png profile in almost every category, except
for photos with a lot of detail:

```
Koi's worst results
┌─────────────────────────────────────┐
│ images/photo_tecnick                │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │ 27624ms │  1304ms │  0.55 │
│ PngFast │   624ms │   843ms │  0.58 │
│ Koi     │  2953ms │  1283ms │  0.60 │
└─────────┴─────────┴─────────┴───────┘
```

The best case scenario for Koi is with images with a lot of transparency and grey colors, like in the icon_512 suite:

```
┌─────────────────────────────────────┐
│ images/icon_512                     │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │  2088ms │   182ms │  0.05 │
│ PngFast │   132ms │   210ms │  0.10 │
│ Koi     │   756ms │   155ms │  0.06 │
└─────────┴─────────┴─────────┴───────┘
```

And for screenshots, Koi actually outperforms both png profiles in every category:

```
┌─────────────────────────────────────┐
│ images/screenshot_web               │
├─────────┬─────────┬─────────┬───────┤
│ format  │ encode  │ decode  │ ratio │
├─────────┼─────────┼─────────┼───────┤
│ Png     │  4824ms │   404ms │  0.08 │
│ PngFast │   265ms │   389ms │  0.12 │
│ Koi     │  1547ms │   356ms │  0.07 │
└─────────┴─────────┴─────────┴───────┘
```

# Future Work

Encoding and Desoding still have a ton of room for optimization, especially in the encoding side. Also, the current implementation of the decoder can produce some artifacts when decoding images with a lot of detail due to collisions in the hash table. I was not able to reliably test memory usage, but it should be on par or better than most png encoders. In the future, I'd like to also add some more image formats to the benchmark, like WebP and Qoi.

# Conclusion

Creating a new image format is actually a lot of fun, and I learned a lot about image compression and the different image formats that are out there. It's totally possible to create something that works well for special use cases like icons or game assets, and simply putting LZ4 on top of it makes it a lot easier to implement. I'm not sure if Koi will ever be used in production, but I'm happy with the results so far.

The source code for the encoder, decoder and benchmark is available on [GitHub](https://github.com/explodingcamera/koi-rs) and licensed under the ISC license.
