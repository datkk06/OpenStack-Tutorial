###Managing Block Storage Service
Volumes are block storage devices that you attach to instances to enable persistent storage. Persistent storage is not deleted when you terminate an instance. You can attach a volume to a running instance or detach a volume and attach it to another instance at any time. You can also delete a volume. Only administrative users can create volume types.

On the other side, the disks associated with the instances are ephemeral, meaning that the data is deleted when the instance is terminated. Snapshots created from a running instance will remain, but any additional data added to the ephemeral storage since last snapshot will be lost.

####Create a volume
In this example, boot an instance from an image and attach a non-bootable volume
```
# nova image-list
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 8c28b5a7-3959-4f61-9f2f-b8ca720be3c2 | linux        | ACTIVE |        |
| 291c448c-2f1d-411e-9706-fab665ad6a33 | ubuntu-14.04 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+
```

Create a 10GB non bootable volume specifying the volume type (i.e. the backend) 
```
# cinder create --display-name my-volume 10 --volume-type lvm_gold
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Multiattach |             Attached
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
| f3af9952-8658-4cc1-ae72-2cb1574a1e28 | available |  my-volume   |  10  |   lvm_gold  |  false   |    False    |                     
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
```

Boot an instance from an image and attach the empty volume to the instance. Make sure to specify flavour, image and network interface of the instance
```
# nova boot --flavor small --image ubuntu-14.04 --nic net-id=a284bf6b-ea4a-40f5-b573-d9393db5a6dc --block-device source=volume,id=f3af9952-8658-4cc1-ae72-2cb1574a1e28,dest=volume,shutdown=preserve myInstanceWithVolume
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ID                                   | Name                 | Status | Task State | Power State | Networks                    |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ecc04ee0-fdd0-4760-bcc7-969adf95c4cb | myInstanceWithVolume | ACTIVE | -          | Running     | tenant-network=192.168.1.15 |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
```

