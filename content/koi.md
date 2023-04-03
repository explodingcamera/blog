+++
title = "building a fast image format for fun and profit"
date = 2023-03-21
draft = true

[taxonomies]
tags = ["qoi", "koi", "image"]
+++

I've been working on a new image format called [koi](https://github.com/explodingcamera/koi) and I wanted to share some of the details.

It's a lossless image format based on ideas from [qoi](https://phoboslab.org/log/2021/11/qoi-fast-lossless-image-compression) and [qoir](https://nigeltao.github.io/blog/2022/qoir.html) that is designed to use a small amount of memory and be fast to decode. I have exactly zero experience with image formats, so this is a learning experience for me. In the end, I managed to get it to a point where it's usable and competetive with PNG in certain situations.

# The basics

The format starts with a header that contains the width and height of the image, the number of channels, and the number of bits per channel. The header is encoded using bson, which is a binary encoding of json. The header is followed by a series of frames, which are compressed chunks of pixel data.

Koi supports 1-4 channels (grayscale, rgba, and both with alpha) and 8 bits per channel. Pixels are encoded in chunks that can be 1-4 bytes long, where the first two bits of the first byte indicate the chunk type and length.

All chunk types of QOI are supported, with the exception of run-length encoding, which is not used in koi. Instead, koi utilizes lz4 which does a better job of compressing runs of pixels through dictionary-matching.

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
      Slighly modified from the hash function used in <a href="https://qoiformat.org/qoi-specification.pdf">qoi</a>
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
      The difference between the current pixel's alpha value and the previous pixel's alpha value. Stored as unsigned integers with a bias of 2. Lengths above 59 are illegal, as they are occupied by other chunk types.
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
      Grayscale pixel, can also be also be used in images with more than one channel if the pixel is the same in all channels. Followed by a single byte containing the grayscale value.
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

# Performance

To compare the performance of koi to other image formats, I wrote a simple benchmark which is available [here](https://github.com/explodingcamera/koi-rs/tree/main/koi-bench). The benchmark decodes a series of images and measures the time it takes to decode them. The images are taken from the Qoi Benchmark Suite, which is a collection of images that are used to test the performance of qoi. The benchmark is run on a Ryzen 7 5800X with 32GB of RAM and images are decoded directly from and into memory.
