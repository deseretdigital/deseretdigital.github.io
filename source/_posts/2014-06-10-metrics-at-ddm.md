title: Metrics at DDM
date: 2014-06-10 16:49:03
tags: 
  - metrics
  - graphite
  - statsd
  - grafana
---
At DDM we're lucky to stand on the shoulders of giants in regards to collecting and monitoring the metrics we collect. If you've never read the post by Etsy's team on "[Measure Anything, Measure Everything](http://codeascraft.com/2011/02/15/measure-anything-measure-everything/)", go read that first then come back.

We've tried to adopt Etsy's attitude on measuring all different aspects of our application and then use graphs to monitor said data. One of the challenges is that for a company our size, we do a significant amount of traffic, so the costs of a tool like New Relic would be significant. We also have a pretty locked down internal environment where our servers live, so it is easier for us to implement something internally.

One important thing for us is low maintenance and ease of use. We don't have a dedicated "DevOps" team, and our Dev Team size is small compared to the # of products & traffic we serve. So what we've implemented should work for the smallest of developer shops & websites.

## Our Setup

We use the following tools to track and measure system and application metrics:

- [StatsD](https://github.com/etsy/statsd/) - Network Daemon that runs on [Node.js](http://nodejs.org/) that collects metrics and timings, aggregates them, and sends them to Graphite.
- [Graphite](http://graphite.readthedocs.org/en/latest/) - Stores real-time data & outputs rendered graphs or json data.
- [Grafana](http://grafana.org/) - An open source Dashboard & Graph Editor. Uses ElasticSearch as it's backend, very lightweight.

We send all of our metrics over UDP to StatsD. StatsD will then collect that data flush it to Graphite every second. We then use the Graphite interface to browse our data or quick one-off graphs. Once we know we have the data we want, we'll create graphs in Grafana. Pretty darn simple.

There are lots of great tutorials for setting this stuff up, so instead of step-by-step, we'll mention a few standout things that have helped our setup.

### StatsD Setup

The only two settings we don't use that are default are we set our flushInterval to 1000 milliseconds, and we set the graphite.legacyNamespace to false sine the new namespace layout is much cleaner and easier to navigate:

``` json
{
  graphitePort: 2003
, graphiteHost: "localhost"
, port: 8125
, backends: [ "./backends/graphite" ]
, flushInterval: 1000
, graphite: {
    legacyNamespace: false
  }
}
```

### Graphite Setup

We have our retention policies setup like so:

```
[stats]
pattern = ^stats.*
retentions = 1s:1h,10s:24h,1m:15d,10m:90d,60m:400d

[carbon]
pattern = ^carbon\.
retentions = 60:90d

[default_1min_for_1day]
pattern = .*
retentions = 60s:1d
```

This ends up making each metric about 600k each that we track. 

One hangup we've had is with Graphite's maxDataPoints feature which is missing from the latest release of Graphite. Grafana's wiki has some information about this, but you can easily [update two files manually to add the ability back](https://github.com/grafana/grafana/wiki/Performance-for-large-time-spans).

The other is [a caching bug](https://github.com/graphite-project/graphite-web/issues/576) we've encountered where graphite will sometimes return the png of a graph instead of the json data for it, breaking Grafana. The solution was to turn off the Cache by setting a Dummy Backend in Graphite's ``webapp/graphite/local_settings.py`` by adding this at the bottom:

``` python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    },
}
```

### Grafana Setup

Grafana was dead simple to setup. The biggest thing is when setting up [ElasticSearch](http://www.elasticsearch.org/) make sure to change the default cluster name, or other ElasticSearch instances could accidentally delete your data.

We just use an Apache alias to point a subdirectory of our graphite server to Grafana. (i.e. http://stats.example.com/grafana)

## Getting Data In

Once we had our StatsD / Graphite / Grafana combo setup, we needed to get data into it. Thankfully StatsD is popular enough that there are many clients for just about any language out there.

### Application Metrics in PHP

In our PHP Apps we use the [domnikl/statsd](https://packagist.org/packages/domnikl/statsd) package, though there are many others out there.

We wrap our StatsD instance in a singleton (gasp! I know) that is autoloaded. The reason is you end up sprinkling StatsD stuff all over your application without worrying about getting it setup properly. You definitely could use some more proper method of dependency injection, but for this particular case it worked well.

### Application Metrics in Node

We really like the [statsd-client](https://www.npmjs.org/package/statsd-client) for Node. Easy to use and it just works.

### JSON Data From Services

We found a great little tool from the folks at FictiveKin called [funnel](https://github.com/fictivekin/funnel). We use this to get data from a JSON endpoint and send it along StatsD. An example is our [Chartbeat](https://chartbeat.com/) visitor real-time data. So using funnel we have a script that looks something like this:

``` javscript
#!/usr/sbin/node

var StatsD_Server = '10.20.30.40';
var StatsD_Port = 8125;
var funnel = require('funnel');

/* Chartbeat Status */
var collectChartbeat = function(){
    var source = funnel.json({
    name: 'chartbeat',
    prefix: 'dnews.local',
    services: {
            'links': 'links'
            ,'people': 'people'
            ,'read': 'read'
            ,'direct': 'direct'
            ,'visits': 'visits'
            ,'subscr': 'subscr'
            ,'pages': 'pages'
            ,'search': 'search'
            ,'crowd': 'crowd'
            ,'write': 'write'
            ,'platform_mobile': 'platform.m'
            ,'platform_desktop': 'platform.d'
            ,'idle': 'idle'
            ,'internal': 'internal'
            ,'social': 'social'
        },
        from: 'http://api.chartbeat.com/live/quickstats/v3/?apikey=mysupersecretapikey&host=example.com'
    });

    funnel.collect(source).toStatsD(StatsD_Server, StatsD_Port);

    setTimeout(collectChartbeat, 5000); 
}

collectChartbeat();
```

Having this external data next to internal data is great, because sometimes things look like they are normal when the packets leave our server, but when visitors spike downwards suddenly we know something is still wrong.

### Monitoring MySQL

To our monitor our mysql instances we wrote a tool call the [statsd-mysql-watcher](https://github.com/deseretdigital/statsd-mysql-watcher) (original name, I know!). We're still working on some documentation, but it basically is a node.js app that you can pass a configuration file and have it watch a server & poll stats data from it.

### Events / Deploys to Production

The latest version of graphite has awesome support for events. You just call a REST api to Push an event. Here is an example doing it in ``bash`` with ``curl``:

``` bash
curl -X POST "http://stats.example.com/events/" \
    -d '{"what": "Example Title", "tags": "deploy-example-project", "data": "$SUDO_USER just updated $branch_name on Example Project Production"}'
```

## Our Dashboards

Grafana has been the missing piece for us with using Graphite and StatsD, it makes it very easy to configure some useful dashboards and save them for later. Previous tools were either too cumbersome to use, or we would just hardcode some custom D3 charts.

Now anyone can easily go add some Dashboards to Grafana. Here is an example of one dashboard I use almost every day:

![DNews Graph](/images/metrics/dnews-graph.jpg)


