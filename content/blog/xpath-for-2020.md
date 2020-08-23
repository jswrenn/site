+++
title = "Javascript Generators, Meet XPath"
date = 2020-06-29
tags = ["javascript"]
+++

Using Generators to Modernize a Geriatric Javascript API for `$CURRENT_YEAR`

<!-- more -->
---

How do you find-and-replace text on an HTML page?
```html
<div>Hello, <span>human</span>!</div>
```
If the text is neatly neatly isolated inside an HTML element, it's easy; this will do:
```javascript
document.querySelector("span").textContent = "evolved ape";
```
**But here's a puzzle**: how do you you change text that *isn't* neatly isolated in an HTML element?

You *could* use `innerHTML`:
```javascript
let elt = document.querySelector("div");
elt.innerHTML = elt.innerHTML.replace("Hello", "Greetings");
```
...but this will hose any event listeners registered on `elt`'s children.


You *could* grapple onto the nearest selectable element:
```javascript
let node = document.querySelector("div").childNodes[0];
node.textContent = node.textContent.replace("Hello", "Greetings");
```
Yuck; this sort of child-node indexing feels *really* brittle.

**Why can't we just *directly* select the text nodes containing `Hello`?**

## XPath
**We can!** Enter: [XPath](https://en.wikipedia.org/wiki/XPath), the *excessively* powerful language for querying XML documents. It's usable in web-browsers with the, uh, *descriptively*-named method [`document.evaluate`](https://developer.mozilla.org/en-US/docs/Web/API/Document/evaluate).

It's a *bit* of a production to use:
```javascript
let xpath = "//text()[contains(., 'Hello')]"; // find text nodes containing 'Hello'
let context = document.body; // look in the body element
let namespace_resolver = null; // some sorta xml voodoo
let result_type = XPathResult.ORDERED_NODE_SNAPSHOT_TYPE; // DEFINITELY MAKE SURE YOU USE THIS

let results = document.evaluate(xpath, context, null, result_type);

for (let i = 0; i < results.snapshotLength; i++) {
  let node = results.snapshotItem(i);
  node.textContent = node.textContent.replace("Hello", "Greetings");
}
```
Yes, you really need to write *all* of that. The `result_type` argument is technically optional, but omit it at your own peril: without it, you must instead stream results via `iterateNext`, and this will crash with an exception if you dare *modify* the queried elements!

It's no wonder `document.evaluate` is seldom used. **Can we improve on it?**

## Iterizing XPath Queries
Yes, with [**generators**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators)! We can exploit the implicit iterability of generators to modernize this unwieldy API:
```javascript
Document.prototype.xpath = Element.prototype.xpath =
  function* xpath(xpath) {
    let context = this instanceof Document ? document.documentElement : this;
    let namespace_resolver = null;
    let result_type = XPathResult.ORDERED_NODE_SNAPSHOT_TYPE;
    let results = document.evaluate(xpath, context, null, result_type);

    for (let i = 0; i < results.snapshotLength; i++)
      yield results.snapshotItem(i);
  };
```
And because the result of this function is iterable, we can use it with [spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax):
```javascript
[...document.xpath("//text()[contains(., 'Hello')]")].
  forEach(node => node.textContent = node.textContent.replace("Hello", "Greetings"));
```