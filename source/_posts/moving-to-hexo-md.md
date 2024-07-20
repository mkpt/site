---
title: New Year, New Platform
date: 2024-07-18 15:17:31
tags: code
categories: 
  - "2024"
---

# Intro

For the past few years I used Squarespace to blog. It's a slick product, but it felt dishonest, like buying herbs from the grocery store when I can grow my own. I should be able to use a standard static site generator and figure out how to host my own site. Plus I think I miss my technical writer days at AWS when I spent the workday in markdown files.

So, what to do? 

# Setup

Let's move 3 years of writing and images to a [hexo](https://hexo.io/) site.

I recently sold my Mac Mini as I wanted to build a new PC. Sometimes you just get tired of the same old interfaces and want a change, and boy, Windows 11 sure offered a change. 

However, I'm not particulary fond of developing on Windows, so I installed [WSL on Windows](https://ubuntu.com/desktop/wsl) to get back to a more familiar set of commands. There was a year period of my life there (2016-2017?) where I distrohopped regularly -- I used everything from Ubuntu to Arch to my personal favorite [Elementary OS](https://elementary.io/).

The basic plan:

1. Get the content out of Squarespace (text & images).
2. Convert the content into markdown (text & images).
3. Build a site locally using hexo.
4. Deploy it somewhere.

# Get the content out of Squarespace (text & images)

With a rough game plan in mind, it was time to get content from Squarespace to markdown. The only option Squarespace gives you is the option to [migrate from Squarespace to Wordpress](https://support.squarespace.com/hc/en-us/articles/206566687-Exporting-your-site?platform=v6&websiteId=6510e0308bfa4f011936cb7c). I'll work with what I'm given.

Step 1 done.

~~1. Get the content out of Squarespace (text & images).~~
2. Convert the content into markdown (text & images).
3. Build a site locally using hexo.
4. Deploy it somewhere.

# Convert the content into markdown (text & images)

After exporting, I found lonekorean's [wordpress-export-to-markdown](https://github.com/lonekorean/wordpress-export-to-markdown) tool on Github to convert the XML export that I got from Squarespace to .md files.

However, I got this error:

```sh
$ npx wordpress-export-to-markdown
/home/prince/blog/node_modules/@mixmark-io/domino/lib/Element.js:1104
if (globalThis.Symbol?.iterator) {
                      ^
SyntaxError: Unexpected token '.'
    at wrapSafe (internal/modules/cjs/loader.js:915:16)
    at Module._compile (internal/modules/cjs/loader.js:963:27)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1027:10)
    at Module.load (internal/modules/cjs/loader.js:863:32)
    at Function.Module._load (internal/modules/cjs/loader.js:708:14)
    at Module.require (internal/modules/cjs/loader.js:887:19)
    at require (internal/modules/cjs/helpers.js:85:18)
    at Object.<anonymous> (/home/prince/blog/node_modules/@mixmark-io/domino/lib/Document.js:7:15)
    at Module._compile (internal/modules/cjs/loader.js:999:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1027:10)
```
Instead of fighting this just to use `npx`, I cloned the repo and installed it using good ol' `npm`.

Unfortunately, I kept getting this error when trying to run things:
```sh
prince@ringo:~/wordpress-export-to-markdown$ npm install && node index.js

up to date, audited 169 packages in 526ms

15 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

Starting wizard...
? Path to WordPress export file? Squarespace-Wordpress-Export-07-18-2024.xml
? Path to output folder? /site
? Create year folders? Yes
? Create month folders? Yes
? Create a folder for each post? No
? Prefix post folders/files with date? Yes
? Save images attached to posts? Yes
? Save images scraped from post body content? Yes
? Include custom post types and pages? Yes

Parsing...
35 "post" posts found.
2 "page" posts found.

Something went wrong, execution halted early.
TypeError: Cannot read property '0' of undefined
    at /home/prince/wordpress-export-to-markdown/src/parser.js:119:34
    at Array.map (<anonymous>)
    at collectAttachedImages (/home/prince/wordpress-export-to-markdown/src/parser.js:117:4)
    at Object.parseFilePromise (/home/prince/wordpress-export-to-markdown/src/parser.js:26:18)
    at async /home/prince/wordpress-export-to-markdown/index.js:15:16
```
The error was occurring in the `collectAttachedImages` function in `src/parser.js`, where the code attempts to access properties of items that might be undefined. I added checks to ensure that the items being accessed are defined and are in fact arrays.

Was:
```js
function collectAttachedImages(channelData) {
	const images = getItemsOfType(channelData, 'attachment')
		// filter to certain image file types
		.filter(attachment => attachment.attachment_url && (/\.(gif|jpe?g|png|webp)$/i).test(attachment.attachment_url[0]))
		.map(attachment => ({
			id: attachment.post_id[0],
			postId: attachment.post_parent[0],
			url: attachment.attachment_url[0]
		}));

	console.log(images.length + ' attached images found.');
	return images;
}
```
Now:
```js
function collectAttachedImages(channelData) {
    const attachments = getItemsOfType(channelData, 'attachment');

    // Ensure attachments are defined and an array
    if (!attachments || !Array.isArray(attachments)) {
        console.error('No attachments found or attachments is not an array');
        return [];
    }

    const images = attachments
        .filter(attachment => attachment.attachment_url && (/\.(gif|jpe?g|png|webp)$/i).test(attachment.attachment_url[0]))
        .map(attachment => {
            // Ensure necessary properties are defined
            const postId = attachment.post_id && attachment.post_id[0];
            const postParentId = attachment.post_parent && attachment.post_parent[0];
            const url = attachment.attachment_url && attachment.attachment_url[0];

            if (!postId || !postParentId || !url) {
                console.error('Attachment properties are not defined correctly', attachment);
                return null; // Skip the attachment
            }

            return {
                id: postId,
                postId: postParentId,
                url: url
            };
        })
        .filter(image => image !== null); // Filter out any null values

    console.log(images.length + ' attached images found.');
    return images;
}
```
That got me my markdown and my images. However, all of the images in the markdown files were looking like this:

```
![](https://images.squarespace-cdn.com/content/v1/6510e0308bfa4f011936cb7c/1695605266244-OSU4GK3QU4OTUXYD3WAL/DSCF6733.jpg)
```

And I needed them to look like this:

```
![](images/DSCF6733.jpg)
```

I am no regex expert, so I asked Chat GPT to write a find and replace like so: 
```
Search field: !\[\]\(https:\/\/[^)]+\/([^\/]+)\)
Replace field: ![](images/\1)
```
Well, it mostly worked. A few finishing touches and we're done.

~~1. Get the content out of Squarespace (text & images).~~
~~2.  Convert the content into markdown (text & images).~~
3. Build a site locally using hexo.
4. Deploy it somewhere.

# Build a site locally using hexo

Installed Node.js, Hexo, and after fighting with the syntax highlighting of the popular [Cactus theme](https://github.com/probberechts/hexo-theme-cactus), I gave up and went with [Oasis](https://github.com/qiantao94/hexo-theme-oasis). Easy to read and not much to take away from the content.

~~1. Get the content out of Squarespace (text & images).~~
~~2.  Convert the content into markdown (text & images).~~
~~3. Build a site locally using hexo.~~
4. Deploy it somewhere.

# Deploy it somewhere

Let's deploy it to GH.

I created a repo using my GH name and a yml file like so
```yml
name: Pages

on:
  push:
    branches:
      - master # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # If your repository depends on submodule, please see: https://github.com/actions/checkout
          submodules: recursive
      - name: Use Node.js 20
        uses: actions/setup-node@v4
        with:
          # Examples: 20, 18.19, >=16.20.2, lts/Iron, lts/Hydrogen, *, latest, current, node
          # Ref: https://github.com/actions/setup-node#supported-version-syntax
          node-version: "20"
      - name: Cache NPM dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

# Finished

All done. 

~~1. Get the content out of Squarespace (text & images).~~
~~2.  Convert the content into markdown (text & images).~~
~~3. Build a site locally using hexo.~~
~~4. Deploy it somewhere.~~