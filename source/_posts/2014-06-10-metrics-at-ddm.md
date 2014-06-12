title: Metrics at DDM
date: 2014-06-11 16:49:03
author: Justin Carmony
tags: 
  - metrics
  - graphite
  - statsd
  - grafana
---
At DDM we're lucky to stand on the shoulders of giants in regards to collecting and monitoring the metrics we collect. It hasn't ever been easier to setup your own internal metrics for what is happening in your production environment. If you've never read the post by Etsy's team on [Measure Anything, Measure Everything](http://codeascraft.com/2011/02/15/measure-anything-measure-everything/), go read that first then come back.

We've tried to adopt Etsy's attitude on measuring all different aspects of our application and then setup graphs for the data. One of the challenges is that for a company our size we do a significant amount of traffic. The costs of a tool like New Relic would be significant. We also have a pretty locked down internal environment where our servers live, so it is easier for us to implement something internally.

So while we can't outsource our metrics, we also have a lot of time to manage a complicated system. It needed to be low maintenance and easy to use. We don't have a dedicated "DevOps" team, and our Dev Team size is small compared to the # of products & traffic we serve. So what we've implemented should work for the smallest of projects & websites.

## Our Setup

We use the following tools to track and measure system and application metrics:

- [StatsD](https://github.com/etsy/statsd/) - Network Daemon that runs on [Node.js](http://nodejs.org/) that collects metrics and timings, aggregates them, and sends them to Graphite.
- [Graphite](http://graphite.readthedocs.org/en/latest/) - Stores real-time data & outputs rendered graphs or json data.
- [Grafana](http://grafana.org/) - An open source Dashboard & Graph Editor. Not too complicated, but has just enough features to make it powerful & useful. Uses ElasticSearch as it's backend, very lightweight.

We send all of our metrics over UDP to StatsD. StatsD will then collect that data and flush it to Graphite every second. We then use the Graphite interface to browse our data or quick one-off graphs. Once we know we have the data we want, we'll create graphs in Grafana. Pretty darn simple, and the time of rolling out an additional metric to track and graph it is minutes.

There are lots of great tutorials for setting this stuff up, so instead of step-by-step, we'll mention a few standout things that have helped our setup.

### StatsD Setup

Our settings are default except for two. We set flushInterval to 1000 milliseconds (every second). We set the graphite.legacyNamespace to false sine the new namespace layout is much cleaner and easier to navigate:

``` son
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

We use the same retention policy for all of our metrics. We might change to only retain 90 days, but we want to see if Year over Year data would be useful. Here are our retention policy settings:

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

While setting up and maintaining our environment, we have had a few hiccups that we've needed to address.

One hangup we've had is with Graphite's maxDataPoints feature which is missing from the latest release of Graphite. Grafana's wiki has some information about this, but you can easily [update two files manually to add the ability back](https://github.com/grafana/grafana/wiki/Performance-for-large-time-spans). This is really important for the performance of Grafana.

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

We wrap our StatsD instance in a singleton (gasp! I know) that is autoloaded. The reason is you end up sprinkling StatsD stuff all over your application. You don't want to worry about making sure it is just setup properly. You definitely could use some more proper method of dependency injection, but for this particular case the singleton worked well for us.

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

The latest version of Graphite has awesome support for events. You just call a REST api to Push an event. Here is an example doing it in ``bash`` with ``curl``:

``` bash
curl -X POST "http://stats.example.com/events/" \
    -d '{"what": "Example Title", "tags": "deploy-example-project", "data": "$SUDO_USER just updated $branch_name on Example Project Production"}'
```

## Our Dashboards

Grafana has been the missing piece for us with using Graphite and StatsD, it makes it very easy to configure some useful dashboards and save them for later. Previous tools were either too cumbersome to use, or we would just hardcode some custom D3 charts.

Now anyone can easily go add some Dashboards to Grafana (even our Product Owners). Here is an example of one dashboard I use almost every day:

![DNews Graph](/images/metrics/dnews-graph.jpg)

You can see we have Grafana annotations displaying our deploy event from Graphite Events. We have internal data from our App showing # of queries, but also metrics from the databases showing the # of queries they are seeing. We have data from our external visitor services, and the core timings we care about for our website.

There are a few graphs we've found helpful I'd like to call out.

### Timings

Timings are crucial for capturing when releases increase or decrease the performance of our website. A timing is super simple to do:

- Get the start time in milliseconds
- Do whatever code / action
- Get the end time in milliseconds
- Compare & send total milliseconds elapsed to StatsD

Most StatsD clients have built in helpers to do just that. If you're using the StatsD client we've recommended and wrapped it into a singleton, you could just do this:

``` php
ExampleStatsD::getInstance()->startTiming('example.metric');

/* Do some code */

ExampleStatsD::getInstance()->endTiming('example.metric');
```

Here is an example of how we do a timing for how long our all our PHP calls take to execute for a particular project:

![PHP All Calls Timing Example](/images/metrics/timings.jpg)

I've added the two arrows myself. You can see we did a deploy and suddenly everything slowed down. We were able to hunt down the bug, fix it, and do another release. Once released we were then able to immediately verify that we did fix the bug.

### Capturing Micro Events

Tracking high volume metrics is important, but there are low volume metrics just as crucial. One example is a scenario where our contributor system will push over a story to our content system. That event only happens ~40 times a day, but is vital for keeping up-to-date content. Other examples of low volume metrics could be user registrations, comments, or checkouts.

The challenge is Graphite/StatsD work in a "per second" view. When the second data is rolled up into a 10 second or minute granularity, you get the per second average, not the total count. This isn't very useful for low volume metrics.

Graphite has a solution with the function "hitcount." You can pass in a time period (5min, 1h, etc) and it will confer per second to total number of "hits". Here is a graph of the hitcount per 5 minutes:

![# of Stories Crossed Per 5 Minutes](/images/metrics/metric-events.jpg) 

This is much easier to see if I go a full hour without stories crossing, I know I need to take a look.

### Slave Replication

Using our statsd-mysql-watcher we can have the master and slaves independently report their current binary log position. We've had problems in the past where the connection between our masters and slaves would die but wouldn't realize it. The master would update, but the slave's ``seconds_behind_master`` would still be 0 because it's connection was dead.

Here we are showing the current binary log position of the master and of the two slaves. You can see they are uniform, which means they are all on the near identical position. Then Slave 1 deviates and doesn't follow the other two. It's connection had died which stopped replication. This graph helped us spot the problem within a minute. Once fixed you can see the slave's position move back in sync, and we knew in real-time we had fixed the problem.

![Slave Replication](/images/metrics/slave-replication.jpg)

## Whats Next

For us, we're looking into better integrating our Nagios alerts into our metrics. The slave replication is an excellent candidate for an alert. So hopefully you'll see more posts on how we integrate the two.

We also want to get [Skyline](https://github.com/etsy/skyline) and [Oculus](https://github.com/etsy/oculus) setup to help recognize and detect anomalies in our graphs.
	
Have any questions? Have suggestions of things we should check out? Feel free to leave some comments!

Justin Carmony,
DDM Tech Team