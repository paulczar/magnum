A Mesos cluster with Heat
=========================

These [Heat][] templates will deploy a [Mesos][] cluster.

[heat]: https://wiki.openstack.org/wiki/Heat
[mesos]: http://mesos.apache.org/

## Requirements

### OpenStack

These templates will work with the Kilo version of Heat.

### Guest image

These templates will work with Ubuntu 14.04 base image with the following
middleware pre-installed:

- `docker`
- `zookeeper`
- `mesos`
- `marathon`

### Building an image

If you do not have a suitable image you can build one easily using one of two methods:

#### Disk Image Builder

See [elements/README.md](elements/README.md) for instructions.

#### Docker

Use the provided [Dockerfile](./Dockerfile) to build the image ( it uses the same DIB scripts as above).  The resultant image will be saved as `/tmp/ubuntu-mesos.qcow2`

```
$ docker build -t magnum/mesos-builder .
$ docker run -v /tmp:/output --rm -ti --privileged magnum/mesos_build
...
Image file /output/ubuntu-mesos.qcow2 created...
$ ls /tmp/ubuntu-mesos.qcow2

```


## Creating the stack

Creating an environment file `local.yaml` with parameters specific to
your environment:

    parameters:
      ssh_key_name: testkey
      external_network: public
      dns_nameserver: 8.8.8.8
      server_image: ubuntu-mesos

And then create the stack, referencing that environment file:

    heat stack-create -f mesoscluster.yaml -e local.yaml my-mesos-cluster

You must provide value for:

- `ssh_key_name`

You can optionally provide values for:

- `server_image` (ubuntu-mesos if not provided)
- `external_network` (public if not provided)
- `dns_nameserver` (8.8.8.8 if not provided)

## Interacting with Mesos

You can get the ip address of the Mesos master using the `heat
output-show` command:

    $ heat output-show my-mesos-cluster mesos_master
    "192.168.200.86"

You can ssh into that server as the `ubuntu` user:

    $ ssh ubuntu@192.168.200.86

You can log into your slaves using the `ubuntu` user as well.  You
can get a list of slaves addresses by running:

    $ heat output-show my-mesos-cluster mesos_slaves
    [
      "192.168.200.182"
    ]

## Testing

Docker containers can be deployed via Marathon's REST API.
Marathon is a mesos framework for long running applications.

We can 'post' a JSON app description to http://<master>:8080/apps to deploy
a Docker container.

    $ cat > app.json << END
    {
      "container": {
        "type": "DOCKER",
        "docker": {
          "image": "libmesos/ubuntu"
        }
      },
      "id": "ubuntu",
      "instances": 1,
      "cpus": 0.5,
      "mem": 512,
      "uris": [],
      "cmd": "while sleep 10; do date -u +%T; done"
    }
    END
    $ MASTER_IP=$(heat output-show my-mesos-cluster mesos_master | tr -d '"')
    $ curl -X POST -H "Content-Type: application/json" \
        http://${MASTER_IP}:8080/v2/apps -d@app.json

Using the Marathon web console (at http://<master>:8080/), you will see the
application you created.

## License

Copyright 2015 Huawei Technologies Co.,LTD.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use these files except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
