# zbxwmi
Zabbix WMI connector - discover and retrieve info from MS Windows hosts without installing agent.

Depends on python library [impacket](https://github.com/CoreSecurity/impacket).

Operates in 4 modes:
* Discover objects
* Get one value
* Get multiple values in JSON
* Get multiple values and send them with `zabbix_sender`

## Command line parameters

```
usage: zbxwmi [-h] [-v] [-action {get,bulk,json,discover,both}]
              [-namespace NAMESPACE] [-key KEY] [-fields FIELDS]
              [-filter FILTER] [-item ITEM] [-server address] [-sender path]
              [-cred CRED] [-dc-ip ip address]
              [-rpc-auth-level [{integrity,privacy,default}]]
              class target

Zabbix WMI connector

positional arguments:
  class                 WMI class
  target                target address

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         show program's version number and exit
  -action {get,bulk,json,discover,both}
                        Action to take (default: get)
  -namespace NAMESPACE  namespace name (default: //./root/cimv2)
  -key KEY              Key
  -fields FIELDS        Field list delimited by comma
  -filter FILTER        Filter
  -item ITEM            Selected item

Zabbix:
  -server address       Zabbix server (default: 127.0.0.1)
  -sender path          Zabbix sender (default: /usr/bin/zabbix_sender)

Authentication:
  -cred CRED            Credential file (default: /etc/zabbix/wmi.pw)
  -dc-ip ip address     IP Address of the domain controller. If ommited it use
                        the domain part (FQDN) specified in the target
                        parameter
  -rpc-auth-level [{integrity,privacy,default}]
                        integrity (RPC_C_AUTHN_LEVEL_PKT_INTEGRITY) or privacy
                        (RPC_C_AUTHN_LEVEL_PKT_PRIVACY). (default: default)
```

Credential file consists of three lines: login, password, domain.

## Installation

Assume you installed Zabbix Appliance with Ubuntu onboard. Access root shell and install appropriate dependencies.

Put `zbxwmi` script to `/usr/lib/zabbix/externalscripts` and set permissions:

```sh
# cd /usr/lib/zabbix/externalscripts
# chmod 755 zbxwmi
# chown root.root zbxwmi
```

Install required python modules:

```sh
# apt install python3-six python3-pycryptodome python3-pyasn1
```

Install [impacket](https://github.com/CoreSecurity/impacket) library.

Download from github or [stripped down](https://13hakta.ru/assets/components/fileattach/connector.php?action=web/download&ctx=web&fid=MDK5dMZwyEHoTNkHGkamjLSs7fIpRXTh) version sufficient to perform WMI calls.
Unpack contents to directory `/usr/lib/python3.6` (check a corresponding version).

Create file /etc/zabbix/wmi.pw with login, password and domain one parameter per line. Set file access:

```sh
# chmod 640 /etc/zabbix/wmi.pw
# chown zabbix.zabbix /etc/zabbix/wmi.pw
```

## Examples

### Get one value:

Receive available drive space

```sh
$ zbxwmi \
 -a get \
 -k DeviceID \
 -fields "FreeSpace" \
 -item "C:" \
 "Win32_LogicalDisk" \
 "remote.domain"
```

Outputs kind of:
`5121286144`

### Get multiple values:

Receive available and total drive space with zabbix_sender

```sh
$ zbxwmi \
 -a bulk \
 -k DeviceID \
 -fields "Size,FreeSpace" \
 -item "C:" \
 "Win32_LogicalDisk" \
 "remote.domain"
```

Receive available and total drive space

```sh
$ zbxwmi \
 -a json \
 -k DeviceID \
 -fields "Size,FreeSpace" \
 -item "C:" \
 "Win32_LogicalDisk" \
 "remote.domain"
```

Outputs kind of:
`[{"Size": "2197918158848", "DeviceID": "C:", "FreeSpace": "5121286144"}]`

### Discover objects:

Discover local drive partitions

```sh
$ zbxwmi \
 -action discover \
 -k DeviceID \
 -filter MediaType=12 \
 "Win32_LogicalDisk" \
 "remote.domain"
```
Outputs kind of:
`{ "data": [ {"{#WMI.DEVICEID}":"C:"}, {"{#WMI.DEVICEID}":"D:"}, {"{#WMI.DEVICEID}":"E:"} ] }`

### Discover

Get processor load:

`zbxwmi["-action","discover","-fields","PercentProcessorTime","-filter","Name<>'_Total'","Win32_PerfFormattedData_PerfOS_Processor",{HOST.HOST}]`

Get disk I/O load:

`zbxwmi["-action","-json","-k","Name","-fields","DiskWritesPersec,DiskWriteBytesPersec,DiskReadsPersec,DiskReadBytesPersec,CurrentDiskQueueLength","-filter","Name='_Total'","Win32_PerfRawData_PerfDisk_LogicalDisk",{HOST.HOST}]`

Get memory load:

`zbxwmi["-action","-json","-fields","AvailableBytes,CommitLimit,CommittedBytes","Win32_PerfRawData_PerfOS_Memory",{HOST.HOST}]`

## Zabbix usage

* Create template
* If your credential file located not in /etc/zabbix/wmi.pw, then set macro `{$WMI_AUTHFILE}` = `/path/to/wmi.pw`
* Create discovery rule with external check script kind of
`zbxwmi["-action","discover","-k","DeviceID","-filter","MediaType=12","-cred","{$WMI_AUTHFILE}","Win32_LogicalDisk",{HOST.HOST}]`
* Create discrovery item prototypes
  * Create main item to receive multiple values kind of `Field[{#WMI.NAME}]`
  * Create dependent items with JSON preprocessing like `Field2[{#WMI.NAME}]` and JSON address `$[0].Field2`
* Create graph prototype
* Optionally create trigger
* Assign template to MS Windows hosts
