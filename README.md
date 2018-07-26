# Cinderlib CSI driver

[![Docker build status](https://img.shields.io/docker/build/akrog/cinderlib-csi.svg)](https://hub.docker.com/r/akrog/cinderlib-csi/) [![Docker build](https://img.shields.io/docker/automated/akrog/cinderlib-csi.svg)](https://hub.docker.com/r/akrog/cinderlib-csi/builds/) [![PyPi](https://img.shields.io/pypi/v/cinderlib_csi.svg)](https://pypi.python.org/pypi/cinderlib_csi) [![PyVersion](https://img.shields.io/pypi/pyversions/cinderlib_csi.svg)](https://pypi.python.org/pypi/cinderlib_csi) [![License](https://img.shields.io/:license-apache-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)

**The project has been renamed to Ember-CSI, so please go to [the new repository](https://github.com/akrog/ember-csi).**

CSI Python driver that leverages all Cinder drivers to provide block volumes without needing to run any additional service, such as RabbitMQ, MariaDB, Cinder-API, Cinder-Scheduler, or Cinder-Volume.

Current code is is a **Proof of Concept** only compatible with Cinder OSP-12/Pike release.

* Free software: Apache Software License 2.0
* Documentation: Pending


## Features

This CSI driver is up to date with latest CSI specs including the [new snapshots feature](https://github.com/container-storage-interface/spec/pull/224) recently introduced.

Currently supported features are:

- Create block volume
- Creating snapshots
- Creating a block volume from a snapshot
- Delete block volume
- Deleting snapshots
- Listing volumes with pagination
- Listing snapshots with pagination
- Attaching volumes
- Detaching volumes
- Reporting storage capacity
- Probing the node
- Retrieving the plugin info


## Runtime Dependencies

This driver requires that Cinder v11.0 (OSP-12/Pike) is already installed in the system, how this is accomplished is left to the installer, as there are multiple ways this can be accomplished:

- From OSP repositories
- From RDO repositories
- From github
- From other repositories

Any other basic requirement is already handled by `cinderlib-csi` when installing from PyPi.

Besides the basic dependencies there are also some drivers that have additional requirements that must be met for proper operation of the driver and/or attachment/detachment operations, just like in Cinder.

Some of these Python dependencies for the Controller servicer are:

- DRBD: dbus and drbdmanage
- HPE 3PAR: python-3parclient
- Kaminario: krest
- Pure: purestorage
- Dell EMC VMAX, IBM DS8K: pyOpenSSL
- HPE Lefthad: python-lefthandclient
- Fujitsu Eternus DX: pywbem
- IBM XIV: pyxcli
- RBD: rados and rbd
- Dell EMC VNX: storops
- Violin: vmemclient
- INFINIDAT: infinisdk, capacity, infy.dtypes.wwn, infi.dtypes.iqn

Other backends may also require additional packages, for example LVM on CentOS/RHEL requires the `targetcli` package, so please check with your hardware vendor.

Besides the Controller requirements there are usually requirements for the Node servicer needed to handle the attaching and detaching of volumes to the node based on the connection used to access the storage.  For example:

- iSCSI: iscsi-initiator-tools and device-mapper-multipath
- RBD/Ceph: ceph-common package


## Installation

First we need to install the Cinder Python package, for example to install from RDO on CentOS:

```
    $ sudo yum install -y centos-release-openstack-pike
    $ sudo yum install -y openstack-cinder python-pip
```


Then we just need to install the `cinderlib-csi` package:

```
    $ sudo pip install cinderlib-csi
```


Now we should install any additional package required by our backend.

For iSCSI backends we'll want to install:

```
    $ sudo yum install iscsi-initiator-utils
    $ sudo yum install device-mapper-multipath
    $ sudo mpathconf --enable --with_multipathd y --user_friendly_names n --find_multipaths y
```


For RBD we'll also need a specific package:

```
    $ sudo yum install ceph-common
```


## Configuration

The CSI driver is configured via environmental variables, any value that doesn't have a default is a required value.

| Name                       | Role       | Description                                                   | Default                                                                                                  | Example                                                                                                                                                                                                                                 |
| -------------------------- | ---------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CSI_ENDPOINT`             | all        | IP and port to bind the service                               | [::]:50051                                                                                               | 192.168.1.22:50050                                                                                                                                                                                                                      |
| `CSI_MODE`                 | all        | Role the service should perform: controller, node, all        | all                                                                                                      | controller                                                                                                                                                                                                                              |
| `X_CSI_STORAGE_NW_IP`      | node       | IP address in the Node used to connect to the storage         | IP resolved from Node's fqdn                                                                             | 192.168.1.22                                                                                                                                                                                                                            |
| `X_CSI_NODE_ID`            | node       | ID used by this node to identify itself to the controller     | Node's fqdn                                                                                              | csi_test_node                                                                                                                                                                                                                           |
| `X_CSI_PERSISTENCE_CONFIG` | all        | Configuration of the `cinderlib` metadata persistence plugin. | {'storage': 'db', 'connection': 'sqlite:///db.sqlite'}                                                   | {'storage': 'db', 'connection': 'mysql+pymysql://root:stackdb@192.168.1.1/cinder?charset=utf8'}                                                                                                                                         |
| `X_CSI_CINDERLIB_CONFIG`   | all        | Global `cinderlib` configuration                              | {'project_id': 'com.redhat.cinderlib-csi', 'user_id': 'com.redhat.cinderlib-csi', 'root_helper': 'sudo'} | {"project_id":"com.redhat.cinderlib-csi","user_id":"com.redhat.cinderlib-csi","root_helper":"sudo"}                                                                                                                                     |
| `X_CSI_BACKEND_CONFIG`     | controller | Driver configuration                                          |                                                                                                          | {"volume_backend_name": "rbd", "volume_driver": "cinder.volume.drivers.rbd.RBDDriver", "rbd_user": "cinder", "rbd_pool": "volumes", "rbd_ceph_conf": "/etc/ceph/ceph.conf", "rbd_keyring_conf": "/etc/ceph/ceph.client.cinder.keyring"} |
| `X_CSI_DEFAULT_MOUNT_FS`   | node       | Default mount filesystem when missing in publish calls        | ext4                                                                                                     | btrfs                                                                                                                                                                                                                                   |
| `X_CSI_DEBUG_MODE`         | all        | Debug mode (rpdb, pdb) to use. Disabled by default.           |                                                                                                          | rpdb                                                                                                                                                                                                                                    |
| `X_CSI_ABORT_DUPLICATES`   | all        | If we want to abort or queue (default) duplicated requests.   | false                                                                                                    | true                                                                                                                                                                                                                                    |

The only role that has been tested at the moment is the default one, where Controller and Node servicer are executed in the same service (`CSI_MODE=all`), and other modes are expected to have issues at the moment.


## Starting the plugin

Once we have installed `cinderlib-csi` and required dependencies (for the backend and for the connection type) we just have to run the `cinderlib-csi` service with a user that can do passwordless sudo:

```
    $ cinderlib-csi
```


## Testing the plugin

There are several examples of running the CSI cinderlib driver in the `examples` directory both for a baremetal deployment and a containerized version of the driver.

In all cases we have to run the plugin first before we can test it, and for that we have to check the configuration provided as a test before starting the plugin.  By default all examples run the service on port 50051.


### Baremetal

For example to test with the LVM driver on our development environment we can just run the following commands from the root of the `cinderlib-csi` project:

*Note*: The iscsi IP addresses are auto-assigned in the [lvm](examples/baremetal/lvm) env file. You may change these IP addresses if desired:

```
    $ cd tmp
    $ sudo dd if=/dev/zero of=cinder-volumes bs=1048576 seek=22527 count=1
    $ lodevice=`sudo losetup --show -f ./cinder-volumes`
    $ sudo pvcreate $lodevice
    $ sudo vgcreate cinder-volumes $lodevice
    $ sudo vgscan --cache
    $ cd ../examples/baremetal
    $ ./run.sh lvm
    py27 develop-inst-nodeps: /home/geguileo/code/reuse-cinder-drivers/cinderlib-csi
    py27 installed: ...
    ___ summary ___
      py27: skipped tests
      congratulations :)
    Starting cinderlib CSI v0.0.2 (cinderlib: v0.2.1, cinder: v11.1.2.dev5, CSI spec: v0.2.0)
    Supported filesystems are: fat, ext4dev, vfat, ext3, ext2, msdos, ext4, hfsplus, cramfs, xfs, ntfs, minix, btrfs
    Running backend LVMVolumeDriver v3.0.0
    Debugging is OFF
    Now serving on [::]:50051...
```


There is also an example of testing a Ceph cluster using a user called "cinder" and the "volumes" pool.  For the Ceph/RBD backend, due to a limitation in Cinder, we need to have both the credentials and the configuration in `/etc/ceph` for it to work:

```
    $ cd examples/baremetal
    $ ./run.sh rbd
    Starting cinderlib CSI v0.0.2 (cinderlib: v0.2.1, cinder: v11.1.2.dev5, CSI spec: v0.2.0)
    Supported filesystems are: fat, ext4dev, vfat, ext3, ext2, msdos, ext4, hfsplus, cramfs, xfs, ntfs, minix, btrfs
    Running backend LVMVolumeDriver v3.0.0
    Debugging is OFF
    Now serving on [::]:50051...
```


There is also an XtremIO example that only requires the iSCSI connection packages.


### Containerized

There is a sample `Dockerfile` included in the project that has been used to create the `akrog/cinderlib-csi` container available in the docker hub.

There are two bash scripts, one for each example, that will run the CSI driver on a container, be aware that the container needs to run as privileged to mount the volumes.

For the RBD example we need to copy our "ceph.conf" and "ceph.client.cinder.keyring" files, assuming we are using the "cinder" user into the example/docker directory replacing the existing ones:

```
    $ cd examples/docker
    $ ./rbd.sh
    Starting cinderlib CSI v0.0.2 (cinderlib: v0.2.1, cinder: v11.1.0, CSI spec: v0.2.0)
    Supported filesystems are: cramfs, minix, ext3, ext2, ext4, xfs, btrfs
    Running backend LVMVolumeDriver v3.0.0
    Debugging is ON with rpdb
    Now serving on [::]:50051...
```

### CSC

Now that we have the service running we can use the [CSC tool](https://github.com/rexray/gocsi/tree/master/csc) to run commands simulating the Container Orchestration system.

Due to the recent changes in the CSI spec not all commands are available yet, so you won't be able to test the snapshot commands.

Checking the plugin info:

```
    $ csc identity plugin-info -e tcp://127.0.0.1:50051
    "com.redhat.cinderlib-csi"      "0.0.2" "cinder-driver"="RBDDriver"     "cinder-driver-supported"="True"        "cinder-driver-version"="1.2.0" "cinder-version"="11.1.0"       "cinderlib-version"="0.2.1"     "persistence"="DBPersistence"
```

Checking the node id:

```
    $ csc node get-id -e tcp://127.0.0.1:50051
    localhost.localdomain

    $ hostname -f
    localhost.localdomain
```

Checking the current backend capacity:

```
    $ csc controller get-capacity -e tcp://127.0.0.1:50051
    24202140712
```

Creating a volume:

```
    $ csc controller create-volume --cap SINGLE_NODE_WRITER,block --req-bytes 2147483648 disk -e tcp://127.0.0.1:50051
    "5ee5fd7c-45cd-44cf-af7b-06081f680f2c"  2147483648
```

Listing volumes:

```
    $ csc controller list-volumes -e tcp://127.0.0.1:50051
    "5ee5fd7c-45cd-44cf-af7b-06081f680f2c"  2147483648
```

Store the volume id for all the following calls:

```
    $ vol_id=`csc controller list-volumes -e tcp://127.0.0.1:50051|awk '{ print gensub("\"","","g",$1)}'`
```

Attaching the volume to `tmp/mnt/publish` on baremetal as a block device:

```
    $ touch tmp/mnt/{staging,publish}

    $ csc controller publish --cap SINGLE_NODE_WRITER,block --node-id `hostname -f` $vol_id -e tcp://127.0.0.1:50051
    "5ee5fd7c-45cd-44cf-af7b-06081f680f2c"  "connection_info"="{\"connector\": {\"initiator\": \"iqn.1994-05.com.redhat:aa532823bac9\", \"ip\": \"127.0.0.1\", \"platform\": \"x86_64\", \"host\": \"localhost.localdomain\", \"do_local_attach\": false, \"os_type\": \"linux2\", \"multipath\": false}, \"conn\": {\"driver_volume_type\": \"rbd\", \"data\": {\"secret_uuid\": null, \"volume_id\": \"5ee5fd7c-45cd-44cf-af7b-06081f680f2c\", \"auth_username\": \"cinder\", \"secret_type\": \"ceph\", \"name\": \"volumes/volume-5ee5fd7c-45cd-44cf-af7b-06081f680f2c\", \"discard\": true, \"keyring\": \"[client.cinder]\\n\\tkey = AQCQPetaof03IxAAoHZJD6kGxiMQfLdn3QzdlQ==\\n\", \"cluster_name\": \"ceph\", \"hosts\": [\"192.168.1.22\"], \"auth_enabled\": true, \"ports\": [\"6789\"]}}}"

    $ csc node stage --pub-info connection_info="irrelevant" --cap SINGLE_NODE_WRITER,block --staging-target-path `realpath tmp/mnt/staging` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc node publish --cap SINGLE_NODE_WRITER,block --pub-info connection_info="irrelevant" --staging-target-path `realpath tmp/mnt/staging` --target-path `realpath tmp/mnt/publish` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c
```

Attaching the volume to `tmp/mnt/publish` on container as a block device:

```
    $ touch tmp/mnt/{staging,publish}

    $ csc controller publish --cap SINGLE_NODE_WRITER,block --node-id `hostname -f` $vol_id -e tcp://127.0.0.1:50051
    "5ee5fd7c-45cd-44cf-af7b-06081f680f2c"  "connection_info"="{\"connector\": {\"initiator\": \"iqn.1994-05.com.redhat:aa532823bac9\", \"ip\": \"127.0.0.1\", \"platform\": \"x86_64\", \"host\": \"localhost.localdomain\", \"do_local_attach\": false, \"os_type\": \"linux2\", \"multipath\": false}, \"conn\": {\"driver_volume_type\": \"rbd\", \"data\": {\"secret_uuid\": null, \"volume_id\": \"5ee5fd7c-45cd-44cf-af7b-06081f680f2c\", \"auth_username\": \"cinder\", \"secret_type\": \"ceph\", \"name\": \"volumes/volume-5ee5fd7c-45cd-44cf-af7b-06081f680f2c\", \"discard\": true, \"keyring\": \"[client.cinder]\\n\\tkey = AQCQPetaof03IxAAoHZJD6kGxiMQfLdn3QzdlQ==\\n\", \"cluster_name\": \"ceph\", \"hosts\": [\"192.168.1.22\"], \"auth_enabled\": true, \"ports\": [\"6789\"]}}}"

    $ csc node stage --pub-info connection_info="irrelevant" --cap SINGLE_NODE_WRITER,block --staging-target-path /mnt/staging $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc node publish --cap SINGLE_NODE_WRITER,block --pub-info connection_info="irrelevant" --staging-target-path /mnt/staging --target-path /mnt/publish $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c
```


Detaching the volume on baremetal:

```
    $ csc node unpublish --target-path `realpath tmp/mnt/publish` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc node unstage --staging-target-path `realpath tmp/mnt/staging` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc controller unpublish --node-id `hostname -f` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c
```

Detaching the volume on container:

```
    $ csc node unpublish --target-path /mnt/publish $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc node unstage --staging-target-path /tmp/mnt/staging $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc controller unpublish --node-id `hostname -f` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c
```

Deleting the volume:

```
    $ csc controller delete-volume $vol_id -e tcp://127.0.0.1:50051
```

If we want to use the mount interface instead of the block one, we can also do it making sure we create directories instead of files and replacing the `block` word with `mount,ext4` if we want an `ext4` filesystem.

For example these would be the commands for the baremetal attach:

```
    $ mkdir tmp/mnt/{staging_dir,publish_dir}

    $ csc controller publish --cap SINGLE_NODE_WRITER,mount,ext4 --node-id `hostname -f` $vol_id -e tcp://127.0.0.1:50051

    $ csc node stage --pub-info connection_info="irrelevant" --cap SINGLE_NODE_WRITER,mount,ext4 --staging-target-path `realpath tmp/mnt/staging_dir` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c

    $ csc node publish --pub-info connection_info="irrelevant" --cap SINGLE_NODE_WRITER,mount,ext4 -staging-target-path `realpath tmp/mnt/staging_dir` --target-path `realpath tmp/mnt/publish_dir` $vol_id -e tcp://127.0.0.1:50051
    5ee5fd7c-45cd-44cf-af7b-06081f680f2c
```


## Capable operational modes

The CSI spec defines a set of `AccessModes` that CSI drivers can support, such as single writer, single reader, multiple writers, single writer and multiple readers.

This CSI driver currently only supports `SINGLE_MODE_WRITER`, although it will also succeed with the `SINGLE_MODE_READER_ONLY` mode and mount it as read/write.


## Debugging

The first tool for debugging is the log that displays detailed information on the driver code used by *cinderlib-CSI*.  We can enable INFO or DEBUG logs using the `X_CSI_CINDERLIB_CONFIG` environmental variable.

To enable logs, defaulting to INFO level, we must set the `disable_logs` key to `false`.  If we want them at DEBUG levels, we also need to set `debug` to `true`.

For baremetal, enablig DEBUG log levels can be done like this:

```
    export X_CSI_CINDERLIB_CONFIG={"project_id":"com.redhat.cinderlib-csi","user_id":"com.redhat.cinderlib-csi","root_helper":"sudo","disable_logs":false,"debug":true}

```

For containers we can just add the environmental variable to a file and import into our run using `--env-file` or adding it to our command line with `-e`.

In both cases it should not have the `export` command:

```
    X_CSI_CINDERLIB_CONFIG={"project_id":"com.redhat.cinderlib-csi","user_id":"com.redhat.cinderlib-csi","root_helper":"sudo","disable_logs":false,"debug":true}

```

Besides this basic debugging level, the cinderlib-CSI plugin also supports live debugging when run in the baremetal and when running as a container.

There are two mechanisms that can be used to debug the driver: with `pdb`, and with `rpdb`.

The difference between them is that `pdb` works with stdin and stdout, whereas `rpdb` opens port 4444 to accept remote connections for debugging.

Debugging the cinderlib-CSI plugin requires enabling debugging on the plugin before starting it, and then one it is running we have to turn it on.

Enabling debugging is done using the `X_CSI_DEBUG_MODE` environmental variable.  Setting it to `pdb` or `rpdb` will enable debugging.  The plugin has this feature disabled by default, but our *latest* and *master* containers have it enabled by default with `rpdb`.

Once we have the plugin running with the debugging enable (we can see it in the start message) we can turn it on and off using the `SIGUSR1` signal, and the service will output the change with a *Debugging is ON* or *Debugging is OFF* message.

After turning it *ON* the plugin will stop for debugging on the next GRPC request.  Going into interactive mode if using `pdb` or opening port 4444 if using `rpdb`.  When using `rpdb` we'll see the following message on the plugin: *pdb is running on 127.0.0.1:4444*

Sending the signal to toggle ON/OFF the debugging is quite easy.  For baremetal we can do:

```
    $ pkill -USR1 cinderlib-csi
```

And for the container (assuming its named `cinderlib-csi` like in the examples) we can do:

```
    $ docker kill -sUSR1 cinderlib-csi
```

If we are using `rpdb` then we'll have to connect to the port:

```
    $ nc 127.0.0.1 4444
```

## Troubleshooting

### CSC commands timeout

If you have a slow backend or a slow data network connection, and you are creating mount volumes, then you may run into "context deadline exceeded" errors when running the node staging command on the volume.

This is just a 60 seconds timeout, and we can easily fix this by increasing allowed timeout for the command to complete.  For example to 5 minutes with `-t5m` or to 1 hour if we are manually debugging things on the server side with `-t1h`.

### Staging fails in container using iSCSI


When I try to stage a volume using a containerized *Node* I see the error "ERROR root VolumeDeviceNotFound: Volume device not found at .".

Turning the DEBUG log levels shows me login errors:

```
    2018-07-03 11:14:57.258 1 WARNING os_brick.initiator.connectors.iscsi [req-0e77bf32-a29b-40d1-b359-9e115435a94a com.redhat.cinderlib-csi com.redhat.cinderlib-csi - - -] Failed to connect to iSCSI portal 192.168.1.1:3260.
    2018-07-03 11:14:57.259 1 WARNING os_brick.initiator.connectors.iscsi [req-0e77bf32-a29b-40d1-b359-9e115435a94a com.redhat.cinderlib-csi com.redhat.cinderlib-csi - - -] Failed to login iSCSI target iqn.2008-05.com.something:smt00153500071-514f0c50023f6c01 on portal 192.168.1.1:3260 (exit code 12).: ProcessExecutionError: Unexpected error while running command.
```

And looking into the host's journal (where the `iscsid` daemon is running) I can see `Kmod` errors:

```
    Jul 03 13:15:02 think iscsid[9509]: Could not insert module . Kmod error -2
```

This seems to be cause by some kind of incompatibility between the host and the container's iSCSI modules.  We currently don't have a solution other than using a CentOS 7 host system.

## Support

For any questions or concerns please file an issue with the [cinderlib-csi](https://github.com/akrog/cinderlib-csi/issues) project or ping me on IRC (my handle is geguileo and I hang on the #openstack-cinder channel in Freenode).


## TODO

There are many things that need to be done in this POC driver, and here's a non exhaustive list:

- Support for NFS volumes
- Support for Kubernetes CRDs as the persistence storage
- Unit tests
- Functional tests
- Improve received parameters checking
- Make driver more resilient
- Test driver in Kubernetes
- Review some of the returned error codes
- Support volume attributes via cinder volume types
- Look into multi-attaching
- Support read-only mode
- Report capacity based on over provisioning values
- Configure the private data location
