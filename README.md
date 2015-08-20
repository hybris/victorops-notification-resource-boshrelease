BOSH release for victorops-notification-resource
=======================

Background
----------

### What is victorops-notification-resource?

Send alerts to your [VictorOps](https://victorops.com) account from Concourse:

Usage
-----

To use this bosh release, first upload it to the BOSH/bosh-lite that is running Concourse:

```
bosh upload release https://github.com/hybris/victorops-notification-resource-boshrelease
```

Next, update your Concourse deployment manifest to add the resource.

Add the `victorops-notification-resource` release to the list:

```yaml
releases:
  - name: concourse
    version: latest
  - name: garden-linux
    version: latest
  - name: victorops-notification-resource
    version: latest
```

Into the `worker` job, add the `{release: victorops-notification-resource, name: install}` job template that will install the package:

```yaml
jobs:
- name: worker
  templates:
    ...
    - {release: victorops-notification-resource, name: install}
```

The final change is to explicitly list all the resource types (they are implicit) and add the `victorops-notification-resource` package to the list:

```yaml
jobs:
- name: worker
  ...
  properties:
    groundcrew:
      resource_types:
      ...
      - type: victorops-notification
        image: /var/vcap/packages/victorops-notification-resource
```

Note that it is the latter two lines that are specific to this BOSH release:

```yaml
- type: victorops-notification
  image: /var/vcap/packages/victorops-notification-resource
```

The former lines should be obtained from the Concourse BOSH release, not the documentation above which might be out of date. Use https://github.com/concourse/concourse/blob/master/jobs/groundcrew/spec#L69-L96

And `bosh deploy` your Concourse manifest.

Usage
-----

An example mini-pipeline that would send an alert:

```yaml
---
jobs:
- name: hello-world
  plan:
    - task: say-hello
      config:
        platform: linux
        image: "docker:///busybox"
        run:
          path: echo
          args: ["Hello, world!"]
      on_success:
        put: vo-alert
        params:
          type: RECOVERY
          entity: "concourse/jobs/hello-world/say-hello"
          message: "build success"
      on_failure:
        put: vo-alert
        params:
          type: CRITICAL
          entity: "concourse/jobs/hello-world/say-hello"
          message: "build failed"

resources:
- name: vo-alert
  type: victorops-notification
  source:
    url: https://alert.victorops.com/integrations/generic/20131114/alert/<API_KEY>/<ROUTING_KEY>
```

Setup pipeline in Concourse
---------------------------

```
fly -t snw c -c pipeline.yml --vars-from credentials.yml victorops-notification-resource-boshrelease
```
