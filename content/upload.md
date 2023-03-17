+++
title = "stop building upload apis"
date = 2023-03-14
draft = false

[taxonomies]
tags = ["s3"]
+++

Instead of reinventing the wheel for every project and spending time understanding the intricacies of multipart uploads, you can upload files directly to S3/R2/whatever using a pre-signed URL.
You don't even need the full AWS SDK, you can use the [aws4fetch](https://github.com/mhart/aws4fetch).

Especially if you are using serverless, this is a great way to save time and money.

```ts
import { AwsClient } from "aws4fetch";

const r2 = new AwsClient({
  accessKeyId: process.env.ACCESS_KEY,
  secretAccessKey: process.env.ACCESS_SECRET,
});

export const getPresignedSignedUrl = async (
  key: string,
  contentLength: number,
  contentType: string = "image/jpeg"
) => {
  const url = new URL(
    `https://something.r2.cloudflarestorage.com/something/${key}`
  );

  // Only sign the URL if the content length is less than 10MB
  if (contentLength > 10000000) return undefined;

  // Specify a custom expiry for the presigned URL, in seconds
  url.searchParams.set("X-Amz-Expires", "3600");

  const signed = await r2.sign(
    new Request(url, {
      method: "PUT",
      headers: {
        "Content-Type": contentType,
        "Content-Length": contentLength.toString(),
      },
    }),
    {
      aws: { signQuery: true, allHeaders: true },
    }
  );

  return signed.url;
};
```

> Note that this will leak your account id and bucket name in the URL which should be fine for most use cases.
