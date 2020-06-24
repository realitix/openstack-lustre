# Lustre driver for Cinder

## Introduction

This paper provides specification needed to develop the Lustre driver for Cinder.
The development will be done in three steps:

1. The proper driver will be developed with basic Lustre functionnality
2. Lustre support will be added to the backup functionnality of Cinder
3. The unit and Tempest tests will be added to validate the good working of the driver
4. Advanced Lustre functionnality will be progressively added to the Cinder driver

To develop this module, a development infrastructure will be installed on the developer computer but should not be used to launch validations tests (Tempest).
Indeed, theses tests could take hours on a basic computer and need a complete installation on OpenStack whereas using directly the OpenStack CI guarantees a good behavior without the complex installation.
In order to launch the tests, the developer should submit or update a gerrit patch and the OpenStack CI will do the job. Moreover, this will allow the developer to continue the work without overloading its computer.
This process will be used for this development.

> Note:
>     At first, this module will not support:
>     - Lustre Striping (as well as PFL) support
>     - Snapshot support
>     - Cloning support
>     - JOB Stats integration for performance monitoring and QoS

## 1 - The Lustre Driver

### Location

The Lustre driver will be developed in the Python module `cinder/volume/drivers/lustre.py` in order to respect the Cinder conventions.
The class will be named `LustreDriver`.


### Class inheritance

#### `volumedriver` interface

The class `LustreDriver` will be annoted with `@interface.volumedriver`.
This interface is the core backend volume driver interface.
As mentionned in the docstring: *All backend drivers should support this interface as a bare minimum*.


#### `RemoteFSSnapDriverDistributed` inheritance

After implementing the volumedriver interface, we need to choose a base class suiting our needs.
The best candidate is the class `cinder.volume.drivers.remotefs.RemoteFSSnapDriverBase`.
This class implements `RemoteFSDriver` which is a *Common base for drivers that work like NFS*, it's indeed what we need.
We don't choose directly `RemoteFSDriver` but `RemoteFSSnapDriverBase` in order to add the `qcow2` snapshot functionality.

#### `ExtendVD` inheritance

Contrary to the last gerrit patch trying to implement the Cinder Lustre Driver, the `ExtendVD` parent class is no more needed for two reasons:

- This class has been deprecated and now deleted
- The `BaseVD` now directly implements the `extend_volume` method
- The `RemoteFsDriver` class arleady inherit from `BaseVD` and thus our `LustreDriver` class inherit from `BaseVD`


### Method implementations

> Note: 
>    Operations such as create/delete/extend volume/snapshot use locking on a per-process basis to prevent multiple threads from modifying qcow2 chains or the snapshot .info file simultaneously.

Here the required methods to implement:

- `do_setup`
- `create_volume`
- `delete_volume`
- `ensure_export`
- `create_export`
- `remove_export`
- `validate_connector`
- `initialize_connection`
- `terminate_connection`
- `extend_volume`


**Let's analyze the needed work for each method.**


#### `do_setup`

Any initialization the volume driver does while starting.
In this method, the driver create the `remotefs` client by calling `RemoteFsClient` constructor.
Then, the Lustre fs is mounted and all mounts points are refreshed.

#### `create_volume`

Create a volume on given `lustre_share`.
First we retrieve properties of the volume (size and path) and check if it already exists.
If the volume type is `thin`, we create a `qcow2` file else directly a raw file using fallocate.

#### `delete_volume`

Delete a logical volume.
After processing the usual tests (share mounted), we simply execute `rm -rf` of the volume to delete.

#### `ensure_export`

Synchronously recreates an export for a logical volume.
We ensure that all Lustre shares are mounted.

#### `create_export`

Exports the volume.
We'll not implement this method.

#### `remove_export`

Removes an export for a logical volume.
We'll not implement this method.


#### `validate_connector`

We'll not implement this method.

#### `initialize_connection`

Allow connection to connector and return connection info.
This method will need several steps:

- First find the active `qcow2` file from volume information.
- Then we test the file to find if it's a `qcow2' or a `raw` file.
- Finally, we send back the connection informations.

#### `terminate_connection`

Disallow connection from connector.
We'll not implement this method.

#### `extend_volume`

We use `qemu-img` to resize the volume.
`qemu-img` can resize both raw and qcow2 files.


## 2 - Backup support for Lustre

### Location

The Lustre backup driver will be developed in the Python module `cinder/backup/drivers/lustre.py` in order to respect the Cinder conventions.
The class will be named `LustreBackupDriver`.


### Class inheritance

#### `backupdriver` interface

The class `LustreBackupDriver` will be annoted with `@interface.backupdriver`.
This interface is the core backup driver interface.
As mentionned in the docstring: *All backup drivers should support this interface as a bare minimum.*.

#### `PosixBackupDriver` inheritance

After implementing the backupdriver interface, we need to choose a base class suiting our needs.
The best candidate is the class `cinder.backup.drivers.posix.PosixBackupDriver`.
This class implements `BackupDriver` which is a *Common backup base class*, it's indeed what we need.


### Method implementations

Provides backup, restore and delete using Lustre repository.
We don't need to override method from `PosixBackupDriver` but we need to give its the `backup_path`.

To get the backup path, we'll use this process:

- We create the `remotefs` client.
- We mount the backup share.
- We ensure we can write to this share.
- Finally we return the mount path.


## 3 - Unit and Tempest tests

### Location

The Lustre tests will be developed in the Python module `cinder/tests/unit/volume/drivers/test_lustre.py` in order to respect the Cinder conventions.
The class will be named `LustreDriverTestCase`.


### Tests

The goal is to provide a maximum of tests.
Here example of tests we will implement:

- Test local path common use case
- Test mount common use case
- Test mount lustre should reraise exception if mount fails
- Test get mount point
- Test calculation of capacity
- Test size of provisionned size
- Test update volume stats with qcow2 files
- Test update volume stats with raw files
- Test ensure share mounted
- Test ensure shares mounted should save share if mounted with success
- Test ensure shares mounted should not save share if failed to mount
- Test find share
- Test find share should throw error if no more space available
- Test create thin volume
- Test create thick fallocate volume
- Test create thick dd volume
- Test create volume should ensure lustre is mounted
- Test delete volume
- Test refresh mounts
- Test refresh mounts with exception
- Test mount shares
- Test unmount shares
- Test read info file
- Test extend volume
- Test extend volume with snapshot
- Test create snapshot
- Test delete snapshot
- Test copy volume from snapshot
- Test create volume from snapshot
- Test initialize connection
- Test backup volume
- Test copy volume to raw image
- Test copy volume to qcow2 image
- Test migrate volume
