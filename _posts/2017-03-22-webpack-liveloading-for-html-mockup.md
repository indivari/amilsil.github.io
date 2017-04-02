---
layout: post
title: WebPack Liveloading For HTML Mockups
tags:
- javascript
- webpack
- html
---

 Have you had a design for a web to be converted to HTML? And you hand code your HTML? Then you know how easy it is to code with *live preview* on.

## Use Page Refreshes (old)

One easy hack to *live preview* is to use the *refresh* meta. This will refresh the page every 0 seconds - that's in immediate succession.

```html
<meta http-equiv='refresh' value='0'/>
```

However, this cannot *pre-process* your SASS or LESS or preserve the page scroll. It gets more tedius also if you want to test page interactions - for the page will refresh before your page interaction. 

A smarter way to do this is to use *webpack hot module reloading*. 

## WebPack HMR

How webpack HMR works is by injecting the browser with a snippet of code of an agent that can load deltas of changes from the server. The *webpack dev server* generates these deltas for you as you make the changes in the code.

Webpack, as you know, can also run CSS *preprocessors* and ES6 *transpiling*.

So what's required to get this working. 
You need at least one javascript file in your code, even if it's empty. Webpack should be used to transpile that javascript, ready to be sent to the browser.



