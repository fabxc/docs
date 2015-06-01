---
title: Service Discovery Introduction
created_at: 2015-06-02
kind: article
author: Fabian Reinartz
---

This week we released Prometheus v0.14.0 — a version with many long-awaited additions
and improvements.

On the user side, new mechanisms for service discovery aim to make it easier to
dynamically update the targets Prometheus scrapes. In addition to DNS-SRV records,
[Consul](https://www.consul.io) is now supported out-of-the-box and a file-based interface
allows you to connect your own discovery mechanism.
Over time, we plan to extend Prometheus with native support for other common ways of doing
service discovery.

Aside from many smaller fixes and improvements, you can now also reload your configuration at 
runtime by sending a `SIGHUP` to the Prometheus process. For a full list of changes, check the
[changelog for this release](tba).

In this blog post, we will take a closer look at the built-in service discovery mechanisms and provide
small practical examples. As an additional resource
[Prometheus's configuration documentation](http://prometheus.io/docs/operating/configuration/),
can be helpful.


## Scrape Configurations and Relabeling

Aside from switching from the ASCII protocol buffer format to YAML, a fundamental change to 
Prometheus's configuration is the switch from per-job to scrape configurations.
Scrape configurations can be thought of equivalent to job configurations for simple setups.
At the same time, they allow greater flexibility for more advanced use cases.

Each scrape configuration is assigned a job name which serves as a default value for the
`job` label. The `job` label can then be redefined for whole target groups or single targets.
For example, we can define two target groups, each of which defines targets for one job. 
To scrape them with the same parameters, we can configure them as follows:

```
scrape_configurations:
- job_name: 'overwritten-default'

  scrape_interval: 10s
  scrape_timeout:  5s

  target_groups:
  - targets: ['10.1.200.130:5051', '10.1.200.134:5051']
    labels:
      job: 'job1'

  - targets: ['10.1.200.130:6220', '10.1.200.134:6221']
    labels:
      job: 'job2'
```

Through a mechanism named _relabeling_ any label can be removed, created, or modified on a per-target level. 
This allows fine-grained labeling that can also enable meta-data coming from the service discovery.

Relabeling works as follows:

- A list of source labels is defined
- For each target the values of those labels are concatenated with a
  separator
- A regular expression is matched against the resulting string
- A new value based on those matches is assigned to another label

Mutiple relabeling rules can be defined for each scrape configuration. A simple one
that squashes two labels into one, looks as follows:

```
relabel_configs:
- source_labels: ['label_a', 'label_b']
  separator:     ';'
  regex:         '(.*);(.*)'
  replacement:   '$1-$2'
  target_label:  'label_c'
```

This rule transforms a target with the label set

```
{
  "job": "job1",
  "label_a": "foo",
  "label_b": "bar"
}
```
into a target with label set

```
{
  "job": "job1",
  "label_c": "foo-bar"
}
```

You can read more about relabeling, and how you can use it to filter targets, in the
[configuration documentation](/docs/operating/configuration#relabeling-relabel_config).

Over the next sections, we will see how you can leverage relabeling when using service discovery.


## Discovery with DNS-SRV records

Prometheus shipped with support for target discovery via DNS-SRV records for some time.
The respective configuration looked like this:

```
job {
  name: "api-server"
  sd_name: "telemetry.server.prod.api.srv.example.org"
  metrics_path: "/metrics"
}
```

With the latest changes not only can you describe multiple SRV records to be queried but can
also use meta labels, that are attached by the service discovery, during the relabeling phase.

When querying the the DNS-SRV records, a label named `__meta_dns_srv_name` is attached to each
target. It's value is set to the record name for which it was returned. Having structured
SRV record names, e.g. `telemetry.<zone>.<job>.srv.example.org`, we can extract various
labels from it:

```
scrape_configs:
- job_name: 'myjob'
  
  dns_sd_configs:
  - names:
    - 'telemetry.eu-west.api.srv.example.org'
    - 'telemetry.us-west.api.srv.example.org'
    - 'telemetry.eu-west.auth.srv.example.org'
    - 'telemetry.us-east.auth.srv.example.org'

  relabel_configs:
  - source_labels: ['__meta_dns_srv_name']
    regex:         'telemetry\.(.+?)\..+?\.srv\.example\.org'
    target_label:  'zone'
    replacement:   '$1'
  - source_labels: ['__meta_dns_srv_name']
    regex:         'telemetry\..+?\.(.+?)\.srv\.example\.org'
    target_label:  'job'
    replacement:   '$1'
```

This will attach the `zone` and `job` label to each target based on the SRV record
it came from.


## Discovery with Consul

Service discovery via Consul is now supported natively. We can configure it by defining 
access parameters for our consul agent and a list of Consul services for which we want
to query targets.

The tags of each Consul node are concatenated by a configurable separator and exposed 
through the `__meta_consul_tags` label. Various other meta labels are extracted.

Scraping all instances for a list of given services can be achieved with a simple
`consul_sd_config` and relabeling rules:

```
scrape_configs:
- job_name: 'overwritten-default'

  consul_sd_configs:
  - server:     '127.0.0.1:5361'
    datacenter: 'eu-west'

    services: ['auth', 'api', 'load-balancer', 'postgres']

  relabel_configs:
  - source_labels: ['__meta_consul_service']
    regex:         '(.*)'
    target_label:  'job'
    replacement:   '$1'
  - source_labels: ['__meta_consul_node']
    regex:         '(.*)'
    target_label:  'instance'
    replacement:   '$1'
  - source_labels: ['__meta_consul_tags']
    regex:         '(production|canary)'
    target_label:  'group'
    replacement:   '$1'
  - source_labels: ['__meta_consul_dc']
    regex:         '(.*)'
    target_label:  'zone'
    replacement:   '$1'
```

A full documentation of all configuration parameters for service discovery via Consul
can be found on the [Prometheus website](/docs/operating/configuration##relabeling-relabel_config).


## Custom Service Discovery

Finally, we added a file-based interface to talk to your custom service discovery or other common mechanisms, 
that are not yet supported out-of-the-box.

Prometheus watches a set of directories or files which contain target group information. Whenever
any of those files changes, a list of target groups is read from the files and scrape targets 
are extracted. 
It's now our job to write a small bridge program that runs as Prometheus's side-kick. 
It retrieves changes from a service discovery mechanism and writes the target information
to the watched files as lists of target groups.

The files can either be in YAML...

```
- targets: ['10.11.150.1:7870', '10.11.150.4:7870']
  labels:
    job: 'mysql'

- targets: ['10.11.122.11:6001', '10.11.122.15:6002']
  labels:
    job: 'postgres'
```

... or JSON format.

```
[
  {
    "targets": ["10.11.150.1:7870", "10.11.150.4:7870"],
    "labels": {
      "job": "mysql"
    }
  },
  {
    "targets": ["10.11.122.11:6001", "10.11.122.15:6002"],
    "labels": {
      "job": "postgres"
    }
  }
]
```

We now configure Prometheus to watch the `tgroups/` folder in its working directory 
for all `.json` files:

```
scrape_configs:
- job_name: 'overwritten-default'

  file_sd_configs:
  - names: ['tgroups/*.json']
```

What's missing now is a program that writes files to this directory. For the sake of this example, 
let's assume we have all our instances for different jobs in one large unnormalized MySQL table.
(Hint: you probably don't want to do service discovery this way.)

Every 30 seconds, we read all instances from the table and writes the result
into JSON files. Note that we do not have to keep state whether or not anything has changed. Prometheus
will automatically detect changes and applies them to targets without interrupting their scrape cycles.

```
import os, time, json
 
from itertools import groupby
from MySQLdb import connect
 
 
def refresh(cur):
    # Fetch all rows.
    cur.execute("SELECT address, job, zone FROM instances")
 
    tgs = []
    # Group all instances by their job and zone values.
    for key, vals in groupby(cur.fetchall(), key=lambda r: (r[1], r[2])):
        tgs.append({
            'labels': dict(zip(['job', 'zone'], key)),
            'targets': [t[0] for t in vals],
        })
 
    # Persist the target groups to disk as JSON file.
    with open('tgroups/target_groups.json.new', 'w') as f:
        json.dump(tgs, f)
        f.flush()
        os.fsync(f.fileno())
 
    os.rename('tgroups/target_groups.json.new', 'tgroups/target_groups.json')
 
 
if __name__ == '__main__':
    while True:
        with connect('localhost', 'root', '', 'test') as cur:
            refresh(cur)
        time.sleep(30)
```

While Prometheus will not apply any malformed changes to files, it is considered best practice to
change your files via renaming, as we do in our example.
It is also recommended to split larger amounts of target groups into several files based on
logical grouping.


## Conclusion

With DNS-SRV records and Consul two major ways of doing service discovery are now natively supported
by Prometheus. From the examples we've seen that relabeling is a powerful approach to make use
of meta data provided by service discovery mechanisms.

Make sure to take a look at the new [configuration documentation](/docs/operating/configuration/)
to upgrade your Prometheus setup to the new release and find out about other configuration options
such as basic HTTP authentication and target filtering via relabeling.

We provide a [migration tool](https://github.com/prometheus/migrate/releases) that upgrades
your existing configuration files to the new YAML format.
For smaller configurations we recommend a manual upgrade to get familiar with the new format and
to preserve comments.
