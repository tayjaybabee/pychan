# pychan

>Update 2025-05-25: this project has been archived and will no longer be maintained, as 4chan
>recently made changes to their firewall infrastructure (in response to the attacks in April 2025)
>to make it more difficult to scrape its content.

1. [Overview](#overview)
2. [Installation](#installation)
3. [Usage](#usage)
   1. [General Notes](#general-notes)
   2. [Setup](#setup)
   3. [Fetch Board Names](#fetch-board-names)
   4. [Fetch Threads](#fetch-threads)
   5. [Fetch Archived Threads](#fetch-archived-threads)
   6. [Search 4chan](#search-4chan)
      1. [About Cloudflare](#about-cloudflare)
      2. [Search 4chan Code Example](#search-4chan-code-example)
   7. [Fetch Posts for a Specific Thread](#fetch-posts-for-a-specific-thread)
4. [pychan Models](#pychan-models)
   1. [Threads](#threads)
   2. [Posts](#posts)
      1. [A Note About Replies](#a-note-about-replies)
   3. [Posters](#posters)
   4. [Files](#files)
5. [Contributing](#contributing)

## Overview

`pychan` is a Python client for interacting with 4chan. 4chan does not have an official API, and
attempts to implement one by third parties have tended to languish, so instead, this library
provides abstractions over interacting with (scraping) 4chan directly. `pychan` is object-oriented
and its implementation is lazy where reasonable (using Python Generators) in order to optimize
performance and minimize superfluous blocking I/O operations.

## Installation

If you have Python >=3.10 and <4.0 installed, `pychan` can be installed from PyPI using
something like

```bash
pip install pychan
```

## Usage

### General Notes

All 4chan interactions are throttled internally by sleeping the executing thread. If you execute
`pychan` in a multithreaded way, you will not get the benefits of this throttling. `pychan` does not
take responsibility for the consequences of excessive HTTP requests in such cases.

### Setup

```python
from pychan import FourChan, LogLevel, PychanLogger

# With all defaults (logging disabled, all exceptions raised)
fourchan = FourChan()

# Tell pychan to gracefully ignore HTTP exceptions, if any, within its internal logic
fourchan = FourChan(raise_http_exceptions=False)

# Tell pychan to gracefully ignore parsing exceptions, if any, within its internal logic
fourchan = FourChan(raise_parsing_exceptions=False)

# Configure logging explicitly
logger = PychanLogger(LogLevel.INFO)
fourchan = FourChan(logger=logger)

# Use all of the above settings at once
logger = PychanLogger(LogLevel.INFO)
fourchan = FourChan(logger=logger, raise_http_exceptions=True, raise_parsing_exceptions=True)
```

The rest of the examples in this `README` assume that you have already created an instance of the
`FourChan` class as shown above.

### Fetch Board Names

This function dynamically fetches boards from 4chan at call time.

>Note: boards which are not compatible with `pychan` are not returned in this list.

```python
boards = fourchan.get_boards()
# Sample return value:
# ['a', 'b', 'c', 'd', 'e', 'g', 'gif', 'h', 'hr', 'k', 'm', 'o', 'p', 'r', 's', 't', 'u', 'v', 'vg', 'vm', 'vmg', 'vr', 'vrpg', 'vst', 'w', 'wg', 'i', 'ic', 'r9k', 's4s', 'vip', 'qa', 'cm', 'hm', 'lgbt', 'y', '3', 'aco', 'adv', 'an', 'bant', 'biz', 'cgl', 'ck', 'co', 'diy', 'fa', 'fit', 'gd', 'hc', 'his', 'int', 'jp', 'lit', 'mlp', 'mu', 'n', 'news', 'out', 'po', 'pol', 'pw', 'qst', 'sci', 'soc', 'sp', 'tg', 'toy', 'trv', 'tv', 'vp', 'vt', 'wsg', 'wsr', 'x', 'xs']
```

### Fetch Threads

```python
# Iterate over all threads in /b/
for thread in fourchan.get_threads("b"):
    # Do stuff with the thread
    print(thread.title)
    # You can also iterate over all the posts in the thread
    for post in fourchan.get_posts(thread):
        # Do stuff with the post - refer to the model documentation in pychan's README for details
        print(post.text)
```

### Fetch Archived Threads

>Note: some boards do not have an archive (e.g. `/b/`). Such boards will either return an empty list
>or raise an exception depending on how you have configured your `FourChan` instance.

The threads returned by this function will always have a `title` field containing the text shown in
4chan's interface under the "Excerpt" column header. This text can be either the thread's real title
or a preview of the original post's text. Passing any of the threads returned by this method to the
`get_posts()` method will automatically correct the `title` field (if necessary) on the thread that
gets attached to the returned posts. See
[Fetch Posts for a Specific Thread](#fetch-posts-for-a-specific-thread) for more details.

>Technically, `pychan` could address the `title` behavior described above by issuing an additional
>HTTP request for each thread to get its real title, but in the spirit of making the smallest number
>of HTTP requests possible, `pychan` directly uses the excerpt instead.

```python
for thread in fourchan.get_archived_threads("pol"):
    # Do stuff with the thread
    print(thread.title)
    # You can also iterate over all the posts in the thread
    for post in fourchan.get_posts(thread):
        # Do stuff with the post - refer to the model documentation in pychan's README for details
        print(post.text)
```

### Search 4chan

#### About Cloudflare

Performing searches against 4chan is much more cumbersome than accessing the rest of 4chan's data.
This is because 4chan has a Cloudflare firewall in front of its REST API, so the only way to get
data back from searches is to supply the HTTP request information needed to bypass Cloudflare's
anti-bot checks. Ultimately, this amounts to passing certain headers along with the HTTP request,
but the challenge comes from actually acquiring such headers.

It is currently beyond the scope of `pychan` to generate these headers for you, so if you would like
to automate the circumvention of Cloudflare's protections, you may want to look into using a project
like one of the following (this list is alphabetized and not exhaustive):

* [ultrafunkamsterdam/undetected-chromedriver](https://github.com/ultrafunkamsterdam/undetected-chromedriver)
* [VeNoMouS/cloudscraper](https://github.com/VeNoMouS/cloudscraper)
* [wkeeling/selenium-wire](https://github.com/wkeeling/selenium-wire)

A manual way to acquire these values is to perform a 4chan search using a web browser and leverage
the browser's Developer Tools to trace the network requests that were made during the search. The
request that contains the Cloudflare values will have been made to `https://find.4chan.org/api` with
some query parameters. Once you have found this request, copy the `User-Agent` and `Cookie` values
that were sent in your request, then pass them to `pychan`'s `search()` method. Be aware that the
Cloudflare cookie(s) have an expiration on them, so this manual workaround will only return results
until Cloudflare invalidates your cookie(s). After that, you will need to acquire new values.

#### Search 4chan Code Example

>Note: closed/stickied/archived threads are never returned in search results.

```python
# This "threads" variable will contain a Python Generator (not a list) in order to facilitate laziness
threads = fourchan.search(
   board="b",
   text="ylyl",
   user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36",
   cloudflare_cookies={
      "cf_clearance": "bm2RICpcDeR4cXoC2nfI_cnZcbAkN4UYpN6c1zzeb8g-1440859602-0-160"
   }
)
for thread in threads:
    # The thread object is the same class as the one returned by get_threads()
    for post in fourchan.get_posts(thread):
       # Do stuff with the post - refer to the model documentation in pychan's README for details
       print(post.text)
```

### Fetch Posts for a Specific Thread

```python
from pychan.models import Thread

# Instantiate a Thread instance with which to query for posts
thread = Thread("int", 168484869)

# Note: the thread contained within the returned posts will have all applicable metadata (such as
# title and sticky status), regardless of whether you provided such data above - pychan will
# "auto-discover" all metadata and include it in the post models' copy of the thread
posts = fourchan.get_posts(thread)
```

## pychan Models

The following tables summarize all the kinds of data that are available on the various models used
by this library.

Also note that all model classes in `pychan` implement the following methods:

* `__repr__`
* `__str__`
* `__hash__`
* `__eq__`
* `__iter__` - this is implemented so that the models may be passed to Python's `tuple()` function
* `__copy__`
* `__deepcopy__`

### Threads

The table below corresponds to the `pychan.models.Thread` class.

| Field | Type | Example Value(s) |
| ----- | ---- | ---------------- |
| `thread.board` | `str` | `"b"`, `"int"`
| `thread.number` | `int` | `882774935`, `168484869`
| `thread.title` | `Optional[str]` | `None`, `"YLYL thread"`
| `thread.is_stickied` | `bool` | `True`, `False`
| `thread.is_closed` | `bool` | `True`, `False`
| `thread.is_archived` | `bool` | `True`, `False`
| `thread.url` | `str` | `"https://boards.4chan.org/a/thread/251097344"`

### Posts

The table below corresponds to the `pychan.models.Post` class.

| Field | Type | Example Value(s) |
| ----- | ---- | ---------------- |
| `post.thread` | `Thread` | `pychan.models.Thread`
| `post.number` | `int` | `882774935`, `882774974`
| `post.timestamp` | [datetime.datetime](https://docs.python.org/3/library/datetime.html#datetime.datetime) | [datetime.datetime](https://docs.python.org/3/library/datetime.html#datetime.datetime)
| `post.poster` | `Poster` | `pychan.models.Poster`
| `post.text` | `str` | `">be me\n>be bored\n>write pychan\n>somehow it works"`
| `post.is_original_post` | `bool` | `True`, `False`
| `post.file` | `Optional[File]` | `None`, `pychan.models.File`
| `post.replies` | `list[Post]` | `[]`, `[pychan.models.Post, pychan.models.Post]`
| `post.url` | `str` | `"https://boards.4chan.org/a/thread/251097344#p251097419"`

#### A Note About Replies

The `replies` field shown above is purely a convenience feature `pychan` provides for accessing all
posts within a thread which used the `>>` operator to "reply" to the current post. However, it is
not necessary to use the `replies` field to access all available posts in a thread; when you call
the `get_posts()` method, you will still receive all the posts (in the order they were posted) as a
single, flat list.

### Posters

The table below corresponds to the `pychan.models.Poster` class.

| Field | Type | Example Value(s) |
| ----- | ---- | ---------------- |
| `poster.name` | `str` | `"Anonymous"`
| `poster.is_moderator` | `bool` | `True`, `False`
| `poster.id` | `Optional[str]` | `None`, `"BYagKQXI"`
| `poster.flag` | `Optional[str]` | `None`, `"United States"`, `"Canada"`

### Files

The table below corresponds to the `pychan.models.File` class.

| Field | Type | Example Value(s) |
| ----- | ---- | ---------------- |
| `file.url` | `str` | `"https://i.4cdn.org/pol/1658892700380132.jpg"`
| `file.name` | `str` | `"wojak.jpg"`, `"i feel alone.jpg"`
| `file.size` | `str` | `"601 KB"`
| `file.dimensions` | `tuple[int, int]` | `(1920, 1080)`, `(800, 600)`
| `file.is_spoiler` | `bool` | `True`, `False`

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for developer-oriented information.
