# linux-snapshot

Recipes for backup/snapshots.

## Installation

### Requirements

- rsync

### Install

Make a local copy. Change script variables either by edit or while invoking.

## Usage

### snapshot

Library to make snapshot backups using hard links of unchanged files.
Public API is consisted of `make_snapshot`, `restore_latest` and plain CLI.

`make_snapshot` takes a target and destination to create a differentiated snapshot
with same files 'copied' as hard links. It will delete missing files (only from
the most recent copy) and skip files who are newer on the destination. A symbolic
link is created at the destination pointing to the most recent backup.

`restore_latest` takes as first argument the snapshot home and the original
target as second. It will copy the content of the latest snapshot, pointed by
`$LATEST` into the original place appending the `$RESTORED` value.

Invoking from CLI without arguments makes a snapshot backup of `$TARGET` into
`$SNAPSHOTS` while updating the `$LATEST` symbolic link. Pass `--restore` option
to reverse the process.

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

```bash
#!/usr/bin/env bash

SNAPSHOT=${HOME}/.local/bin/dpopchev/snapshot
LOGFILE=${HOME}/.snapshot.log
SRC=${HOME}/snapshot/target/dir
DST=/remote/snapshots
RUSER=user
RHOST=hostname
PASSFILE=${HOME}/path/passfile
WIFI_UUID=uuid

active_uuid=$(nmcli --fields uuid,name,type connection show --active \
              | grep wifi \
              | cut -d' ' -f1)

# assure content of directory is stored, e.g. not latest/dir/... but latest/...
[[ -d $SRC ]] && SRC="${SRC%/}/"

[[ $active_uuid == $WIFI_UUID ]] && $SNAPSHOT -s $SRC -d $DST \
                                    --is-remote \
                                    --should-checksum \
                                    -p $PASSFILE \
                                    -u $RUSER \
                                    -h $RHOST \
                                    -l $LOGFILE
```

We can also schedule runs by making an `crontab` entry.

```
# minute hour day(of month) month day(of week)
0 9,21 * * * DISPLAY=:0 ~/.local/bin/dpopchev/snapshot_dir
```

- `DISPLAY=:0` is needed to redirect `notify-send`
- rise the user execution flag of the script `snapshot_dir`

### Restore

#### Preliminaries

You can make available variables from `Schedule` using

```bash
# use with cautions
while IFS= read -r variable; do
    source <(echo "${variable}")
done < <(sed -rn '/^[A-Z]+=/p' snapshot_dir)
```

- `ssh user@server 'CMD'` executes a command on the `server` as `user` over `ssh`.
- `sshpass -f file CMD` will pass password from `file` to command prompt

#### Snapshots

See available snapshots:

```bash
sshpass -f ${PASSFILE} ssh ${USER}@${HOST} ls -la ${DST}/$(basename ${SRC})
```

#### Restore

Change `latest` to snapshot name if in need.

```bash
sshpass -f "${PASSFILE}" \
    rsync --archive --xattrs --progress --verbose --copy-dirlinks \
    "${RUSER}"@"${RHOST}":"${DST}"/$(basename "${SRC}")/latest \
    "${SRC}"/SNAPSHOT_RESTORED/
```

## Acknowledgment

- [arch wiki](https://wiki.archlinux.org/title/rsync)
- [gentoo wiki](https://wiki.gentoo.org/wiki/Rsync)
