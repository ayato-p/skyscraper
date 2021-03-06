# Skyscraper

## Structural scraping

What is structural scraping? Think of [Enlive]. It allows you to parse arbitrary HTML and extract various bits of information out of it: subtrees or parts of subtrees determined by selectors. You can then convert this information to some other format, easier for machine consumption, or process it in whatever other way you wish. This is called _scraping_.

Now imagine that you have to parse a lot of HTML documents. They all come from the same site, so most of them are structured in the same way and can be scraped using the same sets of selectors. But not all of them. There’s an index page, which has a different layout and needs to be treated in its own peculiar way, with pagination and all. There are pages that group together individual pages in categories. And so on. Treating single pages is easy, but with whole collections of pages, you quickly find yourself writing a lot of boilerplate code.

In particular, you realize that you can’t just `wget -r` the whole thing and then parse each page in turn. Rather, you want to simulate the workflow of a user who tries to “click through” the website to obtain the information she’s interested in. Sites have tree-like structure, and you want to keep track of this structure as you traverse the site, and reflect it in your output. I call it “structural scraping”.

## A look at Skyscraper

This is where Skyscraper comes in. Skyscraper grew out of quite a few one-off attempts to create machine-readable, clean “dumps” of different websites. Skyscraper builds on [Enlive] to process single pages, but adds abstractions to facilitate easy processing of entire sites.

Skyscraper can cache pages that it has already downloaded, as well as data extracted from those pages. Thus, if scraping fails for whatever reason (broken connection, OOM error, etc.), Skyscraper doesn’t have to re-download every page it has already processed, and can pick up off wherever it had been interrupted. This also facilitates updating scraped information without having to re-download the entire site.

Skyscraper is work in progress. This means that anything can change at any time. All suggestions, comments, pull requests, wishlists, etc. are welcome.

 [Enlive]: http://cgrand.github.com/enlive

The current release is 0.1.1. To use Skyscraper in your project, add the following to the `dependencies` section in your `project.clj`:

```
[skyscraper "0.1.1"]
```

 [NEWS.md]: https://github.com/nathell/skyscraper/blob/master/NEWS.md

## Contexts

A “context” is a map from keywords to arbitrary data. Think of it as “everything we have scraped so far”. A context has two special keys, `:url` and `:processor`, that contains the next URL to visit and the processor to handle it with (see below).

Scraping works by transforming context to list of contexts. You can think of it as a list monad. The initial list of contexts is supplied by the user, and typically contains a single map with an URL and a root processor.

A typical function producing an initial list of contexts (a _seed_) looks like this:

```clojure
(defn seed [& _]
  [{:url "http://www.example.com",
    :processor :root-page}])
```

## Processors

A “processor” is a unit of scraping: a function that processes sets of HTML pages in a uniform way.

A processor also performs janitorial tasks like downloading pages, storing them in the HTML cache, combining URLs, and caching the output. Processors are defined with the `defprocessor` macro (which expands to an invocation of the more elaborate `processor` function). A typical processor, for a site’s landing page that contains links to other pages within table cells, might look like this:

```clojure
(defprocessor landing-page
  :cache-template "mysite/index"
  :process-fn (fn [res context]
                (for [a (select res [:td :a])]
                  {:page (text a),
                   :url (href a),
                   :processor :subpage})))
```

The most important part is `:process-fn`. This is the function called by the processor to extract new information from a page and include it in the context. It takes two parameters:

 1. an Enlive resource corresponding to the parsed HTML tree of the page being processed,
 2. the current context (i.e., combined outputs of all processors so far).

The output should be a seq of maps that each have a new URL and a new processor (specified as a keyword) to invoke next. If the output doesn’t contain the new URL and processor, it is considered a terminal node and Skyscraper will include it as part of the output. If the processor returns one context only, it is automatically wrapped in a seq.

## Caching

Skyscraper has two kinds of caches: one holding the raw downloaded HTML before parsing and processing (“HTML cache”), and one holding the results of parsing and processing individual pages (“processed cache”). Both caches are enabled by default, but can be disabled as needed.

It is recommended to keep the processed cache enabled at all times. The HTML cache can typically be disabled without many drawbacks, as Skyscraper will not attempt to redownload a page that had been processed already. The advantage of disabling HTML cache is saving disk space: Web pages are typically markup-heavy and the interesting pieces constitute a tiny part of them, so the HTML cache can grow much faster than the processed cache. This can be problematic when scraping huge sites. However, leaving both caches on can greatly speed up development.

Currently, both caches are backed by the filesystem and live in `~/skyscraper-data`, respectively under `cache/html` and `cache/processed`. In future versions, multiple cache backends will be added.

## Templating

Every page cached by Skyscraper is stored under a cache key (a string). It is up to the user to construct a key in a unique way for each page, based on information available in the context. Typically, the key is hierarchical, containing a logical “path” to the page separated by slashes (it may or may not correspond to the page’s URL).

To facilitate construction of such keys, Skyscraper provides a micro-templating framework. The key templates can be specified in a `cache-template` parameter of the `defprocessor` macro (see above). When a template contains parts prefixed by a colon and containing lower-case characters and dashes, these are replaced by corresponding context elements. As an example, the template `"mysite/:surname/:name"`, given the context `{:name "John", :surname "Doe"}`, will generate the key `"mysite/Doe/John"`.

## Invocation and output

Skyscraper is invoked as follows:

```clojure
(scrape seed params)
```

where `seed` is an initial seq of contexts (typically generated by a separate function), and `params` are optional keyword parameters. The parameters currently supported are:

 - `:processed-cache` (default `true`) – Enable or disable the processed cache.
 - `:http-options` – A map of additional options to pass to [clj-http], which Skyscraper uses under the hood to download pages from the Web. These options override the defaults, which are `{:as :auto, :socket-timeout 5000, :decode-body-headers true}` (set timeout to 5 seconds and autodetect the encoding). See clj-http’s documentation for other options you can pass here.

 [clj-http]: https://github.com/dakrone/clj-http

The output is a lazy seq of leaf contexts. This means that each map in this seq contains keys from every context that led to it being generated. For example, given the following scenario:

 - The first processor called by Skyscraper returns `[{:a 1, :url "someurl", :processor :processor2}]`
 - The second processor (`processor2`) returns `[{:b 2, :url "nexturl", :processor :processor3}]`
 - The third and final processor returns `[{:c 3} {:c 4}]`

Then the final output will be `({:a 1, :b 2, :c 3} {:a 1, :b 2, :c 4})`.

## Examples

More elaborate examples can be found in the `examples/` directory of the repo.

## Caveats

Skyscraper is work in progress. Some things are missing. The API is still in flux. Function and macro names, input and output formats are liable to change at any time. Suggestions of improvements are welcome (preferably as GitHub issues), as are pull requests.

## License

Copyright (C) 2015 Daniel Janus, http://danieljanus.pl

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
