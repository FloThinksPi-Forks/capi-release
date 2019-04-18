# Notice

These docs are now a part of the official Cloud Foundry documentation and can be viewed at the link below:

[Information for Operators: Scaling Cloud Controller](https://docs.cloudfoundry.org/running/managing-cf/scaling-cloud-controller.html)

Since this doc has been linked externally, going to keep it around here for a while, but any updates should be made [here](https://github.com/cloudfoundry/docs-running-cf/blob/master/managing-cf/scaling-cloud-controller.html.md.erb).

---

# Scaling Cloud Controller
The purpose of this document is to provide operators with guidance on knowing how and when to scale the jobs within capi-release. It is broken down by BOSH job (e.g. `cloud_controller_ng`) and highlights some of the key metrics, heuristics, and logs we have found helpful. These lists are not exhaustive by any means, but should serve as a good place to start.

As always, suggestions and contributions to this document are welcome!

## Jobs [cf-deployment instance group]
### cloud_controller_ng [api]
The `cloud_controller_ng` Ruby process can be considered the _main_ job in capi-release. It, along with `nginx_cc`, powers the Cloud Controller API that all users of Cloud Foundry interact with. In addition to serving external clients (e.g. the cf cli, web UIs, log/metrics aggregators), `cloud_controller_ng` also provides APIs for internal components within Cloud Foundry, such as the Loggregator and Networking subsystems.

#### When to scale
##### Key Metrics
* `cc.requests.outstanding` is at or consistently near 20
* `system.cpu.user` is above 0.85 utilization of a single core on the api vm (see footnote [1] on BOSH cpu metrics)
* `cc.vitals.cpu_load_avg` is 1 or higher
* `cc.vitals.uptime` is consistently low indicating frequent restarts (possibly due to memory pressure)

##### Heuristics
* Average response latency
* Degraded web UI responsiveness or timeouts

##### Logs
* `/var/vcap/sys/log/cloud_controller_ng/cloud_controller_ng.log`
* `/var/vcap/sys/log/cloud_controller_ng/nginx-access.log`

#### How to scale
Before and after scaling Cloud Controller API VMs, its important to verify that the CC's database is not overloaded. All Cloud Controller processes are backed by the same database (ccdb), so heavy load on the database will impact API performance regardless of the number of Cloud Controllers deployed. Cloud Controller supports both PostgreSQL and MySQL and the ability for users to "bring their own database" so we are unable to provide specific scaling guidance for the database (see footnote [2] on database performance).

In CF deployments with internal MySQL clusters, a single MySQL database VM with CPU usage over ~80% can be considered overloaded. When this happens, the MySQL VMs must be scaled up or the added load of additional Cloud Controllers will only exacerbate the problem. External

Cloud Controllers API VMs should primarily be scaled horizontally. Scaling up the number of cores on a single VM is not helpful because of Ruby's Global Interpreter Lock (GIL). This limits the `cloud_controller_ng` process so that it can only effectively use a single CPU core on a multi-core machine.

### cloud_controller_worker_local [api]
Colloquially known as "local workers," this job is primarily responsible for handling files uploaded to the API VMs during `cf push` (e.g. `packages`, `droplets`, resource matching).

#### When to scale
##### Key Metrics
* `cc.job_queue_length.cc-<VM_NAME>-<VM_INDEX>` (ie. `cc.job_queue_length.cc-api-0`) is continuously growing
* `cc.job_queue_length.total` is continuously growing

##### Heuristics
* `cf push` is intermittently failing
* `cf push` average time is elevated

##### Logs
* `/var/vcap/sys/log/cloud_controller_ng/cloud_controller_ng.log`

#### How to scale
Because local workers are colocated with the Cloud Controller API job, they are scaled horizontally along with the API.

### cloud_controller_worker [cc-worker]
Colloquially known as "generic workers" or just "workers", this job (and VM) is responsible for handling asynchronous work, batch deletes, and other periodic tasks scheduled by the `cloud_controller_clock`.

#### When to scale
##### Key Metrics
* `cc.job_queue_length.cc-<VM_TYPE>-<VM_INDEX>` (ie. `cc.job_queue_length.cc-cc-worker-0`) is continuously growing
* `cc.job_queue_length.total` is continuously growing

##### Heuristics
* `cf delete-org ORG_NAME` appears to leave its contained resources around for a long time
* Users report slow deletes for other resources
* cf-acceptance-tests succeed generally, but fail during cleanup

##### Logs
* `/var/vcap/sys/log/cloud_controller_worker/cloud_controller_worker.log`

#### How to scale
The cc-worker VM can safely scale horizontally in all deployments, but if your worker VMs have CPU/memory headroom you can also use the `cc.jobs.generic.number_of_workers` BOSH property to increase the number of worker processes on each VM.

### cloud_controller_clock and cc_deployment_updater [scheduler]
The `cloud_controller_clock` runs Diego sync process and schedules periodic background jobs. The `cc_deployment_updater` is responsible for handling v3 [rolling app deployments](https://docs.cloudfoundry.org/devguide/deploy-apps/rolling-deploy.html).

#### When to scale
##### Key Metrics
* `cc.Diego_sync.duration` is continuously increasing over time
* `system.cpu.user` is high on the scheduler VM (see footnote [1] on BOSH cpu metrics)

##### Heuristics
* Diego domains are [frequently unfresh](https://github.com/cloudfoundry/bbs/blob/master/doc/domains.md#domain-freshness)
* The Diego Desired LRP count is larger than the total process instance count reported via the Cloud Controller APIs
* Deployments are slow to increase and decrease instance count

##### Logs
* `/var/vcap/sys/log/cloud_controller_clock/cloud_controller_clock.log`
* `/var/vcap/sys/log/cc_deployment_updater/cc_deployment_updater.log`

#### How to scale
Both of these jobs are singletons (only a single instance is active), so extra instances are for failover HA rather than scalability. Performance issues are likely due to database overloading or greedy neighbors on the scheduler VM.

### blobstore_nginx [singleton-blobstore]
The internal [WebDAV](http://www.webdav.org/) blobstore that comes included with CF by default. It is used by the platform to store `packages` (app bits), staged `droplets`, `buildpacks`, and cached app resources. Files are typically uploaded to the internal blobstore via the Cloud Controller local workers and downloaded by Diego when app instances are started.

#### When to scale
##### Key Metrics
* `system.cpu.user` is consistently high on the singleton-blobstore VM
* `system.disk.persistent.percent` is high indicates the blobstore is running out of room for additional files

##### Heuristics
* `cf push` is intermittently failing
* `cf push` average time is elevated
* App droplet downloads are timing out/failing on Diego

##### Logs
* `/var/vcap/sys/log/blobstore/internal_access.log`

#### How to scale
The internal WebDAV blobstore **cannot be scaled horizontally**, not even for availability purposes because of its reliance on the `singleton-blobstore` VM's persistent disk for file storage. For this reason, it is [not recommended](https://docs.cloudfoundry.org/concepts/high-availability.html#blobstore) for environments that require high availability. For these environments we recommend an [external blobstore](https://docs.cloudfoundry.org/deploying/common/cc-blobstore-config.html) be used.

It can be scaled vertically, however, so scaling up the number of CPUs or adding faster disk storage can improve the performance of the internal WebDAV blobstore if it is under high load.

High numbers of concurrent app container starts on Diego can cause stress on the blobstore. This typically can happen during upgrades in environments with a large number of apps and Diego cells. If vertically scaling the blobstore or improving its disk performance is not an option, [limiting the max number of concurrent app container starts](https://bosh.io/jobs/auctioneer?source=github.com/cloudfoundry/diego-release&version=2.29.0#p%3ddiego.auctioneer.starting_container_count_maximum) can be used as a mitigation.


---
### Footnotes

##### [1] BOSH CPU Metrics

BOSH system CPU metrics can be confusing. For example, running `bosh instances --vitals` will return CPU values that may look something like this:
```
CPU    CPU   CPU   CPU
Total  User  Sys   Wait
-      2.9%  3.2%  1.3%
```

The CPU User value corresponds with the `system.cpu.user` metric that was mentioned earlier and it is **scaled by the number of CPUs** so on a 4-core `api` VM, a `cloud_controller_ng` process that is using 100% of a core will appear as using `25%` in the `system.cpu.user` metric.

##### [2] Assessing Database Health
Since Cloud Controller supports both PostgreSQL and MySQL (and the concept of "bring your own database") it is difficult for us to provide absolute guidance on what a healthy database might look like. In general we've found that high database CPU utilization is a good indicator of scaling issues, but always defer to the documentation specific to your database.
