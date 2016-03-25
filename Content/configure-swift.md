
###Working with the Swift Object Storage Service
After starting succesfully the Swift Object Storage Service, it is possible to store and retrive data in containers using the CLI Swift client.

Source the user credentials and query the service using the CLI client
```
# source /root/keystonerc_admin
# swift stat
        Account: AUTH_a6c6d1f171e649ba8af7a7a6cf63c3b9
     Containers: 0
        Objects: 0
          Bytes: 0
X-Put-Timestamp: 1421939769.00333
    X-Timestamp: 1421939769.00333
     X-Trans-Id: tx0df39bb7df404386aa13b-0054c11438
   Content-Type: text/plain; charset=utf-8

```

Create two containers and upload some content
```
# swift post container01
# swift post container02
# swift upload container01 somedata.file
# swift upload container01 somedata2.file
# swift upload container02 somedata3.file
```

View the list and content of containers
```
# swift list
container01
container02
# swift list container01
somedata.file
somedata2.file
# swift list container02
somedata3.file
# swift stat
                        Account: AUTH_a6c6d1f171e649ba8af7a7a6cf63c3b9
                     Containers: 2
                        Objects: 1
                          Bytes: 1024
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 1024
    X-Account-Project-Domain-Id: default
                    X-Timestamp: 1421940332.49074
                     X-Trans-Id: tx3c6abaa53cfb44a485b71-0054c11793
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
[root@caldera01 ~(keystone_admin)]$
```

Delete the containers and their content
```
# swift delete container01
somedata.file
somedata2.file
# swift delete container02
somedata3.file
# swift list
# 
```
