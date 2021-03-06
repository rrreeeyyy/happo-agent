# happo-agent - yet another Nagios nrpe

[![wercker status](https://app.wercker.com/status/1d02bef8da5959d5b6456e25835ae026/s/ "wercker status")](https://app.wercker.com/project/byKey/1d02bef8da5959d5b6456e25835ae026)

## Description

`happo-agent` is yet another Nagios nrpe plugin. And improvement nrpe functions.

- More secure communication. Supports TLS 1.2.
- Less fork cost at bastion(proxy) mode. Proxy request handled by thread (not fork()).
- Metric collection. Compatible to Sensu plugin format.
- inventory collection.


## Usage

### Requires

- Red Hat Enterprise Linux (RHEL) 6.x, 7.x
- CentOS 6.x, 7.x
- Ubuntu 12.04 or later

### Daemon mode (for monitoring)

#### How to execute

```
/path/to/happo-agent daemon -A [Accept from IP/Subnet] -B [Public key file] -R [Private key file] -M [Metric config file (Accept empty file)]
```

#### Monitoring

Call plugin from [`check_happo`](https://github.com/heartbeatsjp/check_happo), `happo-agent` calls local nagios plugin program. Then, return code and value to `check_happo`.

For more information, please see `check_happo` README.

#### Metric collection

Every one minute, execute sensu metrics plugin defined by `metrics.yaml`, and buffering results.

If you collect buffering results, you can use API `/metric` method.

#### Inventory collection

Get command based inventory data via API `/inventory` method.

### API client mode

You create `happo-agent` client management server if you want.

Use api client commands, `happo-agent` calls endpoint url which is client management server.

#### Host add request

```
/path/to/happo-agent add -e [ENDPOINT_URL] -g [GROUP_NAME[!SUB_GROUP_NAME]] -i [OWN_IP] -H [HOSTNAME] [-p BASTON_IP]
```

#### Is host available ?

```
/path/to/happo-agent is_added -e [ENDPOINT_URL] -g [GROUP_NAME[!SUB_GROUP_NAME]] -i [OWN_IP]
```

#### Host remove request

```
/path/to/happo-agent remove -e [ENDPOINT_URL] -g [GROUP_NAME[!SUB_GROUP_NAME]] -i [OWN_IP]
```

## Install

### Source based install (Use upstart)

```bash
$ sudo yum install epel-release
$ sudo yum install nagios-plugins-all
$ go get -dv github.com/heartbeatsjp/happo-agent
$ cd $GOHOME/src/bin
$ openssl genrsa -aes128 -out happo-agent.key 2048
$ openssl req -new -key happo-agent.key -sha256 -out happo-agent.csr
$ openssl x509 -in happo-agent.csr -days 3650 -req -signkey happo-agent.key -sha256 -out happo-agent.pub
$ touch metrics.yaml
$ chmod go-rwx happo-agent.key
$ sudo vim /etc/init/happo-agent.conf
$ sudo initctl reload-configuration
$ sudo initctl start happo-agent
```

happo-agent.conf

```
description "happo-agent"
author  "Your Name <USER@example.com>"

start on runlevel [2345]
stop on runlevel [016]

env LANG=C
env MARTINI_ENV=production
env APPNAME=happo-agent

exec /path/to/${APPNAME} daemon -A [Accept from IP/Subnet] -B ./${APPNAME}.pub -R ./${APPNAME}.key -M metrics.yaml >/dev/null 2>&1
respawn
```

You want to use sensu metrics plugins, should install `/usr/local/bin`.


### Metric collection configuration

metrics.yaml

```
metrics:
  - hostname: [HOSTNAME]
    plugins:
    - plugin_name: [Sensu plugin name (Path not needed)]
      plugin_option: [Sensu plugin name options]
    - ...
  - ...
```


## API

- Listen port: 6777 (Default)
- HTTPS, TLS 1.2, CipherSuite: `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`

### /

Check available.

- Input format
    - None
- Return format
    - String "OK"

```
$ wget -q --no-check-certificate -O - https://127.0.0.1:6777/
OK
```

### /proxy

Use agent bastion(proxy) mode.

- Input format
    - JSON
- Input variables
    - proxy\_hostport:
        - (Array) bastion_ip:port. It can multiple define.
    - request\_type: request type (e.g. `monitor`)
    - request\_json: Send JSON string to server.
- Return format
    - JSON
- Return variables
    - By `request_type` type.

```
$ wget -q --no-check-certificate -O - https://192.0.2.1:6777/proxy --post-data='{"proxy_hostport": ["198.51.100.1:6777"], "request_type": "monitor", "request_json": "{\"apikey\": \"\", \"plugin_name\": \"check_procs\", \"plugin_option\": \"-w 100 -c 200\"}"}'
{"return_value":1,"message":"PROCS WARNING: 168 processes\n"}
```

Example calls `wget host -> https://192.0.2.1:6777/proxy -> https://198.51.100.1:6777/monitor`.

### /inventory

Get inventory information from command.

- Input format
    - JSON
- Input variables
    - apikey: ""
    - command: execute command
    - command\_option: command option
- Return format
    - JSON
- Return variables
    - return\_code: commands return code
    - return\_value: commands return value (stdout, stderr)

```
$ wget -q --no-check-certificate -O - https://127.0.0.1:6777/inventory --post-data='{"apikey": "", "command": "uname", "command_option": "-a"}'
{"return_code":0,"return_value":"Linux saito-hb-vm101 2.6.32-573.3.1.el6.x86_64 #1 SMP Thu Aug 13 22:55:16 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux\n"}
```

### /monitor

Call monitor plugin. It likes nrpe.

- Input format
    - JSON
- Input variables
    - apikey: ""
    - command: execute nagios plugin command
    - command\_option: command option
- Return format
    - JSON
- Return variables
    - return\_code: commands return code
    - return\_value: commands return value (stdout, stderr)

```
$ wget -q --no-check-certificate -O - https://127.0.0.1:6777/monitor --post-data='{"apikey": "", "plugin_name": "check_procs", "plugin_option": "-w 100 -c 200"}'
{"return_value":1,"message":"PROCS WARNING: 168 processes\n"}
```

### /metric

Get collected metric values.

- Input format
    - JSON
- Input variables
    - apikey: ""
- Return format
    - JSON
- Return variables
    - MetricData:
        - (Array)
            - hostname: Hostname
            - timestamp: Unix time
            - metrics: metric name - metric value (key-value)
    - Message: message from agent (if error occurred)

```
$ wget -q --no-check-certificate -O - https://127.0.0.1:6777/metric --post-data='{"apikey": ""}'
{"metric_data":[{"hostname":"saito-hb-vm101","timestamp":1444028730,"metrics":{"linux.context_switches.context_switches":32662,"linux.disk.elapsed.iotime_sda":52,"linux.disk.elapsed.iotime_weighted_sda":82,"linux.disk.rwtime.tsreading_sda":0,"linux.disk.rwtime.tswriting_sda":82,"linux.forks.forks":88,"linux.interrupts.interrupts":19642,"linux.ss.CLOSE-WAIT":0,"linux.ss.CLOSING":0,"linux.ss.ESTAB":9,"linux.ss.FIN-WAIT-1":0,"linux.ss.FIN-WAIT-2":0,"linux.ss.LAST-ACK":0,"linux.ss.LISTEN":31,"linux.ss.SYN-RECV":0,"linux.ss.SYN-SENT":0,"linux.ss.TIME-WAIT":7,"linux.ss.UNCONN":0,"linux.ss.UNKNOWN":0,"linux.swap.pswpin":0,"linux.swap.pswpout":0,"linux.users.users":1}},…(snip)…],"message":""}
```

### /metric/config/update

*TODO*

### /metric/status

Get collected metric status.

- Input format
    - None
- Input variables
    - None
- Return format
    - JSON
- Return variables
    - length: length of metric_data_buffer
    - capacity: capacity of metric_data_buffer
    - oldest_timestamp: oldest Timestamp(int64) in metric_data_buffer
    - newest_timestamp: newest Timestamp(int64) in metric_data_buffer

```
$ wget -q --no-check-certificate -O - https://127.0.0.1:6777/metric/status
{"capacity":4,"length":4,"newest_timestamp":1454654233,"oldest_timestamp":1454654173}
```

## Contribution

1. Fork ([http://github.com/heartbeatsjp/happo-agent/fork](http://github.com/heartbeatsjp/happo-agent/fork))
1. Create a feature branch
1. Commit your changes
1. Rebase your local changes against the master branch
1. Run test suite with the `go test ./...` command and confirm that it passes
1. Run `gofmt -s`
1. Create a new Pull Request


## Author

- [Yuichiro Saito](https://github.com/koemu)
- [Toshiaki Baba](https://github.com/netmarkjp)

## License

Copyright 2016 HEARTBEATS Corporation.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
