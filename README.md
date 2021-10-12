# Kaya

Kaya is a BASH front end for [restic](https://github.com/restic/restic), a
modern incremental backup solution written in Go. Kaya provides centralized
backup functionality via SSH, similar to "pull" mode backups, but with most of
the heavy lifting being done by each client. Kaya makes use of restic's
[rest-server](https://github.com/restic/rest-server) in append-only mode, so in
theory, backed up machines can't delete their past backups, nor see backups for
other machines.

Kaya is **ALPHA** software, and may contain security bugs or other flaws.
Kaya's API is still under development, and may change in future releases.

### License

GPLv3-or-later

## Installation

Copy the binaries into place:

    cp  kaya            /usr/local/bin/
    scp kaya-client     www1.example.com:/usr/local/bin/

    cp kaya.conf        /etc/
    chown root:root     /etc/kaya.conf
    chmod 0600          /etc/kaya.conf
    vim                 /etc/kaya.conf

    mkdir -p    /srv/backups/kaya/
    touch       /srv/backups/kaya/.htpasswd

**Also** install [rest-server](https://github.com/restic/rest-server) on the
backup host, and [restic](https://github.com/restic/restic) on your backup
targets.  Both can be complied into a statically-linked binary, so it's just a
matter of copying the appropriate binaries into the `PATH` of each machine.

## Usage

First you need to start restic's rest-server on the backup host:

    rest-server --path /srv/backups/kaya/ --append-only --private-repos

Now you can initialize or update a backup like so:

    kaya www1.example.com backup -- --one-file-system / /srv /home --exclude /foo

    kaya www1.example.com backup -- / --exclude /proc --exclude /sys

The parameters after `--` are passed to restic on the backup target. (`man
restic-backup`)

Note that the when using `--one-file-system`, you must explicitly include the
path for each mounted file system you wish to backup. If you forget to add a
mount point like `/home`, you could miss important data. The alternative is to
not use that option, but make heavy use of statements like `--exclude /proc`,
etc.

### Extracting data from backup repos

    mkdir ~/mount/

    cd /srv/backups/kaya/www1.example.com/

    restic -p repo-password-keep -r . mount ~/mount/

When you're done, type `^C` or run `fusermount -u ~/mount/`

## Design / Security

The main reasons for using restic is that it is easy to deploy, even on older
systems, and it offers the rest-server mode for interaction.

Kaya trusts the central backup server, so it stores the restic repo password in
plain text in each repo's data directory. That way, it doesn't have to be
synced from the backup server after initializaing the repository, copied to
each backup target, saved in your password list for recoveries, nor quickly
changed after cloning a production server (see below).

With that in mind, the file system you back up to should be encrypted with
LUKS, and it should require a password during the boot process, or when
mounting the backup partition manually.

One of the reasons why the central server initiates backups is so client
machines won't try to send their backups all at the same time, or in a
randomized way.

The other main reason for central coordination is because you may clone a
production machine, and run it as a dev instance, with a different IP address
and domain name. If both machines have the same credentials for pushing their
backup, they would both end up in the same repo. If you then prune backups from
time to time, you would occasionally delete a production backup and leave a dev
backup in place. If you export snapshots to tape archives, then in that
scenario, you would possibly archive a dev backup to tape.

Kaya avoids this issue by connecting via SSH from a backup server to machines
with persistent domain names. It also sends credentials to the target machine
as part of the connection, so the target machine doesn't have to keep track of
them. Authentication and authorization are mediated by SSH keys.

Kaya uses SSH tunneling, with reverse port forwarding, so the client can talk
to the rest-server on the backup server without the need to rely upon CA
certificates. This is especially useful for older machines that may have an
outdated root certificate store.

The rest-server's `--append-only` mode is meant to prevent infected machines
from deleting their own past backups. Target machines are still able to push
new backups, and to read archived data.

## Contributing

If you'd like to contribute to Kaya, feel free to open an issue or pull
request.

