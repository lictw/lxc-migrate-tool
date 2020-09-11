# LXC Migrate Tool

This is a **huge** bash ~~script~~ tool that helps you to migrate your LXC containers. It can dump or restore single container or all of them, or all with some excludes! Can work with container's disk devices, nic devices, migrate other config options, and all this of course with excellent flexibility. Can rename your containers (and container's name in paths of persistent data!) while dumping or restoring. All of this stores in single .tar.gz archive (of course it's just a default) which can be moved anywhere and restored with *uno-click* script. So, its really a Tool, LXC Migrate Tool!

See how it works:
```
# lxc-migrate-tool --full openvpn
(2020-09-11 21:34:40.990405506) $ mkdir -p /root/lxc-migrate-tool.2020-09-11/containers
(2020-09-11 21:34:40.991227572) $ cd /root/lxc-migrate-tool.2020-09-11/containers
(2020-09-11 21:34:40.993589647) $ lxc stop openvpn 2> /dev/null || true
(2020-09-11 21:34:41.013674238) $ mkdir -p openvpn/data
(2020-09-11 21:34:41.015294910) $ lxc publish openvpn --alias openvpn.2020-09-11 > /dev/null
(2020-09-11 21:34:41.017527917) $ lxc image export openvpn.2020-09-11 openvpn/image > /dev/null
(2020-09-11 21:34:41.018536969) $ lxc image delete openvpn.2020-09-11
(2020-09-11 21:34:41.050549125) $ echo 'boot.autostart true' >> openvpn/config
(2020-09-11 21:34:41.071757752) $ echo 'limits.memory 2GB' >> openvpn/config
(2020-09-11 21:34:41.077076181) $ echo 'limits.memory.enforce soft' >> openvpn/config
(2020-09-11 21:34:41.101589819) $ echo 'eth0' >> openvpn/nic.devices
(2020-09-11 21:34:41.156373441) $ echo '10.94.83.254' >> openvpn/nic.devices
(2020-09-11 21:34:41.252688009) $ echo 'logs' >> openvpn/disk.devices
(2020-09-11 21:34:41.277974667) $ tar -czp -f openvpn/data/logs.tar.gz -C /mnt/data/containers/openvpn/var/vpn .
(2020-09-11 21:34:41.298641965) $ lxc start openvpn > /dev/null
(2020-09-11 21:34:41.299592937) $ cd /root/lxc-migrate-tool.2020-09-11
(2020-09-11 21:34:41.300439292) $ echo '#!/bin/sh' > deploy.sh
(2020-09-11 21:34:41.301496256) $ chmod +x deploy.sh
(2020-09-11 21:34:41.302321189) $ cd /root
(2020-09-11 21:34:41.303250564) $ cp ./lxc-migrate-tool /root/lxc-migrate-tool.2020-09-11/lxc-migrate-tool
(2020-09-11 21:34:41.305231561) $ tar -cz -f /root/lxc-migrate-tool.2020-09-11.tar.gz -C /root/lxc-migrate-tool.2020-09-11 .
(2020-09-11 21:34:41.306090602) $ rm -rf /root/lxc-migrate-tool.2020-09-11
(2020-09-11 21:34:41.309216319) $ mv /root/lxc-migrate-tool.2020-09-11.tar.gz ./2020-09-11.HOSTNAME.lxd-data.tar.gz
```

## Workflow

By default it executes in **dump mode** and doesn't execute any command, it just prints commands to execution! You can execute them by yourself or check that everything is fine, then rerun tool with execution flag. After that dump will perform, but there are two modes:
1. Default one, only persistent data (paths of disk devices) will be saved, with permissions!
1. Saves containers as they are, so snapshot of rootFS, config and data will be saved. This called full dump and `--full` flag enables it.

In any case containers are stopping while dump is running (by one) and after that directory with data somehow gets to a new location where the restore process begins, it's called **deploy mode**. The directory will contain the tool, it always works with the data in its directory, so you can run the tool from any place. This is because at end of dump process the tool copy itself in the dump directory, it's supposed that you will restore containers with the same program that you have dumped them. Also the directory will contain deploy.sh, just run it and all that you dump will be restored, preserving full mode. In principle, in deploy mode everything will be just the opposite, but keep in mind that in full mode containers are not only created, but also completely recreated if container with same name already exists!

## Usage

```
lxc-migrate-tool [FLAGS] [CONTAINERS] [DESTINATION]
```
- `CONTAINERS` - it's all found containers by default, but you can specify them, or you can omit it and exclude some of containers with `--exclude` flag. Works in both modes.
- `DESTINATION` - can be used only in dump mode! It's about where and how dump will be placed, variants:
    - If omitted - archive with default name (TIMESTAMP.HOSTNAME.lxd-data.tar.gz) in current directory.
    - If path with trailing / - archive with default name in specified directory.
    - If path ends at .tar.gz - full path, so it will be.
    - If path without trailing / - also full path, but archive willn't be created.
    - Also pattern like `domain:destination` is possible, then scp command will be used.

Destination always relative by where you are and the destination directory will be created if not exists, but of course this is not true with scp.

### Flags

#### Notes
- All flags must be prefixed with `--` and passed before container's names.
- All flags works in both modes if not specified.
- All exclude options can be duplicated.

#### List
- `-h|--help` shows the built-in usage.
- `deploy` switches to deploy mode.
- `full` switches to full mode (dump/restore all).
- `exec` enables commands execution.
- `disable-execution` disables commands execution. You can pass it to suppress previously passed exec flag, use it for wrappers such the `deploy.sh` to disable execution enabled in them by default.
- `exclude CONTAINER` excludes container, exact by name.
- `exclude-data PATTERN` excludes disk's data, but not disk device by itself in full mode, inexact (grep used) by disk name.
- `exclude-config PATTERN` excludes options from config, inexact by name (bash regexp). Needs to add here that in the principle, only `boot|limits|security` options saves.
- `save-config` saves original config (also only `boot|limits|security`) when container recreating in deploy mode.
- `replace PATTERN` renames containers. Container's name and paths of its disk devices will be passed through `sed -E PATTERN` and new name/paths will be used while dumping/restoring. Never combine this with `exec` flag for the first run, check printing and see what will happen, after that rerun.
- `debug` enables debug messages and disable any destructive command, such deletion commands. Keep in mind that `exec` flag also needed that something executes.
- `no-clean` disables deletion of temporary data.
- `no-commands` disables command printing.
- `no-deploy-script` disables `deploy.sh` creation.
