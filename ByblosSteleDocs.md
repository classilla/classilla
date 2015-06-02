**Byblos was introduced in Classilla 9.3.0. This document does not apply to 9.2.3 or previous versions of Classilla.**

[Byblos](http://en.wikipedia.org/wiki/Byblos), in modern Lebanon, is believed by many to be the oldest continuously-inhabited city in the world. The [Byblos syllabary](http://en.wikipedia.org/wiki/Byblos_syllabary) is one of the most famous undeciphered writing systems, found on a few bronze tablets and "spatula"-like objects, and fragments of a monument [stele](http://en.wikipedia.org/wiki/Stele).

Like the famous [Rosetta Stone](http://en.wikipedia.org/wiki/Rosetta_Stone) stele which enabled the translation of Egyptian hieroglyphs from ancient Greek and Demotic scripts (and Apple's Rosetta, allowing superior older PowerPC code to run on newer inferior x86 processors), the Byblos steles you can create for Classilla can allow modern web pages to be rewritten dynamically for Classilla's older layout engine and those of us continuously inhabiting Mac OS 9.

## How Byblos operates ##

Byblos is similar to Greasemonkey in theory, but implementationally quite different. [Greasemonkey](http://en.wikipedia.org/wiki/Greasemonkey) is a tool allowing users of Firefox and selected other browsers to dynamically rewrite the layout of a page in JavaScript by manipulating the Document Object Model, which is the abstract representation of the page after it has been loaded by the browser. Tweaking the DOM works fine if the browser core can handle sophisticated runtime manipulations and can do so in a determinative fashion even if the rendering is wrong. On an older core such as Classilla, however, this may lead to subtle bugs or unexpected results, and the DOM parser may not be able to handle certain complex constructs.

So Byblos works at a lower level than Greasemonkey and allows you to dynamically rewrite the page's HTML _even before the browser tries to process it._ The process is very simple: Byblos will check to see if a stele exists for the domain it is accessing (HTTP or HTTPS). If there is a stele and it can be processed, and the stele agrees to process this URL, Byblos downloads the entire page, gives it to the stele, the stele manipulates it, and the HTML is presented to the browser transparently.

Stelae are dynamic and bulletproof. You can modify them while the browser is running; you can disable them simply by moving them out of the Byblos folder (in the Classilla folder) as the browser is running; and even if you make a mistake or syntax error, the browser continues to function (the stele just doesn't work). This makes it easy to fine-tune your code on the fly. They are written in JavaScript and use a simple API, allowing anyone with a basic knowledge of the language to write one with just a text editor (we recommend BBEdit 6).

In 9.3.2 and later versions of Classilla, you can also write stelae to rewrite style sheets.

Stelae do _not_ translate images, plain text, HTML files on disk, Gopher menus or FTP directory listings. Stelae also do not translate data received through `XMLHttpRequest`s, even if that data is HTML, due to technical constraints.

## Writing your first stele ##

To make the most of Byblos, you need to know basic HTML and JavaScript, tutorials of which are beyond the scope of this document. However, let's look at a simple example.

```
function() {
        return {
                wantURI : function(uri) {
                        return true;
                },
                parseHTML : function(text) {
                        var answer;

                        // Replace all occurrences of google, regardless
                        // of case, with g00G13.
                        answer = text.replace(/google/mi, "g00G13");

                        return {
                                response : "ok",
                                body     : answer
                        };
                }
        };
};
```

Name this stele `www.google.com.js`. If you are using 9.3.3 or later, make a folder in your `Documents` folder called `Byblos` and save it there; otherwise, place it in the `Byblos` folder with your Classilla executable. You do not need to restart Classilla for this to take effect.

Open the JavaScript Console (under `Web Development`) to watch the process, then go to `www.google.com` (you may need to reload the page if it is cached). You will now see that all occurrences of `google` in the page text, even the title, have been immaturely replaced with `g00G13`. This makes the page quite useless, but serves as a demonstration. If you had the Console window open, you will see that Byblos logged the operations it took. If your stele had a syntax or other error in it, Byblos would log it to the console as well.

All stelae consist of a single JavaScript `function()`. This function must return an object with at least two slots: the first, `wantURI`, being a function that accepts an `nsIURI` object and returns `true` or `false` depending on whether the stele wishes to parse this actual URI, and the second, `parseHTML`, also being a function that accepts a string representing the document HTML and returns itself an object of two slots, one being the `response` ("`ok`" or otherwise) and the other being the `body`, which is the HTML after it has been rewritten.

In our case, this stele returns true for any URI it gets (it will only be presented with URIs that point to `www.google.com`), and in `parseHTML`, takes the raw HTML given to it by Byblos and does a simple string replace, and returns that in the response object.

By now, people unfamiliar with Mozilla will wonder what an `nsIURI` is, and will already have visited MDN's [nsIURI Documentation](https://developer.mozilla.org/en/nsIURI). In doing so, you will have executed a stele that comes by default with Classilla which cleans up the MDN pages somewhat. Here is how it appears in 9.3.0:

```

/* Clean up the header for developer.mozilla.org.
   Cameron Kaiser
   Public domain. */


function() {
        return {
                wantURI : function(uri) {
                        if (uri.path == "" || uri.path == "/" || uri.path == "/en-US/")
                                return false; // We can't do much with the main page.
                        return true;
                },
                parseHTML : function(text) {
                        var answer;

                        // Remove the top header.
                        answer = text.replace(/<\/?header[^>]+>/mig, "");
                        // Remove the mozilla-tab.
                        answer = answer.replace(/<a id="moz-tab"/, "<a");
                        // Remove the stupid JavaScript whine.
```
```
                        answer = answer.replace(/<div class="noscript">.+<\/div><\/noscript>/, "</noscript>");
```
```
                        // Remove the styling from the h1 title.
                        answer = answer.replace(/<h1 id="logo">/,
                                "<h1 style=\"color: #404040\">");
                        // Remove the CSS rollovers; they don't work. [\s\S]+? matches all characters,
                        // including newlines; +? is a non-greedy match a la Perl.
                        answer = answer.replace(/<nav id="nav"[\s\S]+?<\/nav>/m, "");

                        return {
                                response : "ok",
                                body     : answer
                        };
                }
        };
};

```

This example actually is not much more complicated. It checks the path of the URI to see if it's something it can help with. The MDN main page has lots of links and widgets which Classilla can't render right even with help (it lacks those facilities), so there is no point in taking the Byblos performance hit there (more about that in the next section) and it returns `false` (prompting Byblos to simply fall through to the default handlers). In the HTML rewriting portion, it's the same basic idea as the `g00G13` example except we're removing portions of the page and entirely rewriting others.

In fact, you could even just take a piece of the page into a temporary variable and write out a new page with just that piece in it, such as for a video player site. The piece would just be the video link, and chaff such as snarky comments and ads would simply be eliminated because you would never pass them back to the browser. **Nothing says that what you spit out has to be anything like what you took in.** The key idea for Byblos is functionality, not rendering fidelity.

## Rewriting CSS (9.3.2 and later) ##

In Classilla 9.3.2, Byblos can also be used to rewrite style sheets. Note that this only works for standalone style sheets; inline styles and inline style sheets are still done in the HTML stelae.

The idea is the same, except your two-slot return object should have functions for `wantURI()` and `parseCSS()` (instead of `wantURI()` and `parseHTML()`). Otherwise, the convention is the same.

To indicate that the stelae is for CSS and not HTML, you place it in the `" CSS"` (note leading space) folder inside the `Byblos` folder.

A handy trivial situation is to return no CSS at all. This would be useful on sites where you want to completely, or mostly, destyle them automatically. Just return a response of `"ok"` as usual, but a body that is an empty string (`""`). You can make this faster with complete replacement in 9.3.3 (see below).

## Controlling domains and subdomains (9.3.2 and later) ##

Occasionally you want to handle a site in general one way, but certain subdomains another. Say that you want `mail.foobar.invalid` handled specifically, but everything else on `foobar.invalid` should be handled another way, such as `maps.foobar.invalid` and `beer.foobar.invalid`.

As usual, you will name your stele for `mail.foobar.invalid` as `mail.foobar.invalid.js`. But for the fallback, name it `ANY..foobar.invalid.js` (notice the double period). This will be then applied to any domain name `*.foobar.invalid` _after_ a search for that specific domain has failed.

## Complete page replacement (9.3.3 and later) ##

If you know that you will not need the page content because the content you will be replacing it with doesn't require it, such as redirect pages, you can speed up your stele considerably by telling Classilla so. This is done with the `fullReplace` function. It is the same as `wantURI`, except that you return `true` if you don't need the actual page (i.e., you're fully replacing it with totally separate content), and `false` otherwise. If you return `true`, your `parseHTML` function is still called, but with an empty string. Here's an example that redirects a site's front page to ours at Floodgap:

```

function() {
	return {
		wantURI : function(uri) {
			if (uri.path == "/" || uri.path == "")
				return true;
			return false;
		},
		fullReplace : function(uri) {
			if (uri.path == "/" || uri.path == "")
				return true;
			return false;
		},
		parseHTML : function(text) {
			return {
				response : "ok",
				body     : '<meta http-equiv="refresh" content="0; url=http://www.floodgap.com/">'
			};
		}
	};
};

```

The `<META>` tag in the body we return immediately redirects us. Because we said we don't need the original page to do the work, Classilla can save considerable time by completely avoiding preprocessing altogether, and on a fast system the redirection will be nearly instantaneous.

`fullReplace()` is never called if `wantURI()` returns `false` for a given URI, and if you do not implement a `fullReplace()` function, it is assumed to be `false` for all URIs to be compatible with stelae from earlier versions of Classilla.

## Performance considerations ##

Because stelae are written in JavaScript, they have the advantage of being dynamic but the disadvantage of being interpreted (and because they are dynamic, are interpreted each time they are invoked).

Furthermore, if the stele indicates to Byblos that it can translate a URI, then Byblos must download the entire HTML of the page first before it can hand it to the stele (for obvious reasons), which means that Classilla's HTML parser cannot be working on the page while it downloads.

You should therefore make sure that your stele is worth the trouble: it should either restore functionality that is not possible any other way, or it should remove portions of the page that are difficult or slow for Classilla to render ordinarily. In the latter case, even with Byblos' performance impact, a stele can boost the browser's overall performance. If you are using 9.3.3 or later, and you know you won't need the page itself to rewrite it, then use the `fullReplace()` API to further reduce the amount of overhead required.

When you are unsure if you can help in a situation (such as the MDN stele above), you should return `false` from `wantURI()`. Even though the cost of interpreting the script is paid, Classilla can still render HTML as it comes down the pipe instead of waiting for all of it and passing it to a stele which can't do anything useful with it anyway, which at least doesn't add insult to injury in the end.

## Limitations ##

When writing your stelae, be aware of the following limitations as well as the performance concerns above:

  1. If you are using Classilla 9.3.3 or later, stelae in `Documents:Byblos:` will take precedence over the built-in stelae. Hope yours are better!
  1. Remember, stelae do not translate XHR or AJAX. Data that comes this way has to be dealt with in some other manner (such as suppressing the script).
  1. Your stele should **only** consist of the single function body. Do not attempt to store persistent data outside of it, or assume that the function is persistent; it can be called any time or never, and the object may be destroyed promptly to conserve memory. That said, the object your function returns _may_ contain multiple slots such as library functions that your routines call (such as `this.libraryFunction`). These are okay, because they will be cleaned up when your object is destructed. However, they, too, should not be counted on to contain persistent data.
  1. **Your stele is privileged code -- it runs with chrome privileges.** This means that if you use your stele to execute untrusted data, you could yourself open a security hole in the browser. Security in Classilla is less sophisticated than later Mozilla releases. Do not try to run JavaScript you receive in the HTML; not only will it probably not work, it may do things to the browser you did not intend. Byblos' error checking will catch innocent mistakes, but it does not trap all attempts at stupidity.
  1. Your stele _may_ attempt to instantiate scriptable XPCOM objects, and this will generally work, but you should be prepared to do your own error handling if they throw and you should clean up these instances before exiting your function or returning a value. However, stelae should not be used as replacements for standard Classilla extensions.

## We want your work ##

Like the Greasemonkey community, we would like to build up a library of useful stelae for sites Classilla can't render properly. Scripts that are judged particularly essential or useful may even be included with future versions of Classilla.

If you have a stele you would like to submit for review, please open an issue and attach it. Don't take it personally if we decline; we're trying to restrict this to useful ones, in our sole purview. But others might like it through unofficial means, and the great thing about Byblos is there is little penalty for experimentation.