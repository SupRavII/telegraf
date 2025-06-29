# S.M.A.R.T. Input Plugin

This plugin collects [Self-Monitoring, Analysis and Reporting Technology][smart]
information for storage devices information using the
[`smartmontools`][smartmon] package. This plugin also supports NVMe devices by
using the [`nvme-cli`][nvmecli] package.

> [!NOTE]
> This plugin requires the [`smartmontools`][smartmon] and, for NVMe devices,
> the [`nvme-cli`][nvmecli] packages to be installed on your system. The
> `smartctl` and `nvme` commands must to be executable by Telegraf.

⭐ Telegraf v1.5.0
🏷️ hardware, system
💻 all

[smart]: https://en.wikipedia.org/wiki/Self-Monitoring,_Analysis_and_Reporting_Technology
[nvmecli]: https://github.com/linux-nvme/nvme-cli

## Global configuration options <!-- @/docs/includes/plugin_config.md -->

In addition to the plugin-specific configuration settings, plugins support
additional global and plugin configuration settings. These settings are used to
modify metrics, tags, and field or create aliases and configure ordering, etc.
See the [CONFIGURATION.md][CONFIGURATION.md] for more details.

[CONFIGURATION.md]: ../../../docs/CONFIGURATION.md#plugins

## Configuration

```toml @sample.conf
# Read metrics from storage devices supporting S.M.A.R.T.
[[inputs.smart]]
    ## Optionally specify the path to the smartctl executable
    # path_smartctl = "/usr/bin/smartctl"

    ## Optionally specify the path to the nvme-cli executable
    # path_nvme = "/usr/bin/nvme"

    ## Optionally specify if vendor specific attributes should be propagated for NVMe disk case
    ## ["auto-on"] - automatically find and enable additional vendor specific disk info
    ## ["vendor1", "vendor2", ...] - e.g. "Intel" enable additional Intel specific disk info
    # enable_extensions = ["auto-on"]

    ## On most platforms used cli utilities requires root access.
    ## Setting 'use_sudo' to true will make use of sudo to run smartctl or nvme-cli.
    ## Sudo must be configured to allow the telegraf user to run smartctl or nvme-cli
    ## without a password.
    # use_sudo = false

    ## Adds an extra tag "device_type", which can be used to differentiate
    ## multiple disks behind the same controller (e.g., MegaRAID).
    # tag_with_device_type = false

    ## Skip checking disks in this power mode. Defaults to
    ## "standby" to not wake up disks that have stopped rotating.
    ## See --nocheck in the man pages for smartctl.
    ## smartctl version 5.41 and 5.42 have faulty detection of
    ## power mode and might require changing this value to
    ## "never" depending on your disks.
    # nocheck = "standby"

    ## Gather all returned S.M.A.R.T. attribute metrics and the detailed
    ## information from each drive into the 'smart_attribute' measurement.
    # attributes = false

    ## Optionally specify devices to exclude from reporting if disks auto-discovery is performed.
    # excludes = [ "/dev/pass6" ]

    ## Optionally specify devices and device type, if unset
    ## a scan (smartctl --scan and smartctl --scan -d nvme) for S.M.A.R.T. devices will be done
    ## and all found will be included except for the excluded in excludes.
    # devices = [ "/dev/ada0 -d atacam", "/dev/nvme0"]

    ## Timeout for the cli command to complete.
    # timeout = "30s"

    ## Optionally call smartctl and nvme-cli with a specific concurrency policy.
    ## By default, smartctl and nvme-cli are called in separate threads (goroutines) to gather disk attributes.
    ## Some devices (e.g. disks in RAID arrays) may have access limitations that require sequential reading of
    ## SMART data - one individual array drive at the time. In such case please set this configuration option
    ## to "sequential" to get readings for all drives.
    ## valid options: concurrent, sequential
    # read_method = "concurrent"
```

### Permissions

It's important to note that this plugin references smartctl and nvme-cli, which
may require additional permissions to execute successfully.  Depending on the
user/group permissions of the telegraf user executing this plugin, you may need
to use sudo.

You will need the following in your telegraf config:

```toml
[[inputs.smart]]
  use_sudo = true
```

You will also need to update your sudoers file:

```bash
$ visudo
# For smartctl add the following lines:
Cmnd_Alias SMARTCTL = /usr/bin/smartctl
telegraf  ALL=(ALL) NOPASSWD: SMARTCTL
Defaults!SMARTCTL !logfile, !syslog, !pam_session

# For nvme-cli add the following lines:
Cmnd_Alias NVME = /path/to/nvme
telegraf  ALL=(ALL) NOPASSWD: NVME
Defaults!NVME !logfile, !syslog, !pam_session
```

To run smartctl or nvme with `sudo` wrapper script can be
created. `path_smartctl` or `path_nvme` in the configuration should be set to
execute this script.

### SMART specific attributes

SMART is a monitoring system included in computer hard disk drives
(HDDs) and solid-state drives (SSDs) that detects and reports on various
indicators of drive reliability, with the intent of enabling the anticipation of
hardware failures.

SMART information is separated between different measurements: `smart_device` is
used for general information, while `smart_attribute` stores the detailed
attribute information if `attributes = true` is enabled in the plugin
configuration.

If no devices are specified, the plugin will scan for SMART devices via the
following command:

```sh
smartctl --scan
```

Metrics will be reported from the following `smartctl` command:

```sh
smartctl --info --attributes --health -n <nocheck> --format=brief <device>
```

This plugin supports`smartmontools` version 5.41 and above, but v. 5.41 and
v. 5.42 might require setting `nocheck`. See the comment in the sample
configuration. Also, NVMe capabilities were introduced in version 6.5.

To enable SMART on a storage device run:

```sh
smartctl -s on <device>
```

### NVMe vendor specific attributes

For NVMe disk type, plugin can use command line utility `nvme-cli`. It has a
feature to easy access a vendor specific attributes. This plugin supports
nmve-cli version 1.5 and above. In case of `nvme-cli` absence NVMe vendor
specific metrics will not be obtained.

Vendor specific SMART metrics for NVMe disks may be reported from the following
`nvme` command:

```sh
nvme <vendor> smart-log-add <device>
```

Note that vendor plugins for `nvme-cli` could require different naming
convention and report format.

To see installed plugin extensions, depended on the nvme-cli version, look at
the bottom of:

```sh
nvme help
```

To gather disk vendor id (vid) `id-ctrl` could be used:

```sh
nvme id-ctrl <device>
```

Association between a vid and company can be found in the
[membership list][members].

Devices affiliation to being NVMe or non NVMe will be determined thanks to:

```sh
smartctl --scan
```

and:

```sh
smartctl --scan -d nvme
```

[members]: https://pcisig.com/membership/member-companies

## Metrics

- smart_device:
  - tags:
    - capacity
    - device
    - device_type (only emitted if `tag_with_device_type` is set to `true`)
    - enabled
    - model
    - serial_no
    - wwn
  - fields:
    - exit_status
    - health_ok
    - media_wearout_indicator
    - percent_lifetime_remain
    - read_error_rate
    - seek_error
    - temp_c
    - udma_crc_errors
    - wear_leveling_count

- smart_attribute:
  - tags:
    - capacity
    - device
    - device_type (only emitted if `tag_with_device_type` is set to `true`)
    - enabled
    - fail
    - flags
    - id
    - model
    - name
    - serial_no
    - wwn
  - fields:
    - exit_status
    - raw_value
    - threshold
    - value
    - worst

### Flags

The interpretation of the tag `flags` is:

- `K` auto-keep
- `C` event count
- `R` error rate
- `S` speed/performance
- `O` updated online
- `P` prefailure warning

### Exit Status

The `exit_status` field captures the exit status of the used cli utilities
command which is defined by a bitmask. For the interpretation of the bitmask see
the man page for smartctl or nvme-cli.

## Device Names

Device names, e.g., `/dev/sda`, are _not persistent_, and may be
subject to change across reboots or system changes. Instead, you can use the
_World Wide Name_ (WWN) or serial number to identify devices. On Linux block
devices can be referenced by the WWN in the following location:
`/dev/disk/by-id/`.

## Troubleshooting

If you expect to see more SMART metrics than this plugin shows, be sure to use a
proper version of smartctl or nvme-cli utility which has the functionality to
gather desired data. Also, check your device capability because not every SMART
metrics are mandatory. For example the number of temperature sensors depends on
the device specification.

If this plugin is not working as expected for your SMART enabled device,
please run these commands and include the output in a bug report:

For non NVMe devices (from smartctl version >= 7.0 this will also return NVMe
devices by default):

```sh
smartctl --scan
```

For NVMe devices:

```sh
smartctl --scan -d nvme
```

Run the following command replacing your configuration setting for NOCHECK and
the DEVICE (name of the device could be taken from the previous command):

```sh
smartctl --info --health --attributes --tolerance=verypermissive --nocheck NOCHECK --format=brief -d DEVICE
```

If you try to gather vendor specific metrics, please provide this command
and replace vendor and device to match your case:

```sh
nvme VENDOR smart-log-add DEVICE
```

If you have specified devices array in configuration file, and Telegraf only
shows data from one device, you should change the plugin configuration to
sequentially gather disk attributes instead of collecting it in separate threads
(goroutines). To do this find in plugin configuration read_method and change it
to sequential:

```toml
    ## Optionally call smartctl and nvme-cli with a specific concurrency policy.
    ## By default, smartctl and nvme-cli are called in separate threads (goroutines) to gather disk attributes.
    ## Some devices (e.g. disks in RAID arrays) may have access limitations that require sequential reading of
    ## SMART data - one individual array drive at the time. In such case please set this configuration option
    ## to "sequential" to get readings for all drives.
    ## valid options: concurrent, sequential
    read_method = "sequential"
```

## Example Output

```text
smart_device,enabled=Enabled,host=mbpro.local,device=rdisk0,model=APPLE\ SSD\ SM0512F,serial_no=S1K5NYCD964433,wwn=5002538655584d30,capacity=500277790720 udma_crc_errors=0i,exit_status=0i,health_ok=true,read_error_rate=0i,temp_c=40i 1502536854000000000
smart_attribute,capacity=500277790720,device=rdisk0,enabled=Enabled,fail=-,flags=-O-RC-,host=mbpro.local,id=199,model=APPLE\ SSD\ SM0512F,name=UDMA_CRC_Error_Count,serial_no=S1K5NYCD964433,wwn=5002538655584d30 exit_status=0i,raw_value=0i,threshold=0i,value=200i,worst=200i 1502536854000000000
smart_attribute,capacity=500277790720,device=rdisk0,enabled=Enabled,fail=-,flags=-O---K,host=mbpro.local,id=199,model=APPLE\ SSD\ SM0512F,name=Unknown_SSD_Attribute,serial_no=S1K5NYCD964433,wwn=5002538655584d30 exit_status=0i,raw_value=0i,threshold=0i,value=100i,worst=100i 1502536854000000000
```
