.. include:: global.rst.inc
.. highlight:: bash
.. _quickstart:

Quick Start
===========

This chapter will get you started with |project_name| and covers
various use cases.

A step by step example
----------------------

.. include:: quickstart_example.rst.inc

Important note about free space
-------------------------------

Before you start creating backups, please make sure that there is *always*
a good amount of free space on the filesystem that has your backup repository
(and also on ~/.cache). A few GB should suffice for most hard-drive sized
repositories. See also :ref:`cache-memory-usage`.

Borg doesn't use space reserved for root on repository disks (even when run as root),
on file systems which do not support this mechanism (e.g. XFS) we recommend to
reserve some space in Borg itself just to be safe by adjusting the
``additional_free_space`` setting in the ``[repository]`` section of a repositories
``config`` file. A good starting point is ``2G``.

If |project_name| runs out of disk space, it tries to free as much space as it
can while aborting the current operation safely, which allows to free more space
by deleting/pruning archives. This mechanism is not bullet-proof in some
circumstances [1]_.

If you *really* run out of disk space, it can be hard or impossible to free space,
because |project_name| needs free space to operate - even to delete backup
archives.

You can use some monitoring process or just include the free space information
in your backup log files (you check them regularly anyway, right?).

Also helpful:

- create a big file as a "space reserve", that you can delete to free space
- if you use LVM: use a LV + a filesystem that you can resize later and have
  some unallocated PEs you can add to the LV.
- consider using quotas
- use `prune` regularly

.. [1] This failsafe can fail in these circumstances:

    - The underlying file system doesn't support statvfs(2), or returns incorrect
      data, or the repository doesn't reside on a single file system
    - Other tasks fill the disk simultaneously
    - Hard quotas (which may not be reflected in statvfs(2))

Automating backups
------------------

The following example script backs up ``/home`` and ``/var/www`` to a remote
server. The script also uses the :ref:`borg_prune` subcommand to maintain a
certain number of old archives:

::

    #!/bin/sh
    # setting this, so the repo does not need to be given on the commandline:
    export BORG_REPO=username@remoteserver.com:backup

    # setting this, so you won't be asked for your passphrase - make sure the
    # script has appropriate owner/group and mode, e.g. root.root 600:
    export BORG_PASSPHRASE=mysecret

    # Backup most important stuff:
    borg create --stats -C lz4 ::'{hostname}-{now:%Y-%m-%d}' \
        /etc                                                 \
        /home                                                \
        /var                                                 \
        --exclude '/home/*/.cache'                           \
        --exclude '*.pyc'

    # Use the `prune` subcommand to maintain 7 daily, 4 weekly and 6 monthly
    # archives of THIS machine. The '{hostname}-' prefix is very important to
    # limit prune's operation to this machine's archives and not apply to
    # other machine's archives also.
    borg prune --list $REPOSITORY --prefix '{hostname}-' \
        --keep-daily=7 --keep-weekly=4 --keep-monthly=6

Pitfalls with shell variables and environment variables
-------------------------------------------------------

This applies to all environment variables you want borg to see, not just
``BORG_PASSPHRASE``. The short explanation is: always ``export`` your variable,
and use single quotes if you're unsure of the details of your shell's expansion
behavior. E.g.::

    export BORG_PASSPHRASE='complicated & long'

This is because ``export`` exposes variables to subprocesses, which borg may be
one of. More on ``export`` can be found in the "ENVIRONMENT" section of the
bash(1) man page.

Beware of how ``sudo`` interacts with environment variables. For example, you
may be surprised that the following ``export`` has no effect on your command::

   export BORG_PASSPHRASE='complicated & long'
   sudo ./yourborgwrapper.sh  # still prompts for password

For more information, see sudo(8) man page. Hint: see ``env_keep`` in
sudoers(5), or try ``sudo BORG_PASSPHRASE='yourphrase' borg`` syntax.

.. Tip::
    To debug what your borg process is actually seeing, find its PID
    (``ps aux|grep borg``) and then look into ``/proc/<PID>/environ``.

.. backup_compression:

Backup compression
------------------

The default is lz4 (very fast, but low compression ratio), but other methods are
supported for different situations.

If you have a fast repo storage and you want minimum CPU usage, no compression::

    $ borg create --compression none /path/to/repo::arch ~

If you have a less fast repo storage and you want a bit more compression (N=0..9,
0 means no compression, 9 means high compression): ::

    $ borg create --compression zlib,N /path/to/repo::arch ~

If you have a very slow repo storage and you want high compression (N=0..9, 0 means
low compression, 9 means high compression): ::

    $ borg create --compression lzma,N /path/to/repo::arch ~

You'll need to experiment a bit to find the best compression for your use case.
Keep an eye on CPU load and throughput.

.. _encrypted_repos:

Repository encryption
---------------------

Repository encryption can be enabled or disabled at repository creation time
(the default is enabled, with `repokey` method)::

    $ borg init --encryption=none|repokey|keyfile PATH

When repository encryption is enabled all data is encrypted using 256-bit AES_
encryption and the integrity and authenticity is verified using `HMAC-SHA256`_.

All data is encrypted on the client before being written to the repository. This
means that an attacker who manages to compromise the host containing an
encrypted archive will not be able to access any of the data, even while the backup
is being made.

|project_name| supports different methods to store the AES and HMAC keys.

``repokey`` mode
    The key is stored inside the repository (in its "config" file).
    Use this mode if you trust in your good passphrase giving you enough
    protection. The repository server never sees the plaintext key.

``keyfile`` mode
    The key is stored on your local disk (in ``~/.config/borg/keys/``).
    Use this mode if you want "passphrase and having-the-key" security.

In both modes, the key is stored in encrypted form and can be only decrypted
by providing the correct passphrase.

For automated backups the passphrase can be specified using the
`BORG_PASSPHRASE` environment variable.

.. note:: Be careful about how you set that environment, see
          :ref:`this note about password environments <password_env>`
          for more information.

.. warning:: The repository data is totally inaccessible without the key
    and the key passphrase.

    Make a backup copy of the key file (``keyfile`` mode) or repo config
    file (``repokey`` mode) and keep it at a safe place, so you still have
    the key in case it gets corrupted or lost. Also keep your passphrase
    at a safe place.

    You can make backups using :ref:`borg_key_export` subcommand.

    If you want to print a backup of your key to paper use the ``--paper``
    option of this command and print the result, or this print `template`_
    if you need a version with QR-Code.

    A backup inside of the backup that is encrypted with that key/passphrase
    won't help you with that, of course.

.. _template: paperkey.html

.. _remote_repos:

Remote repositories
-------------------

|project_name| can initialize and access repositories on remote hosts if the
host is accessible using SSH.  This is fastest and easiest when |project_name|
is installed on the remote host, in which case the following syntax is used::

  $ borg init user@hostname:/path/to/repo

Note: please see the usage chapter for a full documentation of repo URLs.

Remote operations over SSH can be automated with SSH keys. You can restrict the
use of the SSH keypair by prepending a forced command to the SSH public key in
the remote server's `authorized_keys` file. This example will start |project_name|
in server mode and limit it to a specific filesystem path::

  command="borg serve --restrict-to-path /path/to/repo",no-pty,no-agent-forwarding,no-port-forwarding,no-X11-forwarding,no-user-rc ssh-rsa AAAAB3[...]

If it is not possible to install |project_name| on the remote host,
it is still possible to use the remote host to store a repository by
mounting the remote filesystem, for example, using sshfs::

  $ sshfs user@hostname:/path/to /path/to
  $ borg init /path/to/repo
  $ fusermount -u /path/to

You can also use other remote filesystems in a similar way. Just be careful,
not all filesystems out there are really stable and working good enough to
be acceptable for backup usage.
