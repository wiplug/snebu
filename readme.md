Overview
========
Snebu is a backup utility that provides many features that are normally found in Enterprise grade backup systems, yet is designed to be simple to install and use.  In most cases, you install the single binary and configuation file on a server, and you can use a simple shell script to perform backups on a number of POSIX style client systems.  All that is needed on the client is a version of the GNU "find" command (min version 4.2), the "tar" command, and a way to communicate to the backup server such as "ssh".

The base strategy on which Snebu operates is "always incremental" full snapshot backups.  At the beginning of a backup session, a complete list of files and metadata (generated via "find") from the system to be backed up is processed, and compared to what is already in the backup catalog database.  Any new file, or existing file with changed metadata (file mod time, size, inode, owner, etc) is placed in a list and sent back to the client.  The client then uses this list as an input to "tar".  The resulting tar file is then piped back into the server, where it is broken apart, meta data recorded in the database, and the individual file contents are compressed and stored on disk.  The result is that you have the equivalent of a full backup (the full state of the client is captured), but only the incremental backup data needs to be sent to the server to be stored.

What makes it different
-----------------------
Snebu is most similar to backup methods that use a combination of a tools such as Rsync to make incremental copies of data, along with hard links to preserve snapshots of past backups (do a search for `snapshot backups using rsync` for articles describing this method).  However, the Rsync method has two major drawbacks:
* Rsync backups don't store files in compressed format on the target volume.  To get compression, you have to use a filesystem that supports block level compression.
* The file metadata (such as file owner / group, etc) are stored in the filesystem.  This requires root access on the target system in order to preserve file ownership information.

Snebu, on the other hand, stores all file metadata in an database backend.  Therefore, when used on a backup server, it can run under a restricted user account instead of root.  Also, to save space, files are compressed on the fly before being stored on the backup server.  To save even more space, files are automatically deduplicated (at the file level) on the backup server.  This means that when backing up a number of similar systems, most system files are only stored once, in addition to being compressed.  Only the metadata that is unique to each server is stored separately.

Now compare Snebu to other traditional backup methods that use a "full weekly / monthly, daily incremental" method.  That particular method requires at least the last full backup, along with all the individual incremental backups in order to get a system restored.  And some systems don't keep track of deleted files, so after restoring the individual incrementals, you end up with a bunch of files you didn't need.  Snebu, on the other hand, stores a snapshot of what your files looked like at the time of the backup, and will construct a restore that looks exactly like that.

Finally, files are sent to the Snebu server in standard TAR format (it extracts metadata and files from the inbound TAR stream in real time), and it synthesizes a TAR file for restores.  So if you are used to using TAR for backups, then Snebu will be a good match.

Examples
--------
#####Quick start guide:
Compile the source using the include Makefile.  Place the binary `snebu` somewhere convenient (such as `/usr/local/bin`), and create a configuration file `/etc/snebu.conf` with the following values:

```
vault = /var/backups/vault
meta = /var/backups/meta
```

The `vault` directory is where the backup data files are placed.  The `meta` directory is the location of the backup catalog.  In this example, an external USB drive is mounted on `/var/backups`.  Adjust to suit your requirements.

Now create a client backup script called `backup.sh` with the following:

```
#!/bin/bash

SNEBU="/usr/local/bin/snebu"
# Automatically include all mounted ext2, ext3, ext4 filesystems
INCLUDE=( $(mount |grep ext[234] |awk '{print $3}') )

# If the backup drive is mounted locally, we want to exclude it
EXCLUDE=( /var/backups )

NAME=$(hostname)
BK_SERIAL=$(date +%s)

DAY_OF_WEEK=$(date +%u)
DAY_OF_MONTH=$(date +%d)

# Set file retention schedule -- Monthly on the 1st of the month,
# Weekly on Saturdays, and Daily otherwise.
if [ ${DAY_OF_MONTH} = 1 ]; then
    RETENTION=monthly
elif [ ${DAY_OF_WEEK} = 6 ]; then
    RETENTION=weekly
else
    RETENTION=daily
fi

FILE_PATTERN="%y\t%#m\t%D\t%i\t%u\t%U\t%g\t%G\t%s\t0\t%T@\t%p\0"
LINK_PATTERN="%y\t%#m\t%D\t%i\t%u\t%U\t%g\t%G\t%s\t0\t%T@\t%p\0%l\0"

# Build Find exclude commands from exclude list
for i in ${EXCLUDE[@]}
do
    findexclude[$((j * 4 + 1))]="-path"
    findexclude[$((j * 4 + 2))]="$i"
    findexclude[$((j * 4 + 3))]="-prune"
    findexclude[$((j * 4 + 4))]="-o"
    ((j++))
done

FINDCMD() {
    find ${INCLUDE[@]} -xdev "${findexclude[@]}" \( -type f -o -type d \) \
    -printf "${FILE_PATTERN}" -o -type l -printf "${LINK_PATTERN}"
}

# Send list of files and metadata to backup
FINDCMD |$SNEBU newbackup --name ${NAME} --retention ${RETENTION} \
    --datestamp ${BK_SERIAL} --null --null-output \
    >/tmp/backup_include.${BK_SERIAL}

# Now create a tar file and send it to the backup process
tar -P --null -T /tmp/backup_include.${BK_SERIAL} -cf - |\
    $SNEBU submitfiles --name ${NAME} --datestamp ${BK_SERIAL}

rm -f /tmp/backup_include.${BK_SERIAL}

########## END of script ##########
```
If you want to adapt this script to run on a client and send the backups to a remote server, adjust the "SNEBU=" command at the top of the script to use ssh along with a private key:

SNEBU="ssh -C backup@backuphost /usr/local/bin/snebu"

Newbackup
---------
#####Usage: snebu newbackup
Takes a tab-delimited input list of files with metadata (filesize, date,
owner, etc.), checks to see which files are already on the backup server,
and returns a list of files that the server doesn't already have.  References
to the files already on the server are recorded in the current backup session.

Required arguments:
 -n | --name=BACKUPNAME)
Usually the name of the host being backed up.

-d | --datestamp=DATE)
The date of the backup (in seconds), also used as the serial number for the backup.
-r | --retention=SCHEDULE)
Expiration class of the backup.  Typically this will be "daily", "weekly", "monthly", "yearly" or "archive".  Any name can be used though.

Optional arguments:

--graft=PATHA=PATHB)
Replace PATHA at the beginning of files with PATHB on input.  Output file paths are unaffected.  Useful for backing up snapshots from a temporary mount point.
 -T | --files-from=FILE)
Get inbound file list from the given file
--null)
Inbound file list is null terminated.  Default.

--not-null)
Inbound file list is newline terminated, and special characters in filenames are escaped.

--null-output)
Output of required files list is null terminated

--not-null-output)
Output of required files list is newline terminated and special characters are escaped.

--full)
Return all file names submitted, regardless if they are in the backup catalog already

The input file list has the following tab delimited fields:
File Type, Mode, Device, Inode, Owner, Owner Number, Group Owner, Group Number, Size, MD5, Date, Filename, SymLink Target

MD5 is optional, if it is 0 then only the rest of the metadata will be examined to determine if the file has changed.

For null terminated input lists, the filename (last field) is followed by a null and the Symlink Target (for symbolic link file types) are again followed by a null.  If the input list is newline terminated, then there is no null between the filename and the link target.

A suitable input list can be generated as follows:
```
find /source/directory \( -type f -o -type d \) -printf \
    "%y\t%#m\t%D\t%i\t%u\t%U\t%g\t%G\t%s\t0\t%T@\t%p\0"\
    -o -type l -printf \
    "%y\t%#m\t%D\t%i\t%u\t%U\t%g\t%G\t%s\t0\t%T@\t%p\0%l\0"
```

Submitfiles
-----------
#####Usage: snebu submitfiles
Processes an input TAR file that was generated with the output of the newbackup subcommand.

Required arguments:
-n | --name=BACKUPNAME)
Set to the same name that was used for "newbackup".

-d | --datestamp)
Set to the same value that was used for "newbackup".

Optional arguments:


Listbackups
-----------
#####Usage: snebu listbackups
List available backup sets.

Required arguments:
*none*

Optional arguments:

no argument)
Lists all systems and backupsets that are available.

-n | --name=NAME)
Lists all backupsets for the named host.

-d | --datestamp=DATE)
Used in conjuction with the -n argument, this identifies a particular backupset, and outputs a list of files that are contained within it.

SEARCHPATTERN)
One or more files can be listed to restrict the output to matching files.

Restore
-------
#####Usage: snebu restore
Generates a TAR file containing files from a given backupset.

Required arguments:

-n | --name=BACKUPNAME)
Set to the same name that was used for "newbackup", also listed in "listbackups".

-d | --datestamp)
Set to the same value that was used for "newbackup", also listed in "listbackups".

SEARCHPATTERN)
Optionally restrict files in the generated to ones matching the given file list.

Expire
------
#####Usage: snebu expire
Removes a given backupset from the backup catalog database.

Required arguments:

-r | --retention=SCHEDULE)
Retention class of the backupsets to remove. (i.e., "daily", "monthly"...)

-a | --age=DAYS
Removes backupsets (that are part of the given retention class) that are more than the given number of days old.

Optional arguments:

-n | --name=BACKUPNAME)
Set to the same name that was used for "newbackup", also listed in "listbackups".

-m | --minkeep=##)
Keep at least the given number of backups (in this retention class), regardless if the --age value matches.  Useful for making sure that the most recent backups are kept in cases where a system hasn't been backed up for some time.  The default value is 3.


Purge
-----
#####Usage: snebu purge
Permanantly removes files from disk storage that are no longer referenced by any backups.  Run this command after running expire.
