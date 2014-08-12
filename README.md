# storage-gen


**WARNING:** This code is under live development RIGHT NOW... this notice will 
removed once it reaches a beta release state. 

**Name:** storage-gen 

**Description:** Generate linux storage structures from minimal definition files 

**Author:** Ethan Schoonover <es@ethanschoonover.com> 

**Bug-Reports:** http://github.com/altercation/storage-gen/issues 

### Summary: 

storage-gen takes a *storage definition file* and processes it into valid 
general purpose linux storage initialization commands for disk partitioning, 
filesystem creation, encryption, etc. It performs basic validation and will 
infer obvious values and sane defaults, unless instructed not to, and then gift 
wraps the output as an executable script for review/use. There is also a useful 
tree view output and live view. 

### Try It: 

Having booted the Arch Linux install medium, download the storage-gen script 
with the following command 

    # curl -L http://links.ethanschoonover.com/storage-gen | zsh 
    # storage-gen --help 

To create a configuration script for a typical new Arch Linux system, you could 
use the following commands (these only output text or tree information, they do 
not execute any changes to the system) 

    # storage-gen new-simple 
    # storage-gen new-simple --tree 

If you had an existing system with, for example, a home partition you wanted to 
keep, while erasing everything else and reinstalling the system, you could use 
this 

    # storage-gen keep-home --tree 



## It does what? 

### This command 

    # storage-gen new-simple --tree 

### Processes this template file 

    filesystem --size 20G --mountpoint / 
    filesystem --mountpoint /home 
    swap 

### Creating this structure automatically 

    [drive] devpath=/dev/sdb 
       │   
       ├──[partition] size=20G devpath=/dev/sdb1 partnum=1 
       │  │   
       │  └──filesystem fstype=ext4 devpath=/dev/sdb1 mountpoint=/ 
       │   
       ├──[partition] size=ram devpath=/dev/sdb2 partnum=2 
       │  │   
       │  └──swap devpath=/dev/sdb2 
       │   
       └──[partition] size=max devpath=/dev/sdb3 partnum=3 
          │   
          └──filesystem fstype=btrfs mountpoint=/home devpath=/dev/sdb3 

### While this command (without --tree) 

    # storage-gen new-simple 

### Automatically creates this script 

    #!/usr/bin/env zsh 

    # ---------------------------------------------------------- 
    # storage initialization commmands 
    # ---------------------------------------------------------- 

    # Drive formatting and partition structures 
    # ---------------------------------------------------------- 
    sgdisk --new=1:0:+20G  # create fixed size partition 
    sgdisk --new=2:0:16G   # create partition matching system ram size 
    sgdisk --largest-new=3 # create partition filling remaining space 

    # Swap configuration 
    # ---------------------------------------------------------- 
    # Get swap uuid (add --label on swap to use instead) 
    swap_uuid="$(lsblk -no UUID /dev/sdb2)" # no label, get uuid 
    mkswap -U $swap_uuid /dev/sdb2          # make swap device 
    swapon -U $swap_uuid /dev/sdb2          # activate swap device 

    # Filesystem creation 
    # ---------------------------------------------------------- 
    mkfs.ext4 /dev/sdb1           # make filesystem 
    mkfs.btrfs --force /dev/sdb3  # make filesystem 

    # Mount filesystems & create subvolumes 
    # ---------------------------------------------------------- 
    mount defaults,x-mount.mkdir /dev/sdb1 /mnt 
    mount defaults,x-mount.mkdir,compress=lzo,space_cache,autodefrag,inode_ca...

(some comments removed... remove them all with the `--compact` command line 
option) 

### Templates: 

In the above examples, the `new-typical` and `keep-home` command line arguments 
are names of files in the `templates` subdirectory. You can use your own files 
as well, or even use a remote file as long as `curl` can reach it (for example 
`storage-gen http://myserver.net/myfiles/mytemplate`). 

storage-gen templates are intended to be as minimal as possible. They are like 
small text-format outlines, each line of which is a storge element which in 
turn my have certain values associated with it. For example, here is a *very* 
simple template that just formats a drive and makes a filesystem, then mounts 
it under our mountpoint (/mnt by default) for subsequent system installation 

    filesystem --mountpoint / 

Of course we may want to separate out the root and home partitions, and limit 
the amount of space the root partition uses 

    filesystem --mountpoint / --size 20G 
    filesystem --mountpoint /home 

Or create the same thing but using btrfs subvolumes (so no size is necessary as 
they will fall under a single partition) 

    filesystem 
        subvolume --mountpoint / 
        subvolume --mountpoint /home 

Here is a variation that makes an encrypted partition for home and adds in a 
swap partition, which will default to the system ram size 

    filesystem --mountpoint / --size 20G 
    filesystem --mountpoint /home --encrypt 
    swap 

Here is another similar configuration that tells the system to keep the 
existing home partition untouched and to replace the existing system root 
partition with a new installation 

    filesystem --replace --label root --mountpoint / 
    filesystem --keep    --label home --mountpoint /home --encrypt 
    swap 

You can mix in --keep and --replace partitions with the creation of new 
partitions 

    filesystem --keep    --devpath /dev/sda1 --mountpoint /boot 
    filesystem --replace --devpath /dev/sda2 --mountpoint / 
    filesystem --keep    --devpath /dev/sda3 --mountpoint /home --encrypt 
    filesystem --mountpoint /media 

The previous examples would have prompted the user to select the paritions that 
we wished to keep or replace. We can pre-select partitions on the current 
system by providing details about the partitions (code, size, label). Here we 
use labels and code values to help the script figure out which partitions we 
mean 

    filesystem --keep    --code ef00  --mountpoint /boot 
    filesystem --replace --label root --mountpoint / 
    filesystem --keep    --label home --mountpoint /home --encrypt 

Another option would have been to specify a partition number 



    filesystem --keep    --partnum 1 --mountpoint /boot 
    filesystem --replace --partnum 2 --mountpoint / 
    filesystem --keep    --partnum 3 --mountpoint /home --encrypt 

In this partnum example, the script would prompt interactively for a drive to 
be selected. We could add this information directly to the template if we 
wanted to 



    drive --devpath /dev/sda 
        filesystem --keep    --partnum 1 --mountpoint /boot 
        filesystem --replace --partnum 2 --mountpoint / 
        filesystem --keep    --partnum 3 --mountpoint /home --encrypt 

or to each filesystem (and thus the containing parition) 

    filesystem --keep    --devpath /dev/sda1 --mountpoint /boot 
    filesystem --replace --devpath /dev/sda2 --mountpoint / 
    filesystem --keep    --devpath /dev/sda3 --mountpoint /home --encrypt 

or we could have provided this information on the command line 

    storage-gen my-template --drives "/dev/sda,/dev/sdb" 

If the pre-specified drives are not valid for installation, the script will 
prompt for other drives to be selected (prudence). If, however, you are (for 
example) running the script on a machine that is not the installation target, 
you might want to ignore the actual system environment for the time being 

    storage-gen my-template --drives "/dev/sda,/dev/sdb" --ignore 

Finally, if you want the script to then just pick the first available drive and 
skip asking you so many questions, tell it to not query 

    storage-gen my-template --drives "/dev/sda,/dev/sdb" --ignore --noquery 

### Outline Your Storage: 

Storage template files are really just outlines. You can indent (spaces OR 
tabs, not both, and spaces are recommended) in order to provide information 
about which element goes where. For example 

    drive 
        partition --size 200M 
            filesystem --mountpoint /boot --fstype vfat 
        partition 
            filesystem 
                subvolume --mountpoint / 
                subvolume --mountpoint /home 

If you wanted to be lazy about things, the following is equivalent 

        filesystem --size 200M --mountpoint /boot --fstype vfat 
        subvolume --mountpoint / 
        subvolume --mountpoint /home 

A nice tool is the `--tree option` that would show you a structured preview of 
what your template will create 

### Trust but Verify: 

storage-gen is non-destructive. It merely outputs a script that can (and must) 
be reviewed prior to execution. You can run it on your live system safely. 
However the final output script is potentially dangerous in that it makes 
partition changes. It is absolutely your own  responsibility to make sure the 
final script does what you want it to! 

### Requirements: 

Init-storage expects certain linux utilities (lsblk, mount, mkfs.*) as well as 
zsh. Why zsh? Init-storage was written primarily for use on new Arch Linux 
installations, and zsh is the default shell on the Arch install medium. Zsh is 
also a capable and well documented scripting language, albeit it one prone to 
hieroglyphic like parameter expansions. 

### Intended Systems: 

Currently, storage-gen is designed for use on UEFI systems and targets the 
creation of GPT partitioned disks. 

### Usage Example: 

Suppose you want to erase your drive, create a new btrfs file system for your 
system root and also create a new, separate filesystem for your home directory. 
A storage-gen definition file for this might be: 

    filesystem --fstype btrfs --size 30G --mountpoint / 
    filesystem --mountpoint /home 

If this file was named "mystorage" you could then run storage-gen on it 

    # storage-gen mystorage 

And after being asked which of the available drives you wanted to use, the 
following script would be output 

    #!/usr/bin/env zsh 
    sgdisk --zap-all /dev/sda # erase everything! 
    sgdisk --clear /dev/sda   # create new gpt structure 
    sgdisk --new=0:0:+30G     # new partition 
    sgdisk --largest-new=0    # new partition, fills remainder 
    mkfs.ext4 /dev/sda1       # make a filesystem 
    mkfs.ext4 /dev/sda2       # make a filesystem 
    mount defaults,x-mount.mkdir/dev/sda1 /mnt      # mount filesystem 
    mount defaults,x-mount.mkdir/dev/sda2 /mnt/home # mount filesystem 

You could also ask to see a tree style output of the results. This is useful 
while writing storage-definition files so you can confirm the results (for 
example, with an editor in one terminal or virtual console and the tree view in 
another): 

    # storage-gen --tree mystorage 

Or the same without messages-- 

    # storage-gen --tree --quiet mystorage 

Resulting in the following for a drive selection of /dev/sda 

    drive devpath=/dev/sda 
    │   
    ├── partition size=30G devpath=/dev/sda1 
    │   │   
    │   └── filesystem fstype=btrfs mountpoint=/ 
    │   
    └── partition size=max devpath=/dev/sda2 
        │   
        └── filesystem fstype=ext4 mountpoint=/home 

If we wanted do do the same, but use a value of /dev/sdb, without choosing it 
manually we could either insert it into our file: 

    drive --devpath=/dev/sdb 
    filesystem --size 30G --mountpoint / 
    filesystem --mountpoint /home 

or we could append it to the command line using the `--drives` option which 
takes a comma separated list of drives: 

    storage-gen --drives /dev/sdb  mystorage 

### Command Examples: 

Note that several of these examples use the example storage template named 
'new-typical'. This is only one example file. See other samples in the 
'templates' subdirectory, or write your own. 

**Examples:** Ways to reference the storage file - normal script output 

    storage-gen filename-from-storage-subdirectory 
    storage-gen ./relative/path/to/storage-definition-filename 
    storage-gen /full/absolute/path/to/storage-definition-filename 
    storage-gen http://url/of/file/to/download 

**Example:** Show a minimal script output, no messages 

    storage-gen --commentsoff --quiet new-typical 

**Example:** Live tree output with 'watch' 

This is useful for split screen editing, with the storage definition file open 
in an editor on one side and this live view in another. 

    watch -cn1 'storage-gen --tree new-typical' 

**Example:** Save script file and show output together 

    storage-gen new-typical > my-storage-script 

or 

    storage-gen new-typical --output my-storage-script 

See the $SCRIPTROOT/storage subdirectory for a list of prepared storage 
definition files and further details on format.


## Command Line Options

`-a, --advanced `

Like --help, but includes advanced options.

`-c, --compact `

No comments in the final script. Turns off all comments and blank lines in the 
script output. Removing these does not change the functionality of the final 
script.

`-h, --help `

Show summary of help information. Use --usage for expanded help.

`-i, --ignore `

Ignore live system environment. Disregards the current systems drive and 
partition environment. Allows script to create output based on *only* the 
provded template. No interactive queries will be run for missing drive or 
partition information. Drives must either be set in the drive entries in the 
template or using the --drives option. This allows creation of scripts that 
don't match the current system configuration. This is different from full 
environment simulation with the --debug and --debugsource options.

`-l, --list `

List builtin storage-gen templates.

`-n, --noquery `

Skip all interactive queries. Normal script execution results in queries for 
drive and partition selection if those values have not been provided and cannot 
be inferred otherwise. This skips those queries. Use this with the --yes option 
to answer all yes/no queries with a default value.

`-q, --quiet `

No messages or warnings. Only display actual script (or tree) output, skip 
informational messages (the informational messages will still be output in the 
script itself unless the --commentsoff option is selected. With this option, a 
single error message will be output to standard error if the script fails.

`-t, --tree `

Print tree diagram of output. After evaluating the storage template, a tree 
relationship is displayed with colors indicating key values (unless --nocolors 
has been used). Useful in combination with the something like `watch -cn1'. 
This option precludes final script output unless either the --script or 
--output options have also been provided.

`-u, --usage `

Displays detail usage information. Adds a summary of the types of options and 
arguments that are valid for each storage type entry. Redirect to a file or 
pipe through a pager such as less. See also the storage-gen README available in 
the project git repository.

`-C, --coloroff `

Do not use colors in console output.

`-D, --debug `

Turn on debug tracking; always save debug log. This option turns on minimal 
debug tracking and rolling script trace capture (in case of error). This has a 
minimal performance impact so is normally off. The debug log is saved by by 
default if the script encounters an error, and the debug file path will be 
displayed in that case or with the use of this option. This debug file contains 
the block device configuration and the original template file content. It may 
be submitted with bug reports.

`-E, --envdebug `

Dump environment values for debugging.

`-F, --force `

Force the script to always output to console. This happens by default UNLESS 
either the --tree or --output options have been passed on the command line. In 
these cases, passing the --script option restores the visual output of the 
script as well.

`-I, --inferoff `

Turn off inference of missing values.

`-K, --keep `

Keep all existing partitions. Same as setting the --noclobber option on each 
drive. Prevents existing partitions from being affected.

`-L, --listcodes `

List GUID type codes in short and long.

`-M, --multipass `

Output individual passphrase query blocks. If multiple encryption items exist 
storage-gen normally creates a single passphrase query in the final executable 
script. With this option, each encryption device without a preset passphrase 
will receive a separate passphrase query step in the final script. Yes, it's a 
multipass,

`-N, --nomatch `

No matching existing partitions. Normally, if the --keep or --replace option 
has been specified (or inherited) by a partition entry in the storage template 
(or a partition implied by another entry), storage-gen will make best effort to 
match it to an existing partition based on available information in the 
template and on the system itself. See the --usage section on matching for more 
information.

`-R, --readme `

Output usage in README markdown format.

`-V, --verbose `

Display details during processing.

`-W, --warnoff `

Turn off warning query in script output.

`-d, --drives '/dev/sdX,/dev/sdY'`

List of drive paths to use in output. Force the use of drives from a comma 
separated list, for example: --drives '/dev/sda' or --drives 
'/dev/sda,/dev/sdc'. Drives will be assigned to missing devpath values in 
storage definition file using these values, in order. If used in combination 
with the --ignore option, then the drives listed do not need to exist on the 
current system. If the --drives option is used without the --ignore option, 
then the current system is checked for the presence of these drives first.

`-m, --mountroot '/MOUNT/ROOT/PATH'`

Preinstall mountpoint root path. Set mount root directory (if not set, use the 
default '/mnt'). Argument is a valid, absolute path to an existing directory. 
This path is used during system installation as a chroot  and is *not* recorded 
in any fstab.

`-O, --output OUTPUT_FILE_NAME`

Optional output filename. If this option is provided, the final executable 
script will be written to the path provided. If not set, script is displayed on 
standard output and may be saved using redirection.

`-S, --sourcedebug FILENAME`

Simulate environment from debug file. storage-gen will read the captured debug 
output from the provided file and use it to simulate a system environment. 
Simulated values include the block device configuration and the original 
template file content.

`-T, --template TEMPLATE_FILE_PATH`

Alternative syntax for source template. storage-gen normally uses arg 1 (the 
first 'unnamed' argument) on the command line as the name (or path) of the 
template to use. This option is merely a more explicit alternative to that.


## Storage File Definition Overview


### Storage Definition File - Valid Item Options (Fields)

These are the options (fields) that are acceptable for each of the storage types. The 
types listed are the only valid types (drive, partition, etc.).
* `drive:` device ssd
* `partition:` bootable code size partnum new keep replace label
* `logical:` Not yet implemented
* `encryption:` luksformat luksopen pass label encrypt
* `swap:` label
* `filesystem:` fstype mountpoint label
* `subvolume:` mountpoint

### Storage Definition File - Field Descriptions

`--bootable `

Identifies the storage item (drive, partition, filesystem) as a bootable 
device. May not always impact the item initialization.

`--drivekeep `

For drives, prevents any existing partitions from being removed, even without 
explicitly identifying them with noclobber.

`--encrypt `

Used on storage items that may be children of encryption to imply an encryption 
entry in the template file without adding it explicitly.

`--keep `

Keen an existing partition. The partition may be specified by --partnum (1), 
--devpath (/dev/sda1), or a matching unique value such as label, code, or even 
size (assuming and exact match of label or size). If using code to match an 
existing partition, either the sgdisk short code or the long code version of it 
may be used. Run __initializeRulesets --listcodes to list all codes.

`--noclobber `

Prevents changes to the item that matches the devpath specified. For instance, 
if --noclobber is set on a drive entry, and the path of the drive is identified 
as /dev/sda, then no changes will be made to /dev/sda or any child elements of 
/dev/sda.

`--replace `

For partitions, tries to match an existing partition based on partnum, devpath, 
code, label or size and then creates a replacement command (deletes and then 
recreates).

`--ssd `

Identifies a drive as an ssd (if no devpath is specified for a device, you will 
be prompted to select a valid drive from a list and will also be prompted to 
identify is as either a mechanical or ssd drive).

`--code CODE`

The partition GUID code as used by sgdisk. Can be either an sgdisk two-byte hex 
code such as 'ef02', or a full GUID type code such as 
'EBD0A0A2-B9E5-4433-87C0-68B6B72699C7' (see 'man sgdisk' and 'sgdisk -L'). Note 
that this is *not* a UUID.

`--description 'DESCRIPTION'`

A short description that can be reported during script execution. Currently not 
utilized. Standard shell-script style comments prefixed by a # sign are also 
acceptable in the storage definition file and will be ignored by the script.

`--devpath /DEV/PATH`

The device path, for instance /dev/sda for a drive, /dev/sda1 for a partition, 
/dev/mapper/cryptvolume for an open luks device, etc. Not required. If no 
devpath is specified for a drive, you will be prompted to select a valid drive 
when the script executes.

`--fstype FILESYSTEM-NAME`

Filesystem types that this script knows about. Each fstype has a corresponding 
'mkfs.*' command. Known filesystem types on this system include bfs btrfs 
cramfs ext2 ext3 ext4 ext4dev fat jfs minix msdos reiserfs vfat xfs

`--label 'ItemLabel'`

A human readable label for a partition, filesystem, encrypted device, etc.

`--luksformat 'OPTIONS LIST'`

Enclose in single quotes. List of command line options applied by cryptsetup 
when formatting a new dm-crypt device in LUKS mode. For example: --luksformat 
'--cipher aes-xts-plain64 --key-size 256'  Safe defaults are applied if this is 
absent.

`--luksopen 'OPTIONS LIST'`

Enclose in single quotes. List of command line options applied by cryptsetup 
when opening a new or existing dm-crypt device in LUKS mode. For example: 
--luksopen '--cipher aes-xts-plain64 --key-size 256' Safe defaults are applied 
if this is absent.

`--mkfsoptions 'OPTIONS LIST'`

Enclose in single quotes. List of command line options applied to a filesystem 
mkfs command when formatting.

`--mountoptions MOUNT,OPTIONS`

Comma separated list of options to apply to a specific storage device 
containing a filesystem when mounting. Generally not required. For example, the 
default mountoptions value applied to a btrfs filesystem is: '--force${label:+ 
--label }${label:-}'  Note that use of other option values is possible by 
simply using the variable form of the option name (e.g. '$label' for option 
'--label'.

`--mountpoint /MOUNT/POINT`

The absolute path to the mountpoint for a storage item. If not set, then the 
item will not be mounted!

`--partitioning NOT YET IMPLEMENTED`

Type of partitioning. Defaults to 'gpt'. Can be set to gpt, mbr, or btrfs.

`--partnum PARTITION_NUMBER`

This is automatically assigned but may also be specified manually on noclobber 
partitions or on partitions you wish to manually control the order/partition 
number of.

`--pass PASSPHRASE`

Passphrase for a encryption item. Insert obvious security warning about saving 
passphrases in files here.

`--preset PRESETNAME`

Use a preset value for the given type. For example, the preset 'boot' for a 
partition will set the type code, mountpoint, and size to specific values.

`--size SIZE`

Size of a partition. May be specified in either sgdisk friendly absolute size 
format using the following suffixes: kibibytes (K), mebibytes (M), gibibytes 
(G), tebibytes (T), or pebibytes (P), or may be a percentage value which will 
use the specified percent of the *available* space on the drive (excluding any 
partitions preserved using --noclobber) such as '30%'. You may also enter 'max' 
as a value to use the maximum available (remaining) free space after all other 
partitions have been assigned. Using the value 'ram' will similarly set the 
partition size to match the install system memory, useful for partitions that 
contain swap devices. Note that if a partition does contain a swap device 
without a size value assigned to either, a size of 'ram' will be assigned by 
default.

