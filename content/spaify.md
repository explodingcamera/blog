+++
title = "Introducing Spaify - Seamless page transitions for your static site"
description = "A new npm package to add seamless page transitions to your static site, with less than 1kb of JavaScript"
date = 2023-05-14

[taxonomies]
tags = ["npm", "typescript", "open source", "javascript", "spa", "static site", "web development"]
+++

By default, when you click a link on a website, the browser will load the new page from the server and replace the current page with the new one. This is the default behaviour for static sites, and it works well. Spaify lets you skip this step and load the new page in the background, so that when you click a link, the new page is already loaded and ready to be displayed. This makes the page transition seamless, and it's a great way to improve the user experience of your site, and progressively enhance it. And the best part is that it's super easy to use, and it's less than 1kb of JavaScript (minified and gzipped).

All you need to do is to import the Spaify script, add `data-spaify-main` to the element that contains the content that you want to replace, and add `data-spaify-ignore` to any link that you don't want to be handled by Spaify. You can also add `data-spaify-run="once"` to any script tag that you want to run only once, when the page is loaded for the first time, and `data-spaify-run="always"` to any script tag that you want to run every time the page is loaded. Here's an example:

```html
<head>
  <!-- import spaify with default options -->
  <script type="module" src="https://esm.sh/spaify/default"></script>

  <!-- or import spaify with custom options -->
  <script type="module">
    import spaify from "https://esm.sh/spaify";
    spaify();
  </script>

  <script>
    console.log("this script will run on the first page load only");
  </script>

  <script data-spaify-run="once">
    console.log(
      "this script will run every time this page is loaded / navigated to for the first time"
    );
  </script>

  <script data-spaify-run="always">
    console.log(
      "this script will run every time this page is loaded / navigated to"
    );
  </script>
</head>

<body>
  <!-- only the content inside this div will be replaced -->
  <div data-spaify-main>
    <!-- all script tags in here default to a-spaify-run="always" -->

    <!-- this will be fetched via fetch and inserted into the DOM -->
    <a href="/page1">page 1</a>

    <!-- these will be handled by the browser -->
    <a href="/page2" data-spaify-ignore>page 2</a>
    <a href="https://example.com">external link</a>
  </div>

  <!-- all script here are like in the head -->
</body>
```

With the small amount of JavaScript that Spaify adds to your site, you can also include it inlined in your HTML, so that you don't have to make an extra request to load it:

```html
<script type="module">
  // include the code from https://esm.sh/v120/spaify/es2022/default.js here
</script>
```

If you want to see it in action, just click on any link on this site. And if you want to see the code, you can check out the [source code](https://github.com/explodingcamera/esm/tree/main/packages/spaify) on GitHub (licensed under the MIT license).

# Previous Work

There are a few other libraries that do something similar, but they all have some drawbacks. For example, [barba.js](https://barba.js.org/) is a great library, but it's quite heavy (about 10kb) and it requires you to write a lot of code to get it working. There's also the (unfortunately named) [turbo](https://github.com/hotwired/turbo), which has a lot of features, but it's also quite heavy (about 20kb) and it's not very easy to use. Spaify specifically targets static sites, and it's designed to be as simple as possible to use. Of course, if you need more features, like animations, you can always use one of the other libraries, but for many of my sites, Spaify was a drop-in replacement.
