---
integration_title: Cloudfoundry
name: cloudfoundry
kind: integration
git_integration_title: cloud_foundry
newhlevel: true
updated_for_agent: 6.0
description: "Track the health of your Cloud Foundry VMs and the jobs they run."
is_public: true
public_title: Datadog-Cloud Foundry Integration
short_description: "Track the health of your Cloud Foundry VMs and the jobs they run."
categories:
- provisioning
- configuration & deployment
doc_link: https://docs.datadoghq.com/integrations/cloud_foundry/
ddtype: check
---

## Overview

Any Cloud Foundry deployment can send metrics and events to Datadog. The data helps you track the health and availability of all nodes in the deployment, monitor the jobs they run, collect metrics from the Loggregator Firehose, and more.  
Use this page to learn how to monitor your [Cloud Foundry application](#monitor-your-cloud-foundry-application)) and your [Cloud Foundry cluster](#monitoring-your-cloud-foundry-cluster).

## Monitor your Cloud Foundry application

Use **Datadog Cloud Foundry Buildpack** to monitor your Cloud Foundry application. This is a [decorator buildpack][1] for Cloud Foundry that installs a [Datadog DogStatsD binary][2] and [Datadog Trace Agent][3] in the container your app is running on.

### Setup

1. **Install the [Meta Buildpack][4]** to enable apps to use decorator buildpacks.

2. **Upload the Datadog Cloud Foundry Buildpack.**  
  Download the latest Datadog [build pack release][5] and upload it to your Cloud Foundry environment.

  ```shell
  cf create-buildpack datadog-cloudfoundry-buildpack ./datadog-cloudfoundry-buildpack-latest.zip 99 --enable

  cf push
  ```

### Configuration
**Set an API Key in your environment to enable the buildpack**:

```shell
# set the environment variable
cf set-env $YOUR_APP_NAME DD_API_KEY $YOUR_DATADOG_API_KEY
# restage the application to get it to pick up the new environment variable and use the buildpack
cf restage $YOUR_APP_NAME
```

### Build
To build this buildpack, edit the relevant files and run the `./build` script. If you want to upload it, run `./upload`.

### DogStatsD
DogStatsD setup is now complete. See [the documentation][6] for more information. We maintain [a list of DogStatsD libraries][7] compatible with a wide range of applications.

## Monitoring your Cloud Foundry cluster

There are three points of integration with Datadog, each of which achieves a different goal:

* **Datadog Agent BOSH release** — Install the Datadog Agent on every node in your deployment to track system, network, and disk metrics. Enable any other Agent checks you wish.
* **Datadog Firehose Nozzle** — Deploy one or more Datadog Firehose Nozzle jobs. The jobs tap into your deployment's Loggregator Firehose and send all non-container metrics to Datadog.
* **Datadog plugin for BOSH Health Monitor** — Configure your BOSH Director's Health Monitor to send heartbeats (as metrics) and alerts (as events) from each node's BOSH Agent to Datadog.

These integrations are meant for Cloud Foundry deployment administrators, not end users.

### Prerequisites

You need to have a working Cloud Foundry deployment and access to the BOSH Director that manages it. You also need BOSH CLI to deploy each integration. You may use either major version of the CLI—[v1][8] or [v2][9].

To configure the Datadog plugin for BOSH Health Monitor, you need access to the `state.json` (or similarly named) file that accurately reflects the current state of your BOSH Director. If you don't have such a file, you'll need to [create one][10].

### Install the Datadog Agent BOSH Release

Datadog provides tarballs of the Datadog Agent packaged as a BOSH release. You can upload the latest release to your BOSH Director and then easily install it on every node in your deployment as an [addon][11] (i.e. the same way a Director deploys the BOSH Agent to all nodes).

#### Upload Datadog's release to your BOSH Director

```
# BOSH CLI v1
bosh upload release https://cloudfoundry.datadoghq.com/datadog-agent/datadog-agent-boshrelease-latest.tgz

# BOSH CLI v2
bosh upload-release https://cloudfoundry.datadoghq.com/datadog-agent/datadog-agent-boshrelease-latest.tgz
```

If you'd like to create your own release, see the [Datadog Agent BOSH Release repository][12].

#### Configure the Agent as an addon in your BOSH Director

Add the following to your BOSH Director's runtime configuration file (e.g. `runtime.yml`):

```
---
releases:
  - name: datadog-agent
    version: <VERSION_YOU_UPLOADED> # specify the real version, i.e. x.y.z, not 'latest'

addons:
- name: datadog
  jobs:
  - name: dd-agent
    release: datadog-agent
  properties:
    dd:
      use_dogstatsd: yes
      dogstatsd_port: 18125               # Many CF deployments have a StatsD already on port 8125
      api_key: <YOUR_DATADOG_API_KEY>
      tags: ["cloudfoundry_deployment_1"] # any tags you wish
      generate_processes: true            # to enable the process check
```

To see which `datadog-agent` release version you uploaded earlier, run `bosh releases`.

If you don't have a local copy of the runtime configuration, create a `runtime.yml` with the current runtime configuration (`bosh runtime-config`) and add the above YAML to it. If the Director's runtime configuration is empty, create a new `runtime.yml` containing only the above YAML.

#### Enable extra Agent checks

For each extra Agent check you want to enable across your deployment, add its configuration under the `properties.dd.integrations` key, e.g.:

```
  properties:
    dd:
      integrations:
        directory:
          init_config: {}
          instances:
            directory: "."
        #process:
        #  init_config: {}
        #...
```

The configuration under each check name should look the same as if you were configuring the check in its own file in the Agent's conf.d directory.

You cannot configure a check for a subset of nodes in your deployment; everything you configure in `runtime.yml` will apply to every node.

To customize configuration for the default checks—system, network, disk, and ntp—see the [full list of configuration options][13] for the Datadog Agent BOSH release.

#### Sync the runtime configuration to the Director

```
# BOSH CLI v1
bosh update runtime-config runtime.yml

# BOSH CLI v2
bosh update-runtime-config runtime.yml
```

#### Redeploy your Cloud Foundry deployment

```
# BOSH CLI v1
bosh deployment yourDeploymentManifest.yml
bosh -n deploy

# BOSH CLI v2
bosh -n -d yourDeployment deploy yourDeploymentManifest.yml
```

Since runtime configuration applies globally, BOSH will redeploy every node in your deployment. If you have more than one deployment, redeploy those, too, to install the Datadog Agent everywhere.

#### Verify the Agent is installed everywhere

The easiest way to check that Agent installs were successful is to filter for them in the [Host map page][14] in Datadog. The Agent BOSH release tags each host with a generic `cloudfoundry` tag, so filter by that, and optionally group hosts by any tag you wish (e.g. `bosh_job`), as in the following screenshot:

{{< img src="integrations/cloud_foundry/cloud-foundry-host-map.png" alt="cloud-foundry-host-map" responsive="true" popup="true">}}

Click on any host to zoom in, then click **system** within its hexagon to make sure Datadog is receiving metrics for it:

{{< img src="integrations/cloud_foundry/cloud-foundry-host-map-detail.png" alt="cloud-foundry-host-map-detail" responsive="true" popup="true">}}

### Deploy the Datadog Firehose Nozzle

As with the Datadog Agent, Datadog provides a BOSH release of the Datadog Firehose Nozzle. After uploading the release to your Director, you can add the Nozzle to an existing deployment, or create a new deployment that only includes the Nozzle. The instructions below assume you're adding it to an existing Cloud Foundry deployment that has a working Loggregator Firehose.

#### Upload Datadog's release to your BOSH Director

```
# BOSH CLI v1
bosh upload release http://cloudfoundry.datadoghq.com/datadog-firehose-nozzle/datadog-firehose-nozzle-release-latest.tgz

# BOSH CLI v2
bosh upload-release http://cloudfoundry.datadoghq.com/datadog-firehose-nozzle/datadog-firehose-nozzle-release-latest.tgz
```

If you'd like to create your own release, see the [Datadog Firehose Nozzle release repository][15].

#### Configure a UAA client

In the manifest that contains your UAA configuration, add a new client for the Datadog Nozzle so the job(s) can access the Firehose:

```
uaa:
  clients:
    datadog-firehose-nozzle:
      access-token-validity: 1209600
      authorities: doppler.firehose
      authorized-grant-types: client_credentials
      override: true
      scope: doppler.firehose
      secret: <MAKE_IT_A_GOOD_ONE>
```

And redeploy the deployment to add the user.

#### Add Nozzle jobs

Configure one or more Nozzle jobs in your main Cloud Foundry deployment manifest (e.g. cf-manifest.yml):

```
jobs:
#- instances: 4
#  name: some_other_job
#  ...
- instances: 1            # add more instances if one job cannot keep up with the Firehose
  name: datadog_nozzle_z1
  networks:
    - name: cf1           # some network you've configured elsewhere in the manifest
  resource_pool: small_z1 # some resource_pool you've configured elsewhere in the manifest
  templates:
    - name: datadog-firehose-nozzle
      release: datadog-firehose-nozzle
  properties:
    datadog:
      api_key: <YOUR_DATADOG_API_KEY>
      api_url: https://api.datadoghq.com/api/v1/series
      flush_duration_seconds: 15 # seconds between flushes to Datadog. Default is 15.
    loggregator:
      # do NOT append '/firehose' or even a trailing slash to the URL; 'ws://<host>:<port>' will do
      traffic_controller_url: <LOGGREGATOR_URL> # e.g. ws://traffic-controller.your-cf-domain.com:8081
    nozzle:
      deployment: <DEPLOYMENT_NAME>    # tags each firehose metric with 'deployment:<DEPLOYMENT_NAME>'
      subscription_id: datadog-nozzle  # can be anything (firehose streams data evenly to all jobs using the same subscription_id)
      # disable_access_control: true   # for development only
      # insecure_ssl_skip_verify: true # for development only; enable if your UAA does not use a verifiable cert
    uaa:
      client: datadog-firehose-nozzle # client name you just configured
      client_secret: <SECRET_YOU_JUST_CONFIGURED>
      url: <UAA_URL> # e.g. https://uaa.your-cf-domain.com:8443
```

To see all available configuration options, check the [Datadog Firehose Nozzle repository][16].

In the same manifest, add the Datadog Nozzle release name and version:

```
releases:
# - name: some-other-release
#   version: x.y.z
# ...
  - name: datadog-firehose-nozzle
    version: $VERSION_YOU_UPLOADED # specify the real version, i.e. x.y.z, not 'latest'
```

To see which `datadog-firehose-nozzle` release version you uploaded earlier, run `bosh releases`.

#### Redeploy the deployment

```
# BOSH CLI v1
bosh deployment cf-manifest.yml
bosh -n deploy

# BOSH CLI v2
bosh -n -d cf-manifest deploy cf-manifest.yml
```

#### Verify the Nozzle is collecting

On the [Metrics explorer][17] page in Datadog, search for metrics beginning `cloudfoundry.nozzle`:

{{< img src="integrations/cloud_foundry/cloud-foundry-nozzle-metrics.png" alt="cloud-foundry-nozzle-metrics" responsive="true" popup="true">}}

### Configure the Datadog plugin for BOSH Health Monitor

The plugin is packaged with the BOSH Health Monitor, so if the Health Monitor is already installed and running on your BOSH Director, you just need to configure the plugin and redeploy the Director.

#### Add the configuration

In the manifest originally used to deploy the Director (e.g. bosh.yml), configure the Datadog plugin:

```
instance_groups:
- instances: 1
  properties:
    hm:
    # loglevel: info
    # resurrector: enabled
    #...other stuff
      datadog_enabled: true
      datadog:
        api_key: <YOUR_DATADOG_API_KEY>
        application_key: <YOUR_DATADOG_APP_KEY>
        pagerduty_service_name: "your_service_name" # optional
```

#### Redeploy the Director

BOSH cannot simply configure the plugin and restart the Health Monitor; it must destroy the Director and redeploy it as a new instance. To do this while retaining the Director's current disk and database, you MUST redeploy using a `state.json` (or similarly named) file that accurately reflects the current state of your Director. If you don't have such a file, follow the BOSH docs on [Deployment State][10] to create a basic `state.yml`.

You may use [bosh-init][18] or BOSH CLI v2 to redeploy the Director. If you use bosh-init, your state file must be named similarly to your manifest; for a manifest named `bosh.yml`, `bosh-init` expects a state file named `bosh-state.yml`.

```
# bosh-init (legacy)
bosh-init deploy bosh.yml

# BOSH CLI v2 - if your state file is not named bosh-state.yml, provide its name via --state
bosh create-env --state=your-state-file.json bosh.yml
```

#### Verify the plugin is working

On the [Metrics explorer][17] page in Datadog, search for metrics beginning `bosh.healthmonitor`:

{{< img src="integrations/cloud_foundry/cloud-foundry-bosh-hm-metrics.png" alt="cloud-foundry-bosh-hm-metrics" responsive="true" popup="true">}}

## Data Collected
### Events

The BOSH Health Monitor Datadog plugin emits an event to Datadog for any alert it receives from your deployment's BOSH Agents. Read the [BOSH Health Monitor docs][19] to see what kinds of alerts might show up in your Datadog event stream.

The BOSH Agent sets a severity for each alert it generates, and the Datadog Health Monitor plugin uses that severity to prioritize the event it emits. Alerts with an Error, Critical, or Alert severity become Normal priority events in Datadog. Alerts with any other severity become Low priority events.

### Metrics

The following metrics are sent by the Datadog Firehose Nozzle (`cloudfoundry.nozzle`) and the BOSH Health Monitor Datadog plugin (`bosh.healthmonitor`). The Datadog Agent release does not send any special metrics of its own, just the usual metrics from any Agent checks you configure in the Director runtime config (and, by default, [system](/integrations/system/#metrics), [network][20], [disk][21], and [ntp][22] metrics).

The Datadog Firehose Nozzle only collects CounterEvents (as metrics, not events) and ValueMetrics; it ignores LogMessages, Errors, and ContainerMetrics.

{{< get-metrics-from-git "cloud_foundry">}}

[1]: https://github.com/cf-platform-eng/meta-buildpack/blob/master/README.md#decorators
[2]: https://docs.datadoghq.com/developers/dogstatsd/
[3]: https://docs.datadoghq.com/tracing/
[4]: https://github.com/cf-platform-eng/meta-buildpack#how-to-install-the-meta-buildpack
[5]: https://cloudfoundry.datadoghq.com/datadog-cloudfoundry-buildpack/datadog-cloudfoundry-buildpack-latest.zip
[6]: /developers/dogstatsd
[7]: https://docs.datadoghq.com/libraries/
[8]: https://bosh.io/docs/bosh-cli.html
[9]: https://bosh.io/docs/cli-v2.html#install
[10]: https://bosh.io/docs/cli-envs.html#deployment-state
[11]: https://bosh.io/docs/runtime-config.html#addons
[12]: https://github.com/DataDog/datadog-agent-boshrelease
[13]: https://github.com/DataDog/datadog-agent-boshrelease/blob/master/jobs/dd-agent/spec
[14]: https://app.datadoghq.com/graphing/infrastructure/hostmap
[15]: https://github.com/DataDog/datadog-firehose-nozzle-release
[16]: https://github.com/DataDog/datadog-firehose-nozzle-release/blob/master/jobs/datadog-firehose-nozzle/spec
[17]: https://app.datadoghq.com/metric/explorer
[18]: https://bosh.io/docs/
[19]: https://bosh.io/docs/monitoring.html
[20]: /integrations/network/#metrics
[21]: /integrations/disk/#metrics
[22]: /integrations/ntp/#metrics
