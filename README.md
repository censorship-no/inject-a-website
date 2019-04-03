# Inject a website into ouinet

Injection is a three-step process: you

1. Fetch the website

   *(transfer the archive to an airgapped signing computer)*

2. Do offline injection with [ouinet-inject](https://github.com/equalitie/ouinet-inject)

   *(put the data on a usb drive and cross the censored zone)*

3. Upload to the network with [ouinet-upload](https://github.com/equalitie/ouinet-upload)

This note shall focus on (1); check documentation of the corresponding tools for reference on the later steps, and raise an issue if it's lacking.

## Fetching

To fetch a website, one can use any [WARC](https://en.wikipedia.org/wiki/Web_ARChive)-[compatible tool](https://www.archiveteam.org/index.php?title=The_WARC_Ecosystem); we'll cover `wget` and [brozzler](https://github.com/internetarchive/brozzler).

### wget

For cases where a site you want to inject is a simple static one, `wget` is lighterweight and easier to start with:

```sh
wget --delete-after --no-directories --warc-file=example --recursive --level=1 https://example.com/
#    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^             ^^^^^^^             ^^^^^^^^^
#    don't leave "normal" files,                 result              recursion
#    we only want a WARC                         name                level
```

Find a WARC at `example.warc` and proceed with injection steps 2 and 3!

### brozzler

For JS-heavy sites like BBC or Instagram, we use a headful chromium in a virtual framebuffer (xvfb) with a [WARC-writing MITM proxy](https://github.com/internetarchive/warcprox), controlled via remote debug, jobs and dedup data stored in rethinkdb. Replay is done via [pywb](https://github.com/webrecorder/pywb).

Note that deduplication and capture are global between jobs, and we do not currently have a simple way to either get full warcs (with no references to other warcs) for a single job, or to get warcs for only this particular job. Thus, it is recommended to run a separate instance of brozzler for each indepent set of crawljobs.

These instructions were tested on Fedora 29, but should be easy to adapt to any GNU/Linux.

Install rethinkdb:

```sh
sudo curl -L http://download.rethinkdb.com/centos/7/`uname -m`/rethinkdb.repo -o /etc/yum.repos.d/rethinkdb.repo
sudo dnf install -y rethinkdb
```

#### Ad-hoc setup

```
# proceeding w/tmux
# X$ indicated virtual console X

0$ rethinkdb &>>rethinkdb.log &

1$ virtualenv brozzler.env
1$ source brozzler.env/bin/activate
1$ pip install https://github.com/censorship-no/brozzler[easy]

# add a new site to fully recursively crawl
1$ brozzler-new-site https://example.com/
# launch a brozzler-{dashboard,wayback,worker} and warcprox all-in-one
1$ brozzler-easy
```

A web dashboard with job statuses is available at <http://localhost:8881>, and pywb at <http://0.0.0.0:8880>.

Warcs, softlimited at 1GB, are output to `1$ $PWD/warcs`.

You can check pywb to see which warc corresponds to which url, like so: <http://localhost:8880/brozzler/*/https://example.com>.

#### Configurable crawling jobs

For instance, to crawl Persian BBC at maxdepth=4:

```yaml
# bbc-job.yml
seeds:
  - url: http://www.bbc.com/persian
    scope:
      max_hops: 4
```
```sh
$ brozzler-new-job bbc-job.yml
```

See [reference](https://github.com/censorship-no/brozzler/blob/master/job-conf.rst) for more details.

#### Skip cruft capture

To only capture relevant content and skip e.g. advertising requests, one can either employ a manual whitelist (see crawling jobs), or use an adblocker (essentially a crowdsourced blacklist).

For the latter, we'll use chromium's [external extensions](https://developer.chrome.com/extensions/external_extensions.html). For uBlock Origin:

Download its crx (a signed zip distribution file) [somehow](https://stackoverflow.com/questions/7184793), and place it to `/usr/share/chromium/extensions/cjpalhdlnbpafiamejdnhcphjbkeiagm.crx`. Then add a config file like so (note you can't actually have hash-comments in json!):

```
# /usr/share/chromium/extensions/cjpalhdlnbpafiamejdnhcphjbkeiagm.json
#                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                                extension id, must match
{
  "external_crx": "/usr/share/chromium/extensions/cjpalhdlnbpafiamejdnhcphjbkeiagm.crx",
  "external_version": "1.18.4" # extension version
}
```

On next launch chromium should automatically pick the extension up!

#### Skip capturing big content

Use following options:
```sh
warcprox  --max-resource-size 5000000
brozzler-worker --skip-youtube-dl
```

#### Explicit setup

Going from an all-in-one `brozzler-easy` to explicitly launching and configuring the services. To launch a single instance, in a new directory do:

Create a config file for the online wayback service:
```yaml
# pywb.yml
archive_paths: ./warcs/
collections:
  brozzler:
    index_paths: !!python/object:brozzler.pywb.RethinkCDXSource
      db: brozzler
      table: captures
      servers:
      - localhost
enable_auto_colls: false
enable_cdx_api: true
framed_replay: true
port: 8880
```

And start everything:

```sh
1$ rethinkdb
2$ warcprox -p 9999 -z --base32 --rollover-idle-time 180 --rethinkdb-stats-url 'rethinkdb://localhost/brozzler/stats' --rethinkdb-big-table-url 'rethinkdb://localhost/brozzler/captures' --rethinkdb-services-url 'rethinkdb://localhost/brozzler/services' --max-resource-size 5000000
3$ brozzler-worker --warcprox-auto --skip-youtube-dl
4$ brozzler-dashboard
5$ PYWB_CONFIG_FILE=pywb.yml brozzler-wayback
```

Then, a web dashboard with job statuses is available at <http://localhost:8000>, and pywb at <http://0.0.0.0:8880>.
