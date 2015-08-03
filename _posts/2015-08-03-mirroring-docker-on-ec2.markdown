---
layout: post
title:  "Mirroring Docker on EC2"
date:   2015-08-03 14:07:00
categories: freshbooks docker
---

I mentioned in a [previous post]({% post_url 2015-08-01-how-freshbooks-scales-capybara %}) that we're looking at speeding up how long it takes for our build/test cycle takes. We have some more substantial improvements in the works, but found a quick win by simply mirroring our images that are hosted on Docker Hub. The mirror lives on the internal EC2 network.

To create the mirror machine we started by freezing an AMI based on CoreOS that runs the `registry:latest` image from the Docker library. The machine is configured via a very simple `cloud-init` file:

{% highlight yaml %}
#cloud-config
coreos:
  update:
    reboot-strategy: off
  units:
    - name: etcd.service
      mask: true
    - name: etcd2.service
      mask: true
    - name: fleet.service
      mask: true
    - name: locksmith.service
      mask: true
    - name: flannel.service
      mask: true
    - name: docker.service
      command: start
    - name: docker-mirror.service
      command: restart
      enable: true
      content: |
        [Unit]
        Description=Docker Mirror
        After=docker.service
        Requires=docker.service

        [Service]
        Restart=always
        RestartSec=10s
        ExecStartPre=-/usr/bin/docker kill docker-mirror
        ExecStartPre=-/usr/bin/docker rm docker-mirror
        ExecStartPre=/usr/bin/docker pull registry:latest
        ExecStart=/usr/bin/bash -c \
          "/usr/bin/docker run --name docker-mirror -h `hostname` \
          -p 5000:5000 \
          -e STANDALONE=false \
          -e MIRROR_SOURCE=https://registry-1.docker.io \
          -e MIRROR_SOURCE_INDEX=https://index.docker.io \
          registry:latest"
        ExecStop=/usr/bin/docker stop docker-mirror

        [Install]
        WantedBy=multi-user.target
{% endhighlight %}

The above configuration simple configured a CoreOS machine to _not_ run its typical `etcd`, `fleet`, `locksmith` or `flannel` service -- we don't need them. Aditionally, we make sure Docker is running and then specify a `docker-mirror` service that will start on boot and run the registry mirror.

One thing to note is that we _don't_ current mount the data directory outside the container itself. We treat the data as ephemeral since it can always be refetched from the upstream Docker Hub registry.

We put the AMI inside an AWS Launch Configuration so that we can add it to an autoscaling group. We use autoscaling in this case as a quick hack to make sure that there is always `n` of the machines actually running -- though right now we only run a single mirror instance so the autoscaling group's "preferred size" is 1.

We've also created an "internal" ELB that takes external port 80 traffic and routes it to the mirror's port 5000. The health check is currently configured to use an `HTTP` check for the `/` path, which in the latest registry will return a `200` HTTP code.

Normally we'd use Route 53 to create an ALIAS record to the ELB so that we can just refer to the mirror as `docker-mirror.local` internally, but we actually have our own DNS solution that we run inside the test VPC, so we needed to add a `CNAME` record to the ELB there instead. Regardless, any machine inside the VPC can now communicate with `http://docker-mirror.local` and reach our mirrored registry.

The last step is to teach our test (`shortstack`) machines to prefer using our internal mirror instead of the public registry. Shortstacks are running CentOS 7, so their docker daemon configuration is located at `/etc/sysconfig/docker`. We use the following configuration:

{% highlight bash %}
# /etc/sysconfig/docker

OPTIONS='--selinux-enabled --registry-mirror=http://docker-mirror.local'
DOCKER_CERT_PATH=/etc/docker
{% endhighlight %}

A reboot of the docker daemon (or for us -- freezing a new `shortstack` AMI and churning the test infrastructure!) and you should see faster (subsequent!) pulls.

If you'd like to see how the docker mirror is behaving, you can check out the logs by SSH'ing into the CoreOS docker mirror and tail'ing the logs:

{% highlight bash %}
docker logs -f docker-mirror
{% endhighlight %}

Personally, I'm excited to see what comes out of `v2` of the docker registry project; we're already looking at running a completely internal registry backed by S3. Unfortunatly, the `registry:2` image currently doesn't support mirroring.