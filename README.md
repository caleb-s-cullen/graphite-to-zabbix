# Graphite to Zabbix proxy
This tool allows Zabbix to handle alerts based on Graphite metrics. It works as a proxy between Graphite and Zabbix. It uses Graphite as the data source and Zabbix as an alerting system.

The basic idea is to schedule cronjob to run the script.  It makes some requests of the Zabbix API on the server, to get filtered list of monitored metrics, their hosts and those hosts' proxies.  Then it formulates and sends the appropriate requests to Graphite before repackaging and sending the metrics back to Zabbix.

This version is a modified version of the original g2zproxy by Alexey Dubkov (?) at https://github.com/adubkov/graphite-to-zabbix/blob/master/g2zproxy

I (ccullen@easydns.com) have modified it in the following ways:

0. I have updated the script to modern recommended syntax for `pyzabbix` library import, and otherwise updated the script for Python 3.  It is no longer reverse-compatible to Python 2.
0. I have changed the logging code to use rsyslog, on facility `local0`.  There is a comment nearby to enable basic logging to `stdout` once again, but that seems a weird choice for a cronjob.
1. It is now possible to tell the program to strip the hostname (coming from Zabbix, from the API call) down to only the first dot-separated token.  It would be trivial to adjust this to accept a number of starting tokens to keep.  To get this functionality, use the `-n` option.
2. The Graphite metric key was previously required to start with the hostname.  This is still the default behavior, but if the Zabbix metric key contains the string "{h}" then it will parsed with `.format(h=hostname)` where `hostname` has had the transformation described in (1) applied.  This allows for extraction of metrics which are not organized with their host at the top of the hierarchy, e.g. `srv.{h}.cpu.cpu_count`
3. Two different classes have been added, to assist in organizing information in different ways.
   a.  The JSONMetricsSorter class is designed to accumulate metrics for a bunch of different hosts, and then spit those metrics out into a single JSON payload.  This may have future uses, but I am not sure how well a string containing both `[]` and `{}` functions as a key in a JSON dict. That said, some simple logic would allow for the metric keys to be munged in a predictable way; perhaps everything after the `{h}`, inside the `[]`.  This of course requires the use of a "raw" text trapper item and one or more dependent items, in Zabbix.
   b.  The MetricsSorterByProxy class accumulates metrics into buckets (lists) organized by the name/address of the proxy to which they must go, based on information discovered via API.  Please note, this was developed against Zabbix 5.0 -- if you are using an earlier version, the API call involved may need some adjustment.

In its current state, the script accesses Zabbix to find a list of the metrics which are configured to come from graphite, i.e. their keys start with `graphite[`.  It figures out what the proxy is for each host, and gets the metrics, and then sends a payload to each proxy with data meant for its hosts.  It doesn't need to read the Zabbix config file or be told what server to use, to be able to do this -- except that it needs to know the address of the API server of course.

If the `-zs` option is used, that will override any information obtained about what proxies the data should go to.

## How to install
Easiest way to install g2zproxy over pip:
```bash
pip install graphite-to-zabbix
```

## How to use
To run g2zproxy once, just run cmd:
```bash
g2zproxy -z https://zabbix.local -zu {zabbixUser} -zp {zabbixPass} -g http://graphite.local
```
Note that g2zproxy will work with zabbix web api specified in `-z` argument, but it will send metrics to service specified in `/etc/zabbix/zabbix_agentd.conf`.


### Create zabbix metrics
First you need create few metrics to monitor in zabbix. I suppose you familar with zabbix template system. So, I just show to how to make one Item.

Suppose you want have an zabbix alert for some data from graphite. G2ZProxy will join `hostname` and graphite[`key`] from zabbix, and match it with graphite key.
An example graphite key `p-mem001.memcached.memcached_items-current.value` will be match to zabbix
```
host: p-mem001
key: graphite[memcached.memcached_items-current.value]
```

#### Graphite request with functions can be written in that manner.
Graphite request:
```
summarize(sum(statsd.drive_*error*),"5min","avg",true)
```
appropriate zabbix key:
```
graphite[statsd.derive_*error*; summarize(sum({metric}),\"5min\",\"avg\",true)]"
```

![alt tag](https://cloud.githubusercontent.com/assets/4600857/6422051/3749f056-be89-11e4-9dcd-78a1823f9e2a.png)

After you choosed which metric you want monitor, create zabbix item:

1. Create template:

![alt tag](https://cloud.githubusercontent.com/assets/4600857/6421550/e44f0b7e-be84-11e4-99bd-d9775e43b9a0.png)

2. Create item:

![alt tag](https://cloud.githubusercontent.com/assets/4600857/6421551/e44f33f6-be84-11e4-8cfc-175c6d052254.png)

3. Assign template to host:

![alt tag](https://cloud.githubusercontent.com/assets/4600857/6421552/e4510bea-be84-11e4-9b12-73b40eddfe34.png)

### Schedule cronjob task
Make cron task to run g2zproxy each minute:
```bash
$ crontab -e
```
```bash
# graphite to zabbix proxy
*/1 * * * * g2zproxy -z https://zabbix.local -zu {zabbixUser} -zp {zabbixPass} -g http://graphite.local > /dev/null 2>&1
```

### Performance
If g2zproxy seems to work slow, most likely you graphite is bottleneck. It's recomend to user distributed graphite cluster with loadbalanced to make it faster.

By default g2zproxy use 50th thread pool, you can change with `-t 10` argument.
