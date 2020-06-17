# RPKI Validator
### RPKI Validator based on CloudFlare GoRTR and OctoRPKI

A quick a simple docker compose we use internally to run CloudFlare RPKI Validator, both containers are built from the official CloudFlare images. See [resources section](#resources) for links to more info.

## Run with Docker

Clone the repo:

```
$ git clone git@github.com:techfutures/rpki-validator.git
```

Start with docker-compose:

```
$ docker-compose up
```

Check containers are running:

```
$ sudo docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                              NAMES
cd46f0627ce3        cloudflare/gortr:latest      "./gortr -cache http…"   10 days ago         Up 41 minutes       0.0.0.0:8282-8283->8282-8283/tcp   gortr
dbb3480ee86d        cloudflare/octorpki:latest   "./octorpki"             10 days ago         Up 9 minutes        0.0.0.0:8080->8080/tcp             octorpki
```

#### Fetching the TALs

All the TALs files are included in the repo except ARIN.

You can download the RFC 7730 format at the following address: https://www.arin.net/resources/rpki/tal.html and drop it in the TALs data folder.

```
$ wget https://www.arin.net/resources/manage/rpki/arin.tal
```

Check volume name and mountpoint:
```
$ sudo docker volume ls
DRIVER              VOLUME NAME
local               tfi_octorpki-cache
local               tfi_octorpki-tals

$ sudo docker volume inspect tfi_octorpki-tals
[
    {
        "CreatedAt": "2020-04-19T01:21:52-07:00",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "tfi",
            "com.docker.compose.volume": "octorpki-tals"
        },
        "Mountpoint": "/var/lib/docker/volumes/tfi_octorpki-tals/_data",
        "Name": "tfi_octorpki-tals",
        "Options": null,
        "Scope": "local"
    }
]

```

Copy to TALs data volume

```
$ cp arin.tal /var/lib/docker/volumes/tfi_octorpki-tals/_data
```

Confirm all TALs are in the volume

```
$ /var/lib/docker/volumes/tfi_octorpki-tals/_data# ls
afrinic.tal  apnic.tal  arin.tal  lacnic.tal  ripe.tal
```



## Usage Example
#### Configure on Juniper

Configure a session to the RTR server (assuming it runs on `10.0.0.1:8282`)

```
tfi@router> show configuration routing-options validation
group RPKIValidator1 {
    session 10.0.0.1 {
        port 8282;
    }
}
```

Add policies to validate or invalidate prefixes

```
tfi@router> show configuration policy-options policy-statement VALIDATION
term VALID {
    from {
        protocol bgp;
        validation-database valid;
    }
    then {
        validation-state valid;
        community add ORIGIN-VALIDATION-STATE-VALID;
        next policy;
    }
}
term INVALID {
    from {
        protocol bgp;
        validation-database invalid;
    }
    then {
        validation-state invalid;
        community add ORIGIN-VALIDATION-STATE-INVALID;
        reject;
    }
}
term UNKNOWN {
    from protocol bgp;
    then {
        validation-state unknown;
        community add ORIGIN-VALIDATION-STATE-UNKNOWN;
        next policy;
    }
}
```

Display status of the session to the RTR server.

```
tfi@router> show validation session 10.0.0.1 detail
Session 10.0.0.1, State: up, Session index: 3
  Group: RPKIValidator1, Preference: 100
  Port: 8282
  Refresh time: 300s
  Hold time: 600s
  Record Life time: 3600s
  Serial (Full Update): 1355
  Serial (Incremental Update): 1355
    Session flaps: 1
    Session uptime: 3w1d 18:08:08
    Last PDU received: 00:04:47
    IPv4 prefix count: 133156
    IPv6 prefix count: 22453

```

Show content of the database (list the PDUs)

```
tfi@router> show validation database brief
RV database for instance master

Prefix                 Origin-AS Session                                 State   Mismatch
1.0.0.0/24-24              13335 10.0.0.1                                valid
1.1.1.0/24-24              13335 10.0.0.1                                valid
```

## Resources
- https://github.com/cloudflare/cfrpki
- https://github.com/cloudflare/gortr
