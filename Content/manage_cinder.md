###Managing Block Storage Service
Volumes are block storage devices that you attach to instances to enable persistent storage. Persistent storage is not deleted when you terminate an instance. You can attach a volume to a running instance or detach a volume and attach it to another instance at any time. You can also delete a volume. Only administrative users can create volume types.

On the other side, the disks associated with the instances are ephemeral, meaning that the data is deleted when the instance is terminated. Snapshots created from a running instance will remain, but any additional data added to the ephemeral storage since last snapshot will be lost.

####Create a volume
This example creates a new volume volume based on an image. List images and note the ID of the image that you want to use for your volume
```
# nova image-list
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 8c28b5a7-3959-4f61-9f2f-b8ca720be3c2 | linux        | ACTIVE |        |
| 291c448c-2f1d-411e-9706-fab665ad6a33 | ubuntu-14.04 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+
```

Create a volume with 8 GB of space, and specify the availability zone and image
```
# cinder create 8 --display-name my-new-volume --image-id 8c28b5a7-3959-4f61-9f2f-b8ca720be3c2
+--------------------------------------+-----------+---------------+------+-------------+----------+-------------+-------------------------------------+
|                  ID                  |   Status  |      Name     | Size | Volume Type | Bootable | Multiattach |             Attached to              |
+--------------------------------------+-----------+---------------+------+-------------+----------+-------------+--------------------
| e7324652-f7c3-4bd5-8aad-ee6ad3d977ba | available | my-new-volume |  8   |      -      |   true   |    False    |                                      |
+--------------------------------------+-----------+---------------+------+-------------+----------+-------------+-------------------------------------+
```

