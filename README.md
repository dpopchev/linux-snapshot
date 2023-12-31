# linux-snapshot

Make snapshot backups using `rsync` into remote server.

## Installation

### Requirements

- rsync
- sshpass
- libnotify: notify-send

### Install

```
git clone --depth 1 https://github.com/dpopchev/linux-snapshot
cd linux-snapshot
make install
```

## Usage

### Setup

#### User wide available

Add `~/.local/bin/dpopchev` in your `PATH`, e.g.

```
PATH="~/.local/bin/dpopchev:$PATH"
```

### Options

Execute with empty parameters

```
snapshot # see short/long option versions
```

### Make local snapshot

To create a snapshot backup of `targetdir` into `destination` just do

```
snapshot -s targetdir -d destinaton
```

After execution in `destination` you will find:

- snapshot of `targetdir` named after time of execution, i.e. `WEEKNUMBER-DAYOFWEEK`
- `latest` pointing to the most recent backup
- unchanged files in between snapshots are hard link copies

In the directory of execution you will see logfile.

### Make remote snapshot

Similar to the local one but with several more flags:

```
snapshot -s targetdir -d destination --is-remote -p sshpass/passfile -u user -h host
```

### Exclude nodes

```
snapshot targetdir -l destinaton -e exclude_list
```

Sample exclusion of directories and files content:

```
# exclude_list
/dev/*
/proc/*
/lost+found
/home/user/.npm
/home/user/.cache
```

### Restore

Execute the command in reverse with source pointing to `latest`.

### Schedule

Lets make snapshots of some user directorie `dir` to the home NAS. We can
implement wrapper script:

```
#!/usr/bin/env bash

SNAPSHOT=~/.local/bin/dpopchev/snapshot
LOGFILE=~/.snapshot.log
SRC=~/snapshot/target/dir
DST=/remote/snapshots
USER=user
HOST=hostname
PASSFILE=~/path/passfile
WIFI_UUID=uuid

active_uuid=$(nmcli --fields uuid,name,type connection show --active \
              | grep wifi \
              | cut -d' ' -f1)

# assure content of directory is stored, e.g. not latest/dir/... but latest/...
[[ -d $SRC ]] && SRC="${SRC%/}/"

[[ $active_uuid == $WIFI_UUID ]] && $SNAPSHOT -s $SRC -d $DST \
                                    --is-remote \
                                    -p $PASSFILE \
                                    -u $USER \
                                    -h $HOST \
                                    -l $LOGFILE
```

We can also schedule runs by making an `crontab` entry.

```
# minute hour day(of month) month day(of week)
0 9,21 * * * DISPLAY=:0 ~/.local/bin/dpopchev/snapshot_dir
```

- `DISPLAY=:0` is needed to redirect `notify-send`
- rise the user execution flag of the script `snapshot_dir`

## Acknowledgment

- [arch wiki](https://wiki.archlinux.org/title/rsync)
- [gentoo wiki](https://wiki.gentoo.org/wiki/Rsync)
