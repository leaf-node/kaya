# Kaya

Kaya is a BASH frontend for [restic](https://github.com/restic/restic), a
modern incremental backup solution written in Go. Kaya provides functionality
similar to "pull" mode backups, but making use of restic's
[rest-server](https://github.com/restic/rest-server) in append-only mode.

Kaya is **ALPHA** software, and may contain security bugs. Its API may also
change in future releases.

### License

GPLv3-or-later

## Installation

Copy the binaries into place:

    cp kaya         /usr/local/bin/

    cp kaya.conf    /etc/
    chown root:root /etc/kaya.conf
    chmod 0600      /etc/kaya.conf

    scp kaya-client www1.example.com:/usr/local/bin/

    vim /etc/kaya.conf

    mkdir -p /srv/backups/kaya/
    touch    /srv/backups/kaya/.htpasswd

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

Note that the when using `--one-file-system`, you must explicity include the
path for each mounted filesystem you wish to backup. If you forget to add a
mountpoint like `/home`, you could miss important data. The alternative is to
not use that option, but make heavy use of statements like `--exclude /proc`,
etc.

### Extracting data from backup repos

    mkdir ~/mount/

    cd /srv/backups/kaya/www1.example.com/

    restic -p repo-password-keep -r . mount ~/mount/

When you're done:

    fusermount -u ~/mount/

## Design / Security

The main reasons for using restic is that it is easy to deploy, even on older
systems, and it offers the rest-server mode for interaction.

Kaya trusts the central backup server, so it stores the restic repo password in
plaintext in each repo's directory on that server. The password has to be
stored somewhere, and chances are that you don't want to sync another password
every time you deploy a new target machine, nor enter it every time you fetch a
recent snapshot.

With that in mind, the filesystem you back up to should be encrypted with LUKS,
and it should require a password during the boot process, or when mounting
manually.

One of the reasons why the central server initiates backups is so client
machines won't try to send their backups all at the same time, or in a
randomized way.

The other main reason for central coordination is because you may clone a
production machine, and run it as a dev instance, with a different IP address
and domain name. If both machines have the same credentials for pushing their
backup, they'll both end up in the same repo. If you prune backups from time to
time, you would occasionally delete a production backup and leave a dev backup
in palce. If you export snapshots to tape archives, you would possibly archive
a dev backup to tape. Kaya connects to machines via SSH, and based on DNS. It
also sends the needed credentials upon SSH connection to the target machine.
Keeping the domain name associated with the production system avoids this
issue.

The rest-server's `--append-only` mode should prevent infected machines from
deleting their own past backups, but they are still able to push new ones, and
to read past backup history.

Kaya uses SSH tunneling, with a reverse port forward, so the client can talk to
the rest-server on the backup server without the need to rely upon CA
certificates. This is especially useful for older machines that may have an
outdated root certificate store.

