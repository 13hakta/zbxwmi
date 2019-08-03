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
usage: zbxwmi [-h] [-action ACTION] [-namespace NAMESPACE] [-key KEY]
              [-fields FIELDS] [-filter FILTER] [-item ITEM] [-server SERVER]
              [-sender SENDER] [-dc-ip ip address]
              [-rpc-auth-level [{integrity,privacy,default}]]
              cls cred target

Zabbix WMI connector v0.1.1

positional arguments:
  cls                   <WMI Class>
  cred                  <Credential file>
  target                <targetName or address>

optional arguments:
  -h, --help            show this help message and exit
  -action ACTION        The action to take. Possible values : get, bulk, json,
                        discover, both
  -namespace NAMESPACE  namespace name (default //./root/cimv2)
  -key KEY              Key
  -fields FIELDS        Field list delimited by comma
  -filter FILTER        Filter
  -item ITEM            Selected item
  -server SERVER        Zabbix server
  -sender SENDER        Zabbix sender

authentication:
  -dc-ip ip address     IP Address of the domain controller. If ommited it use
                        the domain part (FQDN) specified in the target
                        parameter
  -rpc-auth-level [{integrity,privacy,default}]
                        default, integrity (RPC_C_AUTHN_LEVEL_PKT_INTEGRITY)
                        or privacy (RPC_C_AUTHN_LEVEL_PKT_PRIVACY). For
                        example CIM path "root/MSCluster" would require
                        privacy level by default)
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
 "/etc/zabbix/wmi.pw" \
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
 "/etc/zabbix/wmi.pw" \
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
 "/etc/zabbix/wmi.pw" \
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
 "/etc/zabbix/wmi.pw" \
 "remote.domain"
```
Outputs kind of:
`{ "data": [ {"{#WMI.DEVICEID}":"C:"}, {"{#WMI.DEVICEID}":"D:"}, {"{#WMI.DEVICEID}":"E:"} ] }`

### Discover

Get processor load:

`zbxwmi["-action","discover","-fields","PercentProcessorTime","-filter","Name<>'_Total'","Win32_PerfFormattedData_PerfOS_Processor",{$WMI_AUTHFILE},{HOST.HOST}]`

Get disk I/O load:

`zbxwmi["-action","-json","-k","Name","-fields","DiskWritesPersec,DiskWriteBytesPersec,DiskReadsPersec,DiskReadBytesPersec,CurrentDiskQueueLength","-filter","Name='_Total'","Win32_PerfRawData_PerfDisk_LogicalDisk",{$WMI_AUTHFILE},{HOST.HOST}]`

Get memory load:

`zbxwmi["-action","-json","-fields","AvailableBytes,CommitLimit,CommittedBytes","Win32_PerfRawData_PerfOS_Memory",{$WMI_AUTHFILE},{HOST.HOST}]`

## Zabbix usage

* Create template
* Set macro `{$WMI_AUTHFILE}` = `/etc/zabbix/wmi.pw`
* Create discovery rule with external check script kind of
`zbxwmi["-action","discover","-k","DeviceID","-filter","MediaType=12","Win32_LogicalDisk","{$WMI_AUTHFILE}",{HOST.HOST}]`
* Create discrovery item prototypes
  * Create main item to receive multiple values kind of `Field[{#WMI.NAME}]`
  * Create dependent items with JSON preprocessing like `Field2[{#WMI.NAME}]` and JSON address `$[0].Field2`
* Create graph prototype
* Optionally create trigger
* Assign template to MS Windows hosts
