# A VPP Configuration Utility

`vppcfg` is a commandline utility that applies a YAML based configuration file
safely to a running VPP dataplane. It contains a strict syntax and semantic validation,
and a path planner that brings the dataplane from any configuration state safely to any
other configuration state, as defined by these YAML files.

## User Guide

```
usage: vppcfg [-h] [-d] [-q] [-f] {check,dump,plan,apply} ...

positional arguments:
  {check,dump,plan,apply}
    check               check given YAML config for validity (no VPP)
    dump                dump current running VPP configuration (VPP readonly)
    plan                plan changes from current VPP dataplane to target config (VPP readonly)
    apply               apply changes from current VPP dataplane to target config

optional arguments:
  -h, --help            show this help message and exit
  -d, --debug           enable debug logging, default False
  -q, --quiet           be quiet (only warnings/errors), default False
  -f, --force           force progress despite warnings, default False
```

### vppcfg check

The purpose of the **check** module is to read a YAML configuration file and validate
its syntax (using Yamale) and its semantics (using vppcfg's constraints based system).
If the config file is valid, the return value will be 0. If any syntax errors or semantic
constraint violations are found, the return value will be non-zero.

*Note:* There will be no VPP interaction at all in this mode. It is safe to run on a
machine that does not even have VPP installed.

The configuration file (in YAML format) is given by a mandatory `-c/--config` flag, and
optionally a Yamale schema file, given by the `-s/--schema` flag (which will default to
the built-in default schema). Any violations will be shown in the ERROR log. A succesful
run will look like this:

```
$ vppcfg check -c example.yaml && echo OK
[INFO    ] root.main: Loading configfile example.yaml
[INFO    ] vppcfg.config.valid_config: Configuration validated successfully
[INFO    ] root.main: Configuration is valid
OK

$ echo $?
0
```

A failure to validate can be due to one of two main reasons. Firstly, syntax violations
that trip the syntax parser, which can be seen in the output with the tag `yamale:`:

```
$ cat yamale-invalid.yaml 
interfaces:
  GigabitEthernet1/0/0:
    descr: "the proper field name is description"
    mtu: 127

$ vppcfg check -c yamale-invalid.yaml && echo OK
[INFO    ] root.main: Loading configfile yamale-invalid.yaml
[ERROR   ] vppcfg.config.valid_config: yamale: interfaces.GigabitEthernet1/0/0.descr: Unexpected element
[ERROR   ] vppcfg.config.valid_config: yamale: interfaces.GigabitEthernet1/0/0.mtu: 127 is less than 128
[ERROR   ] root.main: Configuration is not valid, bailing
```

Some configurations may be syntactically correct but still can't be applied, because they
might break some constraint or requirement from VPP. For example, an interface that has an
IP address can't be a member in a bridgedomain, or a sub-interface that has an IP address
with an incompatible encapsulation (notably, the lack of `exact-match`).

Semantic violations are mostly self-explanatory, just be aware that one YAML configuration
error may trip multiple validators:

```
$ cat semantic-invalid.yaml
interfaces:
  GigabitEthernet3/0/0:
    sub-interfaces:
      100:
        addresses: [ 192.0.2.1/30 ]
        encapsulation:
          dot1q: 100
  GigabitEthernet3/0/1:
    mtu: 1500
    addresses: [ 10.0.0.1/29 ]

bridgedomains:
  bd1:
    mtu: 9000
    interfaces: [ GigabitEthernet3/0/1 ]

$ vppcfg check -c semantic-invalid.yaml && echo OK
[INFO    ] root.main: Loading configfile semantic-invalid.yaml
[ERROR   ] vppcfg.config.valid_config: sub-interface GigabitEthernet3/0/0.100 has an address but its encapsulation is not exact-match
[ERROR   ] vppcfg.config.valid_config: interface GigabitEthernet3/0/1 is in L2 mode but has an address
[ERROR   ] vppcfg.config.valid_config: bridgedomain bd1 member GigabitEthernet3/0/1 has an address
[ERROR   ] vppcfg.config.valid_config: bridgedomain bd1 member GigabitEthernet3/0/1 has MTU 1500, while bridge has 9000
[ERROR   ] root.main: Configuration is not valid, bailing
```

In general, it's good practice to check the validity of a YAML file before attempting to
offer it for reconciliation. `vppcfg` will make no guarantees in case its input is not
fully valid! For a full write up of the syntax and semantic validation, see
[this post](https://ipng.ch/s/articles/2022/03/27/vppcfg-1.html).

### vppcfg dump

The purpose of the **dump** module is to connect to the VPP dataplane, and retrieve its
state, emitting the configuration as a YAML file. Although it does contact VPP, it
will perform *readonly* operations and never manipulate state in the dataplane, so it
should be safe to run.

If the flag `-o/--output` is given, the resulting YAML is written to that filename, but
if it is not given, the output will be written to stdout. It will return 0 if the connection
to VPP was established and its state successfully dumped to the logs, and non-zero otherwise.

Use of the **dump** command can be done even if the dataplane was configured outside of
`vppcfg`, although some non-supported scenarios (for example, sub-interfaces on loopbacks)
will be flagged as warnings. If warnings or errors are reported, the YAML file cannot be
assumed safe. Conversely, if no warnings/errors are logged, the resulting YAML should be
a good representation of the dataplane state, as far as `vppcfg` is concerned. A good way
to confirm that is to subsequently run the output file back into `vppcfg check`.

```
$ vppcfg dump || echo "Not a hoopy frood"
[ERROR   ] vppcfg.vppapi.readconfig: Could not connect to VPP
[ERROR   ] root.main: Could not retrieve config from VPP
Not a hoopy frood

pim@hippo:~/src/vpp$ make run
DBGvpp# create sub-interfaces GigabitEthernet3/0/0 100
DBGvpp# set interface ip address GigabitEthernet3/0/0.100 2001:db8:1::1/64
DBGvpp# create bridge-domain 10
DBGvpp# set interface l2 bridge HundredGigabitEthernet12/0/0 10

$ vppcfg dump -o vpp.yaml
[INFO    ] vppcfg.vppapi.connect: VPP version is 22.06-rc0~320-g8f60318ac
[INFO    ] vppcfg.vppapi.write: Wrote YAML config to vpp.yaml

$ cat vpp.yaml
bondethernets: {}
bridgedomains:
  bd10:
    description: ''
    interfaces:
    - HundredGigabitEthernet12/0/0
    mtu: 8996
interfaces:
  GigabitEthernet3/0/0:
    description: ''
    mtu: 9000
    sub-interfaces:
      100:
        addresses:
        - 2001:db8:1::1/64
        description: ''
        encapsulation:
          dot1q: 100
          exact-match: true
        mtu: 9000
  GigabitEthernet3/0/1:
    description: ''
    mtu: 9000
  HundredGigabitEthernet12/0/0:
    description: ''
    mtu: 8996
  HundredGigabitEthernet12/0/1:
    description: ''
    mtu: 8996
loopbacks: {}
vxlan_tunnels: {}

$ vppcfg check -c vpp.yaml
[INFO    ] root.main: Loading configfile vpp.yaml
[INFO    ] vppcfg.config.valid_config: Configuration validated successfully
[INFO    ] root.main: Configuration is valid
```

### vppcfg plan

The purpose of the **plan** module, is to read a configuration file given by the `-c/--config`
flag, ensure it is valid (see the **check** module for details), then connect to the running
VPP instance, retrieve its dataplane configuration into an in-memory cache, and plan a path
from the currently running dataplane configuration to the target configuration given in the
YAML file.

*Note*: The planner will read the VPP runtime state exactly once at startup, and it will not
make any changes to the dataplane. This operation is safe to run.

After it reads the YAML target config and the currently running dataplane config from VPP, it
will plan a path to get into the desired target config. It does this in three phases:

***Pruning***: First, vppcfg will ensure all objects do not have attributes which they should not
(eg. IP addresses) and that objects are destroyed that are not needed (ie. have been removed from
the target config). After this phase, I am certain that any object that exists in the dataplane,
both (a) has the right to exist (because it’s in the target configuration), and (b) has the correct
create-time (ie non syncable) attributes.

***Creating***: Next, vppcfg will ensure that all objects that are not yet present (including the
ones that it just removed because they were present but had incorrect attributes), get (re)created
in the right order. After this phase, I am certain that all objects in the dataplane now (a) have
the right to exist (because they are in the target configuration), (b) have the correct attributes,
but newly, also that (c) all objects that are in the target configuration also got created and now
exist in the dataplane.

***Syncing***: Finally, all objects are synchronized with the target configuration (IP addresses,
MTU etc), taking care to shrink children before their parents, and growing parents before their
children (this is for the special case of any given sub-interface’s MTU having to be equal to or
lower than their parent’s MTU).

If no further flags are given, planning output is given to stdout. Optionally an output file
can be specified by calling with the `-o/--output` flag. The contents of the output is
a set of CLI commands that could be pasted into a `vppctl` shell in the order they are presented.
Alternatively, the output file can be consumed by VPP by issuing `vppctl exec <filename>`, noting
that the filename has to be an absolute path.

Users are not encouraged to program VPP this way (see the **apply** module for that), however
for the sake of completeness:

```
$ vppcfg plan -c example.yaml -o example.exec
[INFO    ] root.main: Loading configfile example.yaml
[INFO    ] vppcfg.config.valid_config: Configuration validated successfully
[INFO    ] root.main: Configuration is valid
[INFO    ] vppcfg.vppapi.connect: VPP version is 22.06-rc0~320-g8f60318ac
[INFO    ] vppcfg.reconciler.write: Wrote 78 lines to example.exec
[INFO    ] root.main: Planning succeeded

$ vppctl exec ~/src/vppcfg/example.exec 

$ vppcfg plan -c example.yaml
[INFO    ] root.main: Loading configfile example.yaml
[INFO    ] vppcfg.config.valid_config: Configuration validated successfully
[INFO    ] root.main: Configuration is valid
[INFO    ] vppcfg.vppapi.connect: VPP version is 22.06-rc0~320-g8f60318ac
[INFO    ] vppcfg.reconciler.write: Wrote 0 lines to (stdout)
[INFO    ] root.main: Planning succeeded
```

For an in-depth discussion on path-planning and how `vppcfg` operates, see
[this post](https://ipng.ch/s/articles/2022/04/02/vppcfg-2.html).

### vppcfg apply

Applying state is not (yet) implemented. Don't worry, it's not much work, but this is punted until
developer community feedback is reviewed :-)
