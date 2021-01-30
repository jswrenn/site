+++
title = "Typesetting a Resume with HTML and CSS"
date = 2021-01-29
tags = ["browser automation", "typesetting", "html", "css"]
+++

With my PhD drawing to a close, I'm on the job market! I'll need a slick resume ASAP, but how to typeset it?
<dl>
  <dt>Microsoft Word?</dt>
  <dd>...don't have it.</dd>
  <dt>LaTeX?</dt>
  <dd>...don't have the patience for it.</dd>
  <dt>HTML + CSS?</dt>
  <dd>...now <em>that</em> could work!</dd>
<dl>

<!-- more -->

I figured I would merely whip up a document of the appropriate size:
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <style>
      html, body {
        margin: 0;
        padding: 0;
      }

      .page {
        width: 8in;
        height: 8in;
        padding: 0.5in;
        outline: 1px black solid;
      }
    </style>
  </head>
  <body>
    <div class="page">
      Hire me, please!
    </div>
  </body>
</html>
```
...maybe grab some [Google Fonts](https://fonts.google.com/) (*way* easier than managing system fonts, or fiddling with LaTeX!), open up DevTools, and bust out a sweet-looking resume. Then, I'd simply use the browser dialog to print to PDF (taking care to set the print margins to 0).

Oh, if it were only that simple.

## Obstacles

### CSS Units ≠ Physical Units
After hours of writing and styling, I produced what I thought was the *perfect* resume, tuned to fit into a single page. Then, I went to print-to-PDF. Disaster unfolded before my eyes: the fonts and spacing were all wrong, and my carefully-constructed columns of prose spilled awkwardly onto a second page. Why?

A CSS inch (as will all "physical" CSS units) has no relation to a physical inch. Rather, it's defined to be exactly 96 CSS pixels. (And, yes, that's 96 *CSS* pixels, which aren't necessarily equal to device pixels.)

At first, I feared that I would need to figure out the mapping between CSS pixels, dots, and inches. [Fortunately, most web browsers *do* map CSS inches to physical inches when printing.](https://hacks.mozilla.org/2013/09/css-length-explained/#footnote_ref4:~:text=As%20a%20side%20note%2C%20if%20the,CSS%20inch%20to%20the%20physical%20inch.) So, all I needed to do was design for what appeared in Chrome's print preview, *not* what appeared in the browser tab.

### No DevTools, Quick Reload with Print Preview
It immediately become apparent that DevTools cannot be used while the print dialog is open. Instead, I needed to CTRL+R-and-CTRL+P after *every* edit-and-save of my resume. This got old *fast*.

**I decided to grin and bear it.** How much longer could this possibly take?

### Font Inconsistencies
*"Huh, I could have sworn I had made that text bold."*

I had.

Chrome's print-to-PDF respected the font-weights I had set for my [Libre Baskerville](https://fonts.google.com/specimen/Libre+Baskerville) headings, but *not* the weights I had set on my [Raleway](https://fonts.google.com/specimen/Raleway) prose. The issue wasn't limited to bold spans; *all* of the body text was *far* too light!

A quick Google search revealed no leads, and the looming specter of LaTeX drew nearer...

## Solution
I recalled that there's *another* way to get PDFs out of Google Chrome: [Puppeteer](https://pptr.dev/#?product=Puppeteer&version=v5.5.0&show=api-pagepdfoptions), a library for programatically controlling the browser. I had recently used this API to generate vector screenshots of a browser-based IDE for [a research paper](/publications/Wrenn,%20Krishnamurthi%20-%20Will%20Students%20Write%20Tests%20Early%20Without%20Coercion.pdf), and I hadn't noticed any font issues then.

**It worked.** No font issues on my resume!

So, I decided to take another fifteen minutes to fix my other gripe: no live preview. I spent another fifteen minutes building a reusable utility that would:
  * Consume two command-line arguments: the path to an HTML input file, and a path to save the outputted PDF to.
  * Watch the input file for changes.
  * Re-render the page as a PDF to the output path upon each change.

The script is invoked like this:
```bash
$ node resume-screenshot.js ~/documents/resume.html /tmp/resume.pdf
```

I then opened [zathura](https://pwmt.org/projects/zathura/), a PDF viewer that reloads the viewed file each time it's changed, and got back to hacking on my resume! I arranged my HTML editor and zathura side-by-side, and each edit-and-save I make to my resume is nearly-instantly reflected in the zathura.

Here's the utility, in all of its glory:
```javascript
const chokidar = require('chokidar'); // ^3.5.1
const puppeteer = require('puppeteer-core'); // ^3.1.0

let [input, output] = process.argv.slice(2);

(async () => { try {
  let browser = await puppeteer.launch({
    headless: true,
    executablePath: '/opt/google/chrome/chrome', /* CHANGE ME */
  });

  const page = await browser.newPage();

  async function render_pdf(input) {
    await page.goto(`file://${input}`, { waitUntil: 'networkidle0' });
    await page.pdf({
      printBackground: true,
      width: '8.5in',
      height: '11in',
      path: output,
    });
    console.log("saved:", output);
  }

  chokidar.watch(input)
    .on('ready', () => console.log("watching", input))
    .on('add', render_pdf).on('change', render_pdf)
    .on('unlink', (path) => process.exit(1))

} catch(e) {
  console.error("Error:");
  console.error(e);
  process.exit(1);
}})();
```

## Other Tips
* Use a [CSS reset stylesheet](https://meyerweb.com/eric/tools/css/reset/), so you start with a blank slate.
* Use a [major second typescale](https://type-scale.com/).
* Use fonts from [Google Fonts](https://fonts.google.com/), not system fonts, to ensure consistency across machines.
* Use the [`@page` rule](https://developer.mozilla.org/en-US/docs/Web/CSS/@page) to set margins.

## Result
I have no doubt that a LaTeX or InDesign expert could produce a nicer looking document. The line-breaking employed by web browsers is poorer than that of LaTeX, and advanced microtypography is not readily available. Nonetheless, I am *extremely* pleased with the overall design of my resume. The ease of using HTML and CSS allowed me to attempt a much more ambitious design than I would have attempted in LaTeX.

I've uploaded the HTML of my (in-progress) resume [here](/resume.html), and the resulting PDF [here](/resume.pdf):
<object height="500px" style="width:100%" type="application/pdf" data="/resume.pdf?#zoom=85&scrollbar=0&toolbar=0&navpanes=0">
</object>

