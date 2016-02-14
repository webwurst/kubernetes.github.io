---
title: "Adding a Kubernetes worker node via Docker."
---
These instructions are very similar to the master set-up above, but they are duplicated for clarity.
You need to repeat these instructions for each node you want to join the cluster.
We will assume that the IP address of this node is `${NODE_IP}` and you have the IP address of the master in `${MASTER_IP}` that you created in the [master instructions](master).

For each worker node, there are three steps:
   * [Set up `flanneld` on the worker node](#set-up-flanneld-on-the-worker-node)
   * [Start Kubernetes on the worker node](#start-kubernetes-on-the-worker-node)
   * [Add the worker to the cluster](#add-the-node-to-the-cluster)

### Set up Flanneld on the worker node

As before, the Flannel daemon is going to provide network connectivity.

_Note_:
There is a [bug](https://github.com/docker/docker/issues/14106) in Docker 1.7.0 that prevents this from working correctly.
Please install Docker 1.6.2 or wait for Docker 1.7.1.


#### Set up a bootstrap docker

As previously, we need a second instance of the Docker daemon running to bootstrap the flannel networking.

Run:

{% highlight sh %}

sudo sh -c 'docker -d -H unix:///var/run/docker-bootstrap.sock -p /var/run/docker-bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker-bootstrap 2> /var/log/docker-bootstrap.log 1> /dev/null &'

{% endhighlight %}

_Important Note_:
If you are running this on a long running system, rather than experimenting, you should run the bootstrap Docker instance under something like SysV init, upstart or systemd so that it is restarted
across reboots and failures.

#### Bring down Docker

To re-configure Docker to use flannel, we need to take docker down, run flannel and then restart Docker.

Turning down Docker is system dependent, it may be:

{% highlight sh %}

sudo /etc/init.d/docker stop

{% endhighlight %}

or

{% highlight sh %}

sudo systemctl stop docker

{% endhighlight %}

or it may be something else.

#### Run flannel

Now run flanneld itself, this call is slightly different from the above, since we point it at the etcd instance on the master.

{% highlight sh %}

sudo docker -H unix:///var/run/docker-bootstrap.sock run -d --net=host --privileged -v /dev/net:/dev/net quay.io/coreos/flannel:0.5.0 /opt/bin/flanneld --etcd-endpoints=http://${MASTER_IP}:4001

{% endhighlight %}

The previous command should have printed a really long hash, copy this hash.

Now get the subnet settings from flannel:

{% highlight sh %}

sudo docker -H unix:///var/run/docker-bootstrap.sock exec <really-long-hash-from-above-here> cat /run/flannel/subnet.env

{% endhighlight %}


#### Edit the docker configuration

You now need to edit the docker configuration to activate new flags.  Again, this is system specific.

This may be in `/etc/default/docker` or `/etc/systemd/service/docker.service` or it may be elsewhere.

Regardless, you need to add the following to the docker command line:

{% highlight sh %}

--bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}

{% endhighlight %}

#### Remove the existing Docker bridge

Docker creates a bridge named `docker0` by default.  You need to remove this:

{% highlight sh %}

sudo /sbin/ifconfig docker0 down
sudo brctl delbr docker0

{% endhighlight %}

You may need to install the `bridge-utils` package for the `brctl` binary.

#### Restart Docker

Again this is system dependent, it may be:

{% highlight sh %}

sudo /etc/init.d/docker start

{% endhighlight %}

it may be:

{% highlight sh %}

systemctl start docker

{% endhighlight %}

### Start Kubernetes on the worker node

#### Run the kubelet

Again this is similar to the above, but the `--api-servers` now points to the master we set up in the beginning.

{% highlight sh %}

sudo docker run \
    --volume=/:/rootfs:ro \
    --volume=/sys:/sys:ro \
    --volume=/dev:/dev \
    --volume=/var/lib/docker/:/var/lib/docker:rw \
    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
    --volume=/var/run:/var/run:rw \
    --net=host \
    --privileged=true \
    --pid=host \ 
    -d \
    gcr.io/google_containers/hyperkube:v1.0.1 /hyperkube kubelet --api-servers=http://${MASTER_IP}:8080 --v=2 --address=0.0.0.0 --enable-server --hostname-override=$(hostname -i) --cluster-dns=10.0.0.10 --cluster-domain=cluster.local

{% endhighlight %}

#### Run the service proxy

The service proxy provides load-balancing between groups of containers defined by Kubernetes `Services`

{% highlight sh %}

sudo docker run -d --net=host --privileged gcr.io/google_containers/hyperkube:v1.0.1 /hyperkube proxy --master=http://${MASTER_IP}:8080 --v=2

{% endhighlight %}

### Next steps

Move on to [testing your cluster](testing) or [add another node](#adding-a-kubernetes-worker-node-via-docker)


