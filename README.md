# Morss - Get full-text RSS feeds

_GNU AGPLv3 code_

Upstream source code: https://git.pictuga.com/pictuga/morss  
Github mirror (for Issues & Pull requests): https://github.com/pictuga/morss  
Homepage: https://morss.it/

This tool's goal is to get full-text RSS feeds out of striped RSS feeds,
commonly available on internet. Indeed most newspapers only make a small
description available to users in their rss feeds, which makes the RSS feed
rather useless. So this tool intends to fix that problem.

This tool opens the links from the rss feed, then downloads the full article
from the newspaper website and puts it back in the rss feed.

Morss also provides additional features, such as: .csv and json export, extended
control over output. A strength of morss is its ability to deal with broken
feeds, and to replace tracking links with direct links to the actual content.

Morss can also generate feeds from html and json files (see `feedify.py`), which
for instance makes it possible to get feeds for Facebook or Twitter, using
hand-written rules (ie. there's no automatic detection of links to build feeds).
Please mind that feeds based on html files may stop working unexpectedly, due to
html structure changes on the target website.

Additionally morss can detect rss feeds in html pages' `<meta>`.

You can use this program online for free at **[morss.it](https://morss.it/)**.

Some features of morss:
- Read RSS/Atom feeds
- Create RSS feeds from json/html pages
- Export feeds as RSS/JSON/CSV/HTML
- Fetch full-text content of feed items
- Follow 301/meta redirects
- Recover xml feeds with corrupt encoding
- Supports gzip-compressed http content
- HTTP caching with 3 different backends (in-memory/sqlite/mysql)
- Works as server/cli tool
- Deobfuscate various tracking links

## Dependencies

You do need:

- [python](http://www.python.org/) >= 2.6 (python 3 is supported)
- [lxml](http://lxml.de/) for xml parsing
- [bs4](https://pypi.org/project/bs4/) for badly-formatted html pages
- [dateutil](http://labix.org/python-dateutil) to parse feed dates
- [chardet](https://pypi.python.org/pypi/chardet)
- [six](https://pypi.python.org/pypi/six), a dependency of chardet
- pymysql

Simplest way to get these:

```shell
pip install git+https://git.pictuga.com/pictuga/morss.git@master
```

The dependency `lxml` is fairly long to install (especially on Raspberry Pi, as
C code needs to be compiled). If possible on your distribution, try installing
it with the system package manager.

You may also need:

- Apache, with python-cgi support, to run on a server
- a fast internet connection

## Arguments

morss accepts some arguments, to lightly alter the output of morss. Arguments
may need to have a value (usually a string or a number). In the different "Use
cases" below is detailed how to pass those arguments to morss.

The arguments are:

- Change what morss does
	- `json`: output as JSON
	- `html`: outpout as HTML
	- `csv`: outpout as CSV
	- `proxy`: doesn't fill the articles
	- `clip`: stick the full article content under the original feed content (useful for twitter)
	- `search=STRING`: does a basic case-sensitive search in the feed
- Advanced
	- `csv`: export to csv
	- `indent`: returns indented XML or JSON, takes more place, but human-readable
	- `nolink`: drop links, but keeps links' inner text
	- `noref`: drop items' link
	- `cache`: only take articles from the cache (ie. don't grab new articles' content), so as to save time
	- `debug`: to have some feedback from the script execution. Useful for debugging
	- `theforce`: force download the rss feed and ignore cached http errros
	- `silent`: don't output the final RSS (useless on its own, but can be nice when debugging)
- http server only
	- `callback=NAME`: for JSONP calls
	- `cors`: allow Cross-origin resource sharing (allows XHR calls from other servers)
	- `txt`: changes the http content-type to txt (for faster "`view-source:`")
- Custom feeds: you can turn any HTML page into a RSS feed using morss, using xpath rules. The article content will be fetched as usual (with readabilite). Please note that you will have to **replace** any `/` in your rule with a `|` when using morss as a webserver
	- `items`: (**mandatory** to activate the custom feeds function) xpath rule to match all the RSS entries
	- `item_link`: xpath rule relative to `items` to point to the entry's link
	- `item_title`: entry's title
	- `item_content`: entry's description
	- `item_time`: entry's date & time (accepts a wide range of time formats)

## Use cases

morss will auto-detect what "mode" to use.

### Running on a server
#### Via mod_cgi/FastCGI with Apache/nginx

For this, you'll want to change a bit the architecture of the files, for example
into something like this.

```
/
├── cgi
│   │
│   ├── main.py
│   ├── morss
│   │   ├── __init__.py
│   │   ├── __main__.py
│   │   ├── morss.py
│   │   └── ...
│   │
│   ├── dateutil
│   └── ...
│
├── .htaccess
├── index.html
└── ...
```

For this, you need to make sure your host allows python script execution. This
method uses HTTP calls to fetch the RSS feeds, which will be handled through
`mod_cgi` for example on Apache severs.

Please pay attention to `main.py` permissions for it to be executable. Also
ensure that the provided `/www/.htaccess` works well with your server.

#### Using uWSGI

Running this command should do:

```shell
uwsgi --http :8080 --plugin python --wsgi-file main.py
```

#### Using Gunicorn

```shell
gunicorn morss:cgi_standalone_app
```

#### Using docker

Build & run

```shell
docker build https://git.pictuga.com/pictuga/morss.git -t morss
docker run -p 8080:8080 morss
```

In one line

```shell
docker run -p 8080:8080 $(docker build -q https://git.pictuga.com/pictuga/morss.git)
```

#### Using morss' internal HTTP server

Morss can run its own HTTP server. The later should start when you run morss
without any argument, on port 8080.

```shell
morss
```

You can change the port like this `morss 9000`.

#### Passing arguments

Then visit:
```
http://PATH/TO/MORSS/[main.py/][:argwithoutvalue[:argwithvalue=value[...]]]/FEEDURL
```
For example: `http://morss.example/:clip/https://twitter.com/pictuga`

*(Brackets indicate optional text)*

The `main.py` part is only needed if your server doesn't support the Apache redirect rule set in the provided `.htaccess`.

Works like a charm with [Tiny Tiny RSS](http://tt-rss.org/redmine/projects/tt-rss/wiki), and most probably other clients.

### As a CLI application

Run:
```
morss [argwithoutvalue] [argwithvalue=value] [...] FEEDURL
```
For example: `morss debug http://feeds.bbci.co.uk/news/rss.xml`

*(Brackets indicate optional text)*

### As a newsreader hook

To use it, the newsreader [Liferea](http://lzone.de/liferea/) is required
(unless other newsreaders provide the same kind of feature), since custom
scripts can be run on top of the RSS feed, using its
[output](http://lzone.de/liferea/scraping.htm) as an RSS feed.

To use this script, you have to enable "(Unix) command" in liferea feed settings, and use the command:
```
morss [argwithoutvalue] [argwithvalue=value] [...] FEEDURL
```
For example: `morss http://feeds.bbci.co.uk/news/rss.xml`

*(Brackets indicate optional text)*

### As a python library

Quickly get a full-text feed:
```python
>>> import morss
>>> xml_string = morss.process('http://feeds.bbci.co.uk/news/rss.xml')
>>> xml_string[:50]
"<?xml version='1.0' encoding='UTF-8'?>\n<?xml-style"
```

Using cache and passing arguments:
```python
>>> import morss
>>> url = 'http://feeds.bbci.co.uk/news/rss.xml'
>>> cache = '/tmp/morss-cache.db' # sqlite cache location
>>> options = {'csv':True}
>>> xml_string = morss.process(url, cache, options)
>>> xml_string[:50]
'{"title": "BBC News - Home", "desc": "The latest s'
```

`morss.process` is actually a wrapper around simpler function. It's still
possible to call the simpler functions, to have more control on what's happening
under the hood.

Doing it step-by-step:
```python
import morss, morss.crawler

url = 'http://newspaper.example/feed.xml'
options = morss.Options(csv=True) # arguments
morss.crawler.sqlite_default = '/tmp/morss-cache.db' # sqlite cache location

url, rss = morss.FeedFetch(url, options) # this only grabs the RSS feed
rss = morss.FeedGather(rss, url, options) # this fills the feed and cleans it up

output = morss.FeedFormat(rss, options, 'unicode') # formats final feed
```

## Cache information

morss uses caching to make loading faster. There are 3 possible cache backends
(visible in `morss/crawler.py`):

- `{}`: a simple python in-memory dict() object
- `SQLiteCache`: sqlite3 cache. Default file location is in-memory (i.e. it will
be cleared every time the program is run
- `MySQLCacheHandler`

## Configuration
### Length limitation

When parsing long feeds, with a lot of items (100+), morss might take a lot of
time to parse it, or might even run into a memory overflow on some shared
hosting plans (limits around 10Mb), in which case you might want to adjust the
different values at the top of the script.

- `MAX_TIME` sets the maximum amount of time spent *fetching* articles, more time might be spent taking older articles from cache. `-1` for unlimited.
- `MAX_ITEM` sets the maximum number of articles to fetch. `-1` for unlimited. More articles will be taken from cache following the nexts settings.
- `LIM_TIME` sets the maximum amount of time spent working on the feed (whether or not it's already cached). Articles beyond that limit will be dropped from the feed. `-1` for unlimited.
- `LIM_ITEM` sets the maximum number of article checked, limiting both the number of articles fetched and taken from cache. Articles beyond that limit will be dropped from the feed, even if they're cached. `-1` for unlimited.

### Other settings

- `DELAY` sets the browser cache delay, only for HTTP clients
- `TIMEOUT` sets the HTTP timeout when fetching rss feeds and articles

### Content matching

The content of articles is grabbed with our own readability fork. This means
that most of the time the right content is matched. However sometimes it fails,
therefore some tweaking is required. Most of the time, what has to be done is to
add some "rules" in the main script file in *readability* (not in morss).

Most of the time when hardly nothing is matched, it means that the main content
of the article is made of images, videos, pictures, etc., which readability
doesn't detect. Also, readability has some trouble to match content of very
small articles.

morss will also try to figure out whether the full content is already in place
(for those websites which understood the whole point of RSS feeds). However this
detection is very simple, and only works if the actual content is put in the
"content" section in the feed and not in the "summary" section.

***

## Todo

You can contribute to this project. If you're not sure what to do, you can pick
from this list:

- Add ability to run morss.py as an update daemon
- Add ability to use custom xpath rule instead of readability
- More ideas here <https://github.com/pictuga/morss/issues/15>
