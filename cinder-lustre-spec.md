# Lustre driver for Cinder

## Introduction

This paper provides specification needed to develop the Lustre driver for Cinder.
The development will be done in three steps:

1. The proper driver will be developed with basic Lustre functionality.
2. Lustre support will be added to the backup functionality of Cinder.
3. The unit and Tempest tests will be added to validate the good working of the driver.
4. Performance tests will be added at several levels.
5. Advanced Lustre functionality will be progressively added to the Cinder driver.

### Specification source

Theses specification are inspired from this patch: https://review.opendev.org/#/c/395572/

The first implementation is rebased on master Cinder code in order to comply with last Cinder version.
Indeed, you will find some differences between the patch and this specification document depending on the Cinder API update.


### Methodology
 
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

The class `LustreDriver` will be annotated with `@interface.volumedriver`.
This interface is the core backend volume driver interface.
As mentioned in the docstring: *All backend drivers should support this interface as a bare minimum*.


#### `RemoteFSSnapDriverDistributed` inheritance

After implementing the volumedriver interface, we need to choose a base class suiting our needs.
The best candidate is the class `cinder.volume.drivers.remotefs.RemoteFSSnapDriverBase`.
This class implements `RemoteFSDriver` which is a *Common base for drivers that work like NFS*, it's indeed what we need.
We don't choose directly `RemoteFSDriver` but `RemoteFSSnapDriverBase` in order to add the `qcow2` snapshot functionality.

#### `ExtendVD` inheritance

Contrary to the last gerrit patch trying to implement the Cinder Lustre Driver, the `ExtendVD` parent class is no more needed for two reasons:

- This class has been deprecated and now deleted
- The `BaseVD` now directly implements the `extend_volume` method
- The `RemoteFsDriver` class already inherit from `BaseVD` and thus our `LustreDriver` class inherit from `BaseVD`


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


#### do_setup

```
def do_setup(self, context) -> None
    - context: Context (cinder environment)

```

Any initialization the volume driver does while starting.
In this method, the driver creates the `remotefs` client by calling `RemoteFsClient` constructor.
Then, the Lustre fs is mounted and all mounts points are refreshed.

#### create_volume

```
def create_volume(self, volume) -> {'provider_location':}
    - volume: Volume to create
    return the Provider location
```

Create a volume on given `lustre_share`.
First we retrieve properties of the volume (size and path) and check if it already exists.
If the volume type is `thin`, we create a `qcow2` file else directly a raw file using `fallocate`.

#### delete_volume

```
def delete_volume(self, volume) -> None
    - volume: Volume to delete
```

Delete a logical volume.
After processing the usual tests (share mounted), we simply execute `rm -rf` on the volume to delete.

#### ensure_export

```
def ensure_export(self, context, volume) -> None
    - context: Context (cinder environment)
    - volume: Volume to export
```

Synchronously recreates an export for a logical volume.
We ensure that all Lustre shares are mounted.

#### create_export

```
def create_export(self, context, volume, connector) -> None
    - context: Context (cinder environment)
    - volume: Volume to export
    - connector: Connector to Lustre
```

Exports the volume.
We'll not implement this method.

#### remove_export

```
def remove_export(self, context, volume) -> None
    - context: Context (cinder environment)
    - volume: Volume to export
```

Removes an export for a logical volume.
We'll not implement this method.

#### validate_connector

```
def validate_connector(self, connector) -> None
    - connector: Connector to Lustre
```

We'll not implement this method.

#### initialize_connection

```
def initialize_connection(self, volume, connector) -> {'driver_volume_type','data','mount_point_base'}
    - volume: Volume to connect
    - connector: Connector to Lustre
```

Allow connection to connector and return connection info.
This method will need several steps:

- First find the active `qcow2` file from volume informations.
- Then we test the file to find if it's a `qcow2' or a `raw` file.
- Finally, we send back the connection informations.

#### terminate_connection

```
def terminate_connection(self, volume, connector) -> None
    - volume: Volume to connect
    - connector: Connector to Lustre
```

Disallow connection from connector.
We'll not implement this method.

#### `extend_volume`

```
def extend_volume(selv, volume, size_gb) -> None
    - volume: Volume to update
    - size_gb: New size in GB
```

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
We don't need to override method from `PosixBackupDriver` but we need to pass the `backup_path` to the constructor.

To get the backup path, we'll use this process:

- We create the `remotefs` client.
- We mount the backup share.
- We ensure we can write to this share.
- Finally we return the mount path.


## 3 - Unit and Tempest tests

### Location

The Lustre tests will be developed in the Python module `cinder/tests/unit/volume/drivers/test_lustre.py` in order to respect Cinder conventions.
This class will be named `LustreDriverTestCase`.


### Tests

The goal is to provide a large test coverage.
Here are examples of test we will implement:

- Test local path common use case

```
def test_local_patch(self)
```

- Test mount common use case

```
def test_mount_lustre(self)
```

- Test mount lustre should reraise exception if mount fails

```
def test_mount_lustre_should_reraise_exception_on_failure(self)
```

- Test get mount point

```
def test_get_mount_point_for_share(self)
```

- Test calculation of capacity

```
def test_get_available_capacity_with_df(delf)
```

- Test size of provisionned size

```
def test_get_provisioned_capacity(self)
```

- Test update volume stats with qcow2 files

```
def test_update_volume_stats_thin(self)
```

- Test update volume stats with raw files

```
def test_update_volume_stats_thick(self)
```

- Test ensure share mounted

```
def test_ensure_share_mounted(self)
```

- Test ensure shares mounted should save share if mounted with success

```
def test_ensure_shares_mounted_should_save_mounting_successfully(self)
```

- Test ensure shares mounted should not save share if failed to mount

```
def test_ensure_shares_mounted_should_not_save_mounting_with_error(self)
```

- Test find share

```
def test_find_share(self)
```

- Test find share should throw error if no more space available

```
def test_find_share_should_throw_error_if_there_is_no_enough_place(self)
```

- Test create thin volume

```
def test_create_thin_volume(self)
```

- Test create thick fallocate volume

```
def test_create_thick_fallocate_volume(self)
```

- Test create thick dd volume

```
def test_create_thick_dd_volume(self)
```

- Test create volume should ensure lustre is mounted

```
def test_create_volume_should_ensure_lustre_mounted(self)
```

- Test delete volume

```
def test_delete_volume(self, mock_delete_if_exists)
```

- Test refresh mounts

```
def test_refresh_mounts(self)
```

- Test refresh mounts with exception

```
def test_refresh_mounts_with_excp(self)
```

- Test mount shares

```
def test_do_mount(self)
```

- Test unmount shares

```
def test_do_umount(self)
```

- Test read info file

```
def test_read_info_file(self)
```

- Test extend volume

```
def test_extend_volume(self)
```

- Test extend volume with snapshot

```
def test_extend_volume_with_snapshot(self)
```

- Test create snapshot

```
def test_create_snapshot_online(self)
```

- Test delete snapshot

```
def test_delete_snapshot_online(self)
```

- Test copy volume from snapshot

```
def test_copy_volume_from_snapshot(self)
```

- Test create volume from snapshot

```
def test_create_volume_from_snapshot(self)
```

- Test initialize connection

```
def test_initialize_connection(self)
```

- Test backup volume

```
def test_backup_volume(self)
```

- Test copy volume to raw image

```
def test_copy_volume_to_image_raw_image(self)
```

- Test copy volume to qcow2 image

```
def test_copy_volume_to_image_qcow2_image(self)
```

- Test migrate volume

```
def test_migrate_volume_is_there(self)
```


## 4 - Performance tests

### Location

The performance tests will be developed in the Python module `cinder/tests/unit/volume/drivers/test_lustre_performance.py` in order to respect Cinder conventions.
This class will be named `LustreDriverPerformanceTestCase`.


### Tests


