
# Prepare Isilon OneFS

## 1. Create Swift Zone  

```shell
BeijingCSC-1# isi zone create --name swift-zone1 --groupnet groupnet0 --create-path --path /ifs/gaozy/swift

BeijingCSC-1# isi zone view swift-zone1 
                       Name: swift-zone1
                       Path: /ifs/gaozy/swift
                   Groupnet: groupnet0
              Map Untrusted: -
             Auth Providers: lsa-local-provider:swift-zone1
               NetBIOS Name: -
         User Mapping Rules: -
       Home Directory Umask: 0077
         Skeleton Directory: /usr/share/skel
         Cache Entry Expiry: 4H
Negative Cache Entry Expiry: 1m
                    Zone ID: 17
BeijingCSC-1# 
```

## 2. Create Swift Network Pool  
```shell
BeijingCSC-1# isi network pools create --id=groupnet0:subnet1:swift-zone1-pool --ranges=192.168.20.55-192.168.20.56 --access-zone=swift-zone1 --alloc-method=dynamic --ifaces=4-6:10gige-1, --sc-subnet=subnet1 --sc-dns-zone=swift-zone1.isilon.com --description="swift-zone1 pool"

BeijingCSC-1# isi network pools view --id=groupnet0:subnet1:swift-zone1-pool
                     ID: groupnet0.subnet1.swift-zone1-pool
               Groupnet: groupnet0
                 Subnet: subnet1
                   Name: swift-zone1-pool
                  Rules: -
            Access Zone: swift-zone1
      Allocation Method: dynamic
       Aggregation Mode: lacp
     SC Suspended Nodes: -
            Description: swift-zone1 pool
                 Ifaces: 4:10gige-1, 5:10gige-1, 6:10gige-1
              IP Ranges: 192.168.20.55-192.168.20.56
       Rebalance Policy: auto
SC Auto Unsuspend Delay: 0
      SC Connect Policy: round_robin
                SC Zone: swift-zone1.isilon.com
    SC DNS Zone Aliases: -
     SC Failover Policy: round_robin
              SC Subnet: subnet0
                 SC TTL: 0
          Static Routes: -


Isilon IP Address : swift-zone1.isilon.com,192.168.20.55-192.168.20.56
Local User : user01, user02
Local Group : swiftadmin
Swift Account Name : swift_account
Access Zone : swift-zone1
Access Zone home directory : /ifs/gaozy/swift/
```

## 3. Create Isilon Users & Groups  
```shell
BeijingCSC-1# isi auth groups create swiftadmin --zone swift-zone1
BeijingCSC-1# isi auth users create user01 --enabled 1 --password 123456 --primary-group swiftadmin --zone swift-zone1
BeijingCSC-1# isi auth users create user02 --enabled 1 --password 123456 --primary-group swiftadmin --zone swift-zone1
```


## 4. Create Swift Users & Groups  
```shell
BeijingCSC-1# isi swift accounts create swift_account user01 swiftadmin --users user01,user02 --zone swift-zone1
BeijingCSC-1# isi swift accounts list --zone swift-zone1
Name          Users 
--------------------
swift_account user01
              user02
--------------------
Total: 1

BeijingCSC-1# isi swift accounts view swift_account --zone swift-zone1
          Name: swift_account
          Zone: swift-zone1
          Path: /ifs/gaozy/swift/isi_lwswift/swift_account
    Swift User: user01
   Swift Group: swiftadmin
Assigned Users: user01, user02
```

## 5. Create Swift Containers  
```shell
BeijingCSC-1# mkdir -p /ifs/gaozy/swift/home/user01/user01-container
BeijingCSC-1# mkdir -p /ifs/gaozy/swift/home/user02/user02-container


BeijingCSC-1# mv /ifs/gaozy/swift/home/user01/user01-container /ifs/gaozy/swift/isi_lwswift/swift_account
BeijingCSC-1# mv /ifs/gaozy/swift/home/user02/user02-container /ifs/gaozy/swift/isi_lwswift/swift_account


BeijingCSC-1# isi_run -z 17 chown -R user01:swiftadmin /ifs/gaozy/swift/isi_lwswift/swift_account/user01-container
BeijingCSC-1# isi_run -z 17 chown -R user02:swiftadmin /ifs/gaozy/swift/isi_lwswift/swift_account/user02-container
```

# Test Swift with Curl
## 1. Example with HTTP  
```shell
curl -H "X-Auth-User:swift_account:user01" -H "X-Auth-Key:123456" -v "http://192.168.20.55:28080/auth/v1.0" -X GET
curl -H "X-Auth-Token:AUTH_tka708fa1da12eba3fcf7b3ff438a9591b" -i "http://192.168.20.55:28080/v1/AUTH_swift_account?format=json" -X GET
```

## 2. Example with HTTPS  
```shell
curl --insecure -H "X-Auth-User:swift_account:user01" -H "X-Auth-Key:123456" -v "https://192.168.20.56:8083/auth/v1.0" -X GET
curl --insecure -H "X-Auth-Token: AUTH_tkc94ac7b400c6b262ec177e32499dee39" -i "https://192.168.20.56:8083/v1/AUTH_swift_account?format=json" -X GET
```

## 3. HTTP Response Sample  
```shell
[root@linux01 ~]# curl -H "X-Auth-User:swift_account:user01" -H "X-Auth-Key:123456" -v "http://192.168.64.204:28080/auth/v1.0" -X GET
* About to connect() to 192.168.64.204 port 28080 (#0)
*   Trying 192.168.64.204... connected
* Connected to 192.168.64.204 (192.168.64.204) port 28080 (#0)
> GET /auth/v1.0 HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.19.1 Basic ECC zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: 192.168.64.204:28080
> Accept: */*
> X-Auth-User:swift_account:user01
> X-Auth-Key:123456
> 
< HTTP/1.1 200 OK
< Content-Length: 109
< Content-Type: application/json; charset=utf-8
< Date: Wed, 10 Jan 2018 03:14:30 UTC
< X-Auth-Token: AUTH_tk700825344f688d497665b78b87dcfba5
< X-Storage-Token: AUTH_tk700825344f688d497665b78b87dcfba5
< X-Storage-Url: http://192.168.64.204:28080/v1/AUTH_swift_account
< X-Trans-Id: txa6ee069aab1e49f69e462-005a558516
< 
* Connection #0 to host 192.168.64.204 left intact
* Closing connection #0
{"storage": {"cluster_name": "http://192.168.64.204:28080/v1/AUTH_swift_account", "default": "cluster_name"}}

```

```shell
[root@linux01 ~]# curl -H "X-Auth-Token: AUTH_tk700825344f688d497665b78b87dcfba5" -i "http://192.168.64.204:28080/v1/AUTH_swift_account?format=json" -X GET
HTTP/1.1 200 OK
Content-Length: 104
Content-Type: application/json; charset=utf-8
Date: Wed, 10 Jan 2018 03:14:32 UTC
Last-Modified: Wed, 10 Jan 2018 03:11:20 UTC
X-Account-Bytes-Used: 0
X-Account-Container-Count: 2
X-Account-Object-Count: 0
X-Timestamp: 131600274809915577
X-Trans-Id: tx840421bab7544c6dba2f2-005a558518

[{"bytes": 0,"count": 0,"name": "user01-container"},{"bytes": 0,"count": 0,"name": "user02-container"}]
```

## 4. HTTPS Response Sample  

```shell
[root@linux01 ~]# curl --insecure -H "X-Auth-User:swift_account:user01" -H "X-Auth-Key:123456" -v "https://192.168.64.204:8083/auth/v1.0" -X GET
* About to connect() to 192.168.64.204 port 8083 (#0)
*   Trying 192.168.64.204... connected
* Connected to 192.168.64.204 (192.168.64.204) port 8083 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* warning: ignoring value of ssl.verifyhost
* skipping SSL peer certificate verification
* SSL connection using TLS_DHE_RSA_WITH_AES_256_CBC_SHA
* Server certificate:
* 	subject: E=support@isilon.com,CN=Isilon Systems,OU=Isilon Systems,O="Isilon Systems, Inc.",L=Seattle,ST=Washington,C=US
* 	start date: May 18 22:49:07 2017 GMT
* 	expire date: May 17 22:49:07 2020 GMT
* 	common name: Isilon Systems
* 	issuer: E=support@isilon.com,CN=Isilon Systems,OU=Isilon Systems,O="Isilon Systems, Inc.",L=Seattle,ST=Washington,C=US
> GET /auth/v1.0 HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.19.1 Basic ECC zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: 192.168.64.204:8083
> Accept: */*
> X-Auth-User:swift_account:user01
> X-Auth-Key:123456
> 
< HTTP/1.1 200 OK
< Date: Wed, 10 Jan 2018 03:14:38 GMT
< Content-Length: 109
< Content-Type: application/json; charset=utf-8
< X-Auth-Token: AUTH_tk700825344f688d497665b78b87dcfba5
< X-Storage-Token: AUTH_tk700825344f688d497665b78b87dcfba5
< X-Storage-Url: https://192.168.64.204:8083/v1/AUTH_swift_account
< X-Trans-Id: tx1cd0aa868bef4412818a8-005a55851e
< 
* Connection #0 to host 192.168.64.204 left intact
* Closing connection #0
{"storage": {"cluster_name": "https://192.168.64.204:8083/v1/AUTH_swift_account", "default": "cluster_name"}}
```

```shell
[root@linux01 ~]# curl --insecure -H "X-Auth-Token: AUTH_tk700825344f688d497665b78b87dcfba5" -i "https://192.168.64.204:8083/v1/AUTH_swift_account?format=json" -X GET
HTTP/1.1 200 OK
Date: Wed, 10 Jan 2018 03:14:40 GMT
Content-Length: 104
Content-Type: application/json; charset=utf-8
Last-Modified: Wed, 10 Jan 2018 03:11:20 GMT
X-Account-Bytes-Used: 0
X-Account-Container-Count: 2
X-Account-Object-Count: 0
X-Timestamp: 131600274809915577
X-Trans-Id: txa92e697cd761405fb8190-005a558520

[{"bytes": 0,"count": 0,"name": "user01-container"},{"bytes": 0,"count": 0,"name": "user02-container"}]

```

