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
# launch brozzler
1$ brozzler-easy
```

A web dashboard with job statuses is available at <http://localhost:8881>, and pywb at <http://0.0.0.0:8880>.

Warcs are output to `1$ $PWD/warcs`, and you can check the dashboard to see which warc corresponds to which job. A larger job may have multiple warcs, each with a sequence number, soft limited at 1GB.

This can be wrapped up in nice service files and scaled both at single and multiple hosts, but docs for that are TBD.

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