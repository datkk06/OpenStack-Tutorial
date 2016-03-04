###Managing Block Storage Service
Volumes are block storage devices that you attach to instances to enable persistent storage. Persistent storage is not deleted when you terminate an instance. You can attach a volume to a running instance or detach a volume and attach it to another instance at any time. You can also delete a volume. Only administrative users can create volume types.

On the other side, the disks associated with the instances are ephemeral, meaning that the data is deleted when the instance is terminated. Snapshots created from a running instance will remain, but any additional data added to the ephemeral storage since last snapshot will be lost.

####Create a non bootable volume
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

Boot an instance from an image and attach the empty volume to the instance. Make sure to specify flavour, image, network interface and volume ID
```
# nova boot --flavor small --image ubuntu-14.04 --nic net-id=a284bf6b-ea4a-40f5-b573-d9393db5a6dc --block-device source=volume,id=f3af9952-8658-4cc1-ae72-2cb1574a1e28,dest=volume,shutdown=preserve myInstanceWithVolume
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ID                                   | Name                 | Status | Task State | Power State | Networks                    |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ecc04ee0-fdd0-4760-bcc7-969adf95c4cb | myInstanceWithVolume | ACTIVE | -          | Running     | tenant-network=192.168.1.15 |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
```

To detach the volume from the image, specify the instance name and the volume ID
```
# nova volume-detach myInstanceWithVolume f3af9952-8658-4cc1-ae72-2cb1574a1e28
```

####Create a bootable volume
In this example, create a bootable volume from an image and boot a persistent instance.

Make sure to specify:
* --flavor=ID
* --image=ID
* --nic net-id=ID
* --block-device source=volume,id=ID,dest=DEST,size=SIZE,shutdown={preserve|remove},bootindex=0

```
# nova list
# nova flavor-list
# neutron net-list
+--------------------------------------+-----------------------+-----------------------------------------------------+
| id                                   | name                  | subnets                                             |
+--------------------------------------+-----------------------+-----------------------------------------------------+
| a284bf6b-ea4a-40f5-b573-d9393db5a6dc | tenant-network        | 75208e5c-a3f0-4f9f-a763-dd3b256fec56 192.168.1.0/24 |
| 47f17be1-7a93-4045-85c7-12e3a3b30446 | dmz-network           | 358702ed-1d7a-4c72-a455-faff96b616ae 192.168.0.0/24 |
| 66cde443-7030-4413-816d-ee140442d89e | provider-flat-network | 4b57b58b-146e-4170-992e-40131df9e8c3 172.16.1.0/24  |
+--------------------------------------+-----------------------+-----------------------------------------------------+
# nova image-list
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 8c28b5a7-3959-4f61-9f2f-b8ca720be3c2 | linux        | ACTIVE |        |
| 291c448c-2f1d-411e-9706-fab665ad6a33 | ubuntu-14.04 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+

# nova boot --flavor m1.small --nic net-id=a284bf6b-ea4a-40f5-b573-d9393db5a6dc --block-device source=image,id=291c448c-2f1d-411e-9706-fab665ad6a33,dest=volume,size=10,shutdown=preserve,bootindex=0 myInstanceFromVolume

# cinder list
+--------------------------------------+--------+------+------+-------------+----------+-------------+--------------------------------
|                  ID                  | Status | Name | Size | Volume Type | Bootable | Multiattach |             Attached to        
+--------------------------------------+--------+------+------+-------------+----------+-------------+--------------------------------
| 756dc2a8-444f-4f16-ab6a-1474300b4cde | in-use |      |  10  |   lvm_gold  |   true   |    False    | 5d4682a8-5791-4bc2-86fd-9aeff69f517a |
+--------------------------------------+--------+------+------+-------------+----------+-------------+--------------------------------
# nova list
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ID                                   | Name                 | Status | Task State | Power State | Networks                    |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| 5d4682a8-5791-4bc2-86fd-9aeff69f517a | myInstanceFromVolume | ACTIVE | -          | Running     | tenant-network=192.168.1.17 |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
```



