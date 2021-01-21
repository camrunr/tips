This doc will decribe a few methods of collecting volume stats into a metrics index. This allows for quickly being able to report on usage trends over long periods of time.

There are 2 logs that track usage: license logs and metrics logs (not to be confused with metrics type indexes).
 
 ## License Log ##
 
The license log is the SOR. If you have a centralized license server, the license log will be created there. If you have license files installed on multiple indexers, you'll find license logs on all that have them. The license events consist of a byte value qualified by source (s), sourcetype (st), host (h), index (idx), pool, and indexer (i). Sample:

    LicenseUsage - type=Usage s="/var/log/apache/access_log" st=access h="" o="" idx="acme_main" i="C709F63F-8585-4DC9-9000-7355A8A2EC1D" pool="Pool1" b=295120 poolsz=6501171200000

The type for this one is listed as "Usage". There's also a daily summary type called "RolloverSummary". For our purposes here, I'm using `type=Usage` because sometimes I like to see intraday volume stats. The daily "RolloverSummary" events show you totals for the day. Your choice which you need.

The search below is what I use to collect volume stats hourly. NOTE: The final command sends the results into a metrics-type index called spl_metrics for later use. I have this index set for much longer retention than the _internal index.

```
   index=_internal source=*/license_usage.log host IN (license_servers)
    | addinfo
    | stats min(info_min_time) as _time sum(b) as B by h idx pool
    | eval _value=round(B/1024/1024,0),
           idx=if(idx=="","n/a",idx),
           h=if(h=="","n/a",h),
           metric_name="license.MB"
    | fields - B
    | mcollect index="spl_metrics"
```

In my environment this results in about 25,000 entries per run. If I include st (sourcetype) in the split, that jumps about 4x. YMWV. You may also to decide to have separate searches for the different split bys, rather than one. I like having the capability to report on combos of host, pool and index, so this works for me.

Here's a sample search using the metrics-fied data, giving me a weekly max-day reading by pool. A 1 year run returns very quickly:

```
    | mstats sum(license.MB) as mb_used  where index=spl_metrics span=1d by pool
    | timechart span=7d max(eval(mb_used/1024)) as GB by pool
```


## Metrics Logs ##    

The metrics log will be all Indexers and *usually* on Forwarders. It offers a "close enough" approximation of usage if you're mindful. Both sending Forwarders and receiving Indexers log metrics data, so it's important to make sure you're only calculating from one side or the other. Stats are tracked by host, source, sourcetype, and index. There is no cross-referencing in this file (ie, each event represents exactly one value of the type it is tracking; there is no way to grab how many MBs were logged to sourcetype_A and index_1).  
 
By default Splunk only tracks the top 10 of each type of metric in each 30 second post window. Example, it collects KBs sent, by index, for 30 seconds. When the 30 seconds is up, it writes an event for the top 10 indexes. This is adjustable in the limits.conf file (showing defaults below):
 
    [metrics]
    interval  = 30
    maxseries = 10
 
Adjusting these will impact storage, CPU, and/or memory usage.  
 
The metrics logs look like this:
 
    Metrics - group=per_index_thruput, series="acme_main", kbps=4.206, eps=1.771, kb=130.613, ev=55, avg_age=0.272, max_age=2
 

Remember, it's important to only use 1 side of the connection. You can query Forwarder hosts for metrics logs and that will give you higher cardinality on the series since each Forwarder has its own limit for what it tracks. Which is nice. However, HEC volume data will only be in the HEC receivers' metrics. If this is the case, it's probably better to only collect from the Indexer logs. One last caveat: Some Splunk admins turn off metrics on the Forwarder side, so Indexer side would be needed in that scenario.

In this sample below I'm using "per_index_thruput". I'd have this scheduled to run hourly. Substitute host, source, or sourcetype as appropriate. The series field will represent the name of the particular index represented. Also, as with the license version above, I've included the mcollect command to send the results back into Splunk for later use.
 
```
    index=_internal source=*/metrics.log* group=per_index_thruput host IN (indexerlist)
    | addinfo
    | stats min(info_min_time) as _time sum(kb) as KB by series
    | rename series as idx
    | eval _value=round(KB/1024,0),
           metric_name="volume.MB"
    | fields - KB
    | mcollect index="spl_metrics"
```

Here's a sample search using the metrics-fied metrics.log data, giving me a weekly max-day reading by index:

```
    | mstats sum(volume.MB) as mb_used  where index=spl_metrics span=1d by idx
    | timechart span=7d max(eval(mb_used/1024)) as GB by idx
```

I'd recommend running separate reports by index, host and maybe sourcetype. Source is less useful as log file names can get pretty unique, so you end up with thousands upon thousands of entries (and hit the limit discussed above). Not great.
 
It should be noted that internal indexes (_internal, _introspection, _audit) and summary indexes don't count against license. (Summary is only excluded from license if it's being used as intended. If a Forwarder sends data directly to a summary index, that still counts.)

---

Above are 2 methods for tracking log volume. Hopefully they provide a good starting point for you to create your own.

If you have questions, spot a mistake, or have a recommendation, feel free to submit a PR.

Enjoy
