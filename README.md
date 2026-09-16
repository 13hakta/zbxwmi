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
              [-filter FILTER] [-type TYPE] [-item ITEM]
              [-server address] [-sender path] [-cred CRED]
              [-dc-ip ip address] [-rpc-auth-level [{integrity,privacy,default}]]
              class target

usage: zbxwmi [-h] [-v] [-action {get,bulk,json,discover,both}] [-namespace NAMESPACE] [-key KEY] [-fields FIELDS] [-type TYPE] [-filter FILTER]
              [-item ITEM] [-server address] [-sender path] [-cred CRED] [-dc-ip ip address] [-rpc-auth-level [{integrity,privacy,default}]]
              class target



Zabbix WMI connector

positional arguments:
  class                 WMI class
  target                Target address. Must be FQDN for kerberos auth type.

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         show program's version number and exit
  -action {get,bulk,json,discover,both}
                        Action to take (default: get)
  -namespace NAMESPACE  namespace name (default: //./root/cimv2)
  -key KEY              Key
  -fields FIELDS        Field list delimited by comma
  -type TYPE            Field type hint delimited by comma: n - number, s - string (default)
  -filter FILTER        Filter
  -item ITEM            Selected item

Zabbix:
  -server address       Zabbix server (default: 127.0.0.1)
  -sender path          Zabbix sender (default: /usr/bin/zabbix_sender)

Authentication:
  -cred CRED            Credential file (default: /etc/zabbix/wmi.pw)
                        Contain values per line: username, password, domain
  -a {ntlm,kerberos}, --auth-type {ntlm,kerberos}
                        Authentication type: ntlm (default) or kerberos
  -dc-ip ip address     IP Address of the domain controller. If ommited it use
                        the domain part (FQDN) specified in the target
                        parameter
  -rpc-auth-level [{integrity,privacy,default}]
                        integrity (RPC_C_AUTHN_LEVEL_PKT_INTEGRITY) or privacy
                        (RPC_C_AUTHN_LEVEL_PKT_PRIVACY). (default: default)
```

Credential file consists of three lines: login, password, domain.

## Troubleshooting

### `An error occured: rpc_s_access_denied`

The DCOM/RPC level on the Windows target rejected the connection before any WMI
query ran. Check on the target host:

* the account from the credential file is allowed remote WMI access
  (`wmimgmt.msc` → WMI Control → Security → Root → Permissions: Enable and
  Remote Enable);
* the account has Remote Launch / Remote Activation rights
  (`dcomcnfg` → My Computer → COM Security → Launch and Activation Permissions);
* if the account is a **local** administrator of the target, UAC remote
  restrictions strip its admin token over the network. Use a domain account or
  set the registry value
  `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy`
  to `1` (DWORD) and reboot;
* Windows Firewall allows WMI (RPC TCP/135 plus dynamic RPC ports). Quick check
  from the Zabbix server: `rpcclient`/`wbemtest` from a Windows host;
* login, password and domain in the credential file are correct and the
  password has not expired.

Also verify the usage: the first positional argument is a WMI class, e.g.
`Win32_OperatingSystem`, not the literal string `WMI`:

```sh
$ zbxwmi -action get -fields FreePhysicalMemory Win32_OperatingSystem target
```

## Kerberos

Use `-a kerberos` to authenticate via Kerberos (the ticket is obtained for domain
spesified in the credential file where two first lines may be empty, no TGT caching required):

```sh
$ zbxwmi -a kerberos Win32_LogicalDisk host.domain.com
```

Requirements and notes:

* the target must be an FQDN hostname (e.g. `host.domain.com`), not an IP
  address — the Kerberos ticket is issued for the SPN `HOST/<target>`;
* DNS must resolve the target FQDN;
* `-dc-ip` may be used to point at the domain controller explicitly, otherwise
  the KDC is looked up by the domain part of the target FQDN;
* system time on the Zabbix server must be in sync with the domain controller
  (default Kerberos clock skew is 5 minutes).

## Installation

### Ubuntu / Debian (Zabbix Appliance)

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

### RHEL based distros (Oracle Linux, RHEL, CentOS, Rocky, Alma, Fedora)

The script itself is distro independent — only the package manager, paths and
SELinux need attention.

1. Enable EPEL (Oracle Linux: `dnf install oracle-epel-release-el9` or use the
   built-in `ol9_developer_EPEL` repo; RHEL: `dnf install
   https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm`;
   Rocky/Alma: `dnf install epel-release`).

2. Install impacket with its python dependencies from EPEL:

```sh
# dnf install python3-impacket
```

This pulls in `python3-pyasn1`, `python3-pycryptodome` and `python3-six`
automatically. Alternatively install from PyPI into system python:
`dnf install python3-pip && pip3 install impacket`.

3. Put `zbxwmi` to the Zabbix external scripts directory and set permissions:

```sh
# cd /usr/share/zabbix/externalscripts     # zabbix server package (RHEL path)
# # or /usr/lib/zabbix/externalscripts     # zabbix-appliance and other layouts
# chmod 755 zbxwmi
# chown root.root zbxwmi
```

Check the actual location with: `grep -r ExternalScripts /etc/zabbix/`.

4. Create `/etc/zabbix/wmi.pw` (login, password, domain — one per line) and
   restrict access:

```sh
# chmod 640 /etc/zabbix/wmi.pw
# chown zabbix.zabbix /etc/zabbix/wmi.pw
```

5. SELinux: the zabbix server runs confined, and an external script executing
   python and making outbound network connections is denied by default. Either
   allow it with a local module:

```sh
# dnf install policycoreutils-python-utils
# grep zabbix /var/log/audit/audit.log | audit2allow -M zbxwmi
# semodule -i zbxwmi.pp
```

or (quicker but less strict) make the Zabbix server domain permissive:

```sh
# dnf install policycoreutils-python-utils
# semanage permissive -a zabbix_t
```

If the Zabbix server runs in a Docker container or SELinux is disabled
(`getenforce` returns `Disabled`), skip this step.

6. If the script is invoked as `zabbix` user manually for testing, remember
   that user must be able to read the credential file.


Install [impacket](https://github.com/CoreSecurity/impacket) library.

Download from github or [stripped down](https://13hakta.ru/assets/components/fileattach/connector.php?action=web/download&ctx=web&fid=MDK5dMZwyEHoTNkHGkamjLSs7fIpRXTh) version sufficient to perform WMI calls.
Unpack contents to directory `/usr/lib/python3.6` (check a corresponding version).

Create file /etc/zabbix/wmi.pw with login, password and domain one parameter per line. Set file access:

```sh
# chmod 640 /etc/zabbix/wmi.pw
# chown zabbix.zabbix /etc/zabbix/wmi.pw

```

## Empty values

For string type returned '' (empty string). For numeric type returned '0'.
If a field assumed to be numeric then you should set type hinting for this field.

Let say you have 4 fields: 1 is string, 2 is number, 3 is number, 4 is a string then you must add option: `-type s,n,n,s`

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

`zbxwmi["-action","discover","-type","n","-fields","PercentProcessorTime","-filter","Name<>'_Total'","Win32_PerfFormattedData_PerfOS_Processor",{HOST.HOST}]`

Get disk I/O load:

`zbxwmi["-action","json","-k","Name","-type","n,n,n,n,n","-fields","DiskWritesPersec,DiskWriteBytesPersec,DiskReadsPersec,DiskReadBytesPersec,CurrentDiskQueueLength","-filter","Name='_Total'","Win32_PerfRawData_PerfDisk_LogicalDisk",{HOST.HOST}]`

Get memory load:

`zbxwmi["-action","json","-type","n,n,n","-fields","AvailableBytes,CommitLimit,CommittedBytes","Win32_PerfRawData_PerfOS_Memory",{HOST.HOST}]`

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
