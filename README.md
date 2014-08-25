# storage-gen


**WARNING:** This code is under live development RIGHT NOW... this notice will be 
removed once it reaches a beta release state. 

ARCH BBS THREAD: https://bbs.archlinux.org/viewtopic.php?pid=1450265 

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
use the following commands (these only output text or tree information, **they 
do not execute any changes to the system**) 

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

    [drive] devpath=/dev/sda 
       │   
       ├──[partition] size=20G partnum=1 
       │  │   
       │  └──filesystem fstype=ext4 mountpoint=/ 
       │   
       ├──[partition] size=ram partnum=2 
       │  │   
       │  └──swap  
       │   
       └──[partition] size=max partnum=3 
          │   
          └──filesystem fstype=ext4 mountpoint=/home 

### While this command (without --tree) 

    # storage-gen new-simple 

### Automatically creates this script 

Note that it does **not** automatically run this script. You may save it via 
standard output redirection or using the `--output` command line option. This 
gives you the important opportunity to review the output and make changes to 
either the template or the raw script output itself. 

    #!/usr/bin/env zsh 
    # ---------------------------------------------------------- 
    # Storage initialization commmands 
    # ---------------------------------------------------------- 

    # Drive formatting and partition structures 
    # ---------------------------------------------------------- 
    backupdir=$(mktemp --tmpdir=/tmp -d part-backup-XXX) 
    backupfile="$backupdir/_dev_sda_backup" 
    sgdisk --backup=$backupdir # backup existing partition table 
    print "Partition table backed up to $backupdir" 1>&2 
    sgdisk --zap-all /dev/sda # nuke from orbit, just to be sure 
    sgdisk --mbrtogpt --clear /dev/sda # create new gpt structure 
    sgdisk --new=1:0:+20G /dev/sda # create fixed size partition 
    sgdisk --new=2:0:16G /dev/sda # create partition matching system ram size 
    sgdisk --largest-new=3 /dev/sda # create partition filling remaining space 

    # Swap configuration 
    # ---------------------------------------------------------- 
    # Get swap uuid (add --label on swap to use instead) 
    swap_uuid="$(lsblk -no UUID /dev/sda2)" # get swap uuid if no label 
    mkswap -U $swap_uuid /dev/sda2 # make swap device 
    swapon -U $swap_uuid  /dev/sda2 # activate swap device 

    # Filesystem creation 
    # ---------------------------------------------------------- 
    mkfs.ext4 /dev/sda1  # make filesystem 
    mkfs.ext4 /dev/sda3  # make filesystem 

    # Mount filesystems and subvolumes 
    # ---------------------------------------------------------- 
    opts_ext4=defaults,x-mount.mkdir 
    mount -o $opts_ext4 /dev/sda1 /mnt 
    mount -o $opts_ext4 /dev/sda3 /mnt/home 

(some comments removed... remove them all with the `--compact` command line 
option) 

## Simplifies complex configuration 

Adding encryption can be done implicitly on any acceptable entry 

    filesystem --size 20G --mountpoint / 
    filesystem --mountpoint /home --encrypt 
    swap 

Or can be added explicitly (or mixed, with --encrypt on swap and encryption as 
a parent of filesystem in this example) 

    filesystem --size 20G --mountpoint / 
    encryption 
        filesystem --mountpoint /home 
    swap --encrypt 

Creating the following structure (and appropriate LUKS config in the script) 

    [drive] devpath=/dev/sda 
       │   
       ├──[partition] size=20G partnum=1 
       │  │   
       │  └──filesystem fstype=ext4 mountpoint=/ 
       │   
       ├──[partition] size=ram partnum=2 
       │  │   
       │  └──[encryption]  
       │     │   
       │     └──swap  
       │   
       └──[partition] size=max partnum=3 
          │   
          └──encryption  
             │   
             └──filesystem fstype=ext4 mountpoint=/home 

## Get crazy 

Here is another common template example. This preserves an existing EFI 
partition, (for example on an existing system or in a dual boot configuration) 
mounting it under /boot while creating a new encrypted root and home subvolume 
under a btrfs filesystem which is, itself, not mounted. Note that since there 
is an existing partition that will be preserved, no drive erasure takes place. 
For this example, we look at the script output. 

    # keep-efi template - preserves existing EFI partition 
    # and makes new btrfs subvolume config for system and home 

    partition --keep --code ef00 --mountpoint /boot 
    filesystem --encrypt 
        subvolume --mountpoint / 
        subvolume --mountpoint /home 

You can try this template now and use the same simulated machine environment 
using the following command (run from the root of the storage-gen script 
directory, or adjusting the `--sourcedebug` path to locate the correct file). 

    storage-gen keep-efi \ 
        --sourcedebug extras/environments/surfacepro3.debug \ 
        --yes \ 
        --warnoff 

With the following output resulting 

    #!/usr/bin/env zsh 
    # ---------------------------------------------------------- 
    # Storage initialization commmands 
    # ---------------------------------------------------------- 

    # Drive formatting and partition structures 
    # ---------------------------------------------------------- 
    sgdisk --largest-new=6 /dev/sda # create partition filling remaining space 

    # Encryption 
    # ---------------------------------------------------------- 
    # Query and confirm passphrase 
    while ! ${${${pass::=$( 
    read -Ers "?Passphrase: ")}:#$( 
    read -Ers "?Confirmation: ")}:+false}; 
    do print "Try again"; done 

    print -r $pass | cryptsetup luksFormat /dev/sda6 
    print -r $pass | cryptsetup open /dev/sda6 _dev_sda6  

    # Filesystem creation 
    # ---------------------------------------------------------- 
    mkfs.btrfs --force /dev/mapper/_dev_sda6  # make filesystem 

    # Subvolume creation 
    # ---------------------------------------------------------- 
    [[ -d /tmp/btrfs ]] || mkdir -p /tmp/btrfs 
    mount -t btrfs /dev/mapper/_dev_sda6 /tmp/btrfs 
    btrfs subvolume create /tmp/btrfs/root 
    btrfs subvolume create /tmp/btrfs/home 
    umount /tmp/btrfs 

    # Mount filesystems and subvolumes 
    # ---------------------------------------------------------- 
    opts_btrfs=defaults,x-mount.mkdir,compress=lzo,space_cache,autodefrag,ino...
    mount -t btrfs -o subvol=root,$opts_btrfs /dev/mapper/_dev_sda6 /mnt 
    opts_ext4=defaults,x-mount.mkdir 
    mount -o $opts_ext4 -L SYSTEM /mnt/boot 
    mount -t btrfs -o subvol=home,$opts_btrfs /dev/mapper/_dev_sda6 /mnt/home 

## Ok, so what else can it do? 

* Handles many different storage configurations 

* Allows you to identify partitions that you want to keep 

* Allows you to replace partitions with new content 

* Encrypts 

* Understands most of what you throw at it via the templates 

* Knows what you need... you want a filesystem and a swap? It figures 
  out what partitions, etc. 

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

    storage-gen my-template --drives "/dev/sda,/dev/sdb" --just 

Finally, if you want the script to then just pick the first available drive and 
skip asking you so many questions, tell it to not query 

    storage-gen my-template --drives "/dev/sda,/dev/sdb" --just --guess 

### Outline Your Storage: 

Storage template files are really just outlines. You can indent (spaces OR 
tabs, not both, and spaces are recommended) in order to provide information 
about which element goes where. For example 

    drive   # oh yeah, comments are ok 
        partition --size 200M 
            filesystem --mountpoint /boot --fstype vfat 
        partition 
            # filesystem  (this filesystem is commented out 
            #              but the script knows we need one 
            #              there so it will make one) 
                subvolume --mountpoint / 
                subvolume --mountpoint /home 

If you wanted to be lazy about things, the following is equivalent 

        filesystem --size 200M --mountpoint /boot --fstype vfat 
        subvolume --mountpoint / 
        subvolume --mountpoint /home 

A nice tool is the `--tree option` that would show you a structured preview of 
what your template will create 

## Choosing Drives and Partitions 

### You have three ways to select a specific drive or partition: 

1. Embed the information in the template 2. Include the information on the 
command line 3. Let the script prompt you 

### Embedding in a template 

See the `templates/specific-devices` template for this example. You can skip 
the reading in of the current environment with the following command: 

    storage-gen specific-devices --skip 

which will then create a script based **solely** on the drive and partition 
data embedded in the template file below 

    # this template is an example of specifying specific devices 
    # in the template itself. this is a convenient way to prepare 
    # a script on a system other than the target install platform 
    # by running storage-gen with the --skip option (to skip 
    # reading and using the live system environment) 

    drive --devpath /dev/sda 
        partition --partnum 2 --size 20G 
            filesystem --fstype btrfs --mountpoint / 
        partition --partnum 1 --size 200M --code ef00 --keep 
            filesystem --fstype fat --mountpoint /boot 
        partition --partnum 3 
            filesystem --fstype btrfs --mountpoint /home 

### Include the information on the command line 

You could use the same template as above and change the drive to /dev/sdb by 
simply including that on the command line: 

    storage-gen specific-devices --skip --drives /dev/sdb 

### Or you could include multiple drives for a template that had multiple drives: 

    storage-gen multi-drives --skip --drives /dev/sdb,/dev/sdc 

(if you left out the `--skip` option above the drives would be selected and the 
script would then use the actual specified drives on the system to further 
identify partitions, etc.) 

### Let the script prompt you 

Finally, if the script cannot figure out what you want to do, it will prompt 
you based on the current system environment (not possible if using the --skip 
option). 

### Trust but Verify: 

storage-gen is non-destructive. It merely outputs a script that can (and must) 
be reviewed prior to execution. You can run storage-gen on a live system 
safely. However the final output script is potentially dangerous in that it 
makes partition changes. It is absolutely your own responsibility to make sure 
the final script does what you want it to! 

### Requirements: 

Init-storage expects certain linux utilities (lsblk, mount, mkfs.*) as well as 
zsh. An aside: Why zsh? Init-storage was written primarily for use on new Arch 
Linux installations, and zsh is the default shell on the Arch install medium. 
Zsh is a capable and well documented scripting language, albeit it one prone to 
hieroglyphic like parameter expansions. Regardless, something of this scope may 
be best rewritten in another language. 

### Intended Systems: 

Currently, storage-gen is designed for use on UEFI systems and targets the 
creation of GPT partitioned disks. 

### Command Examples: 

Note that several of these examples use the example storage template named 
`new-typical`. This is only one example file. See other samples in the 
`templates` subdirectory, or write your own. 

**Examples:** Ways to reference the storage file - normal script output 

    storage-gen filename-from-templates-subdirectory 
    storage-gen ./relative/path/to/storage-definition-filename 
    storage-gen /full/absolute/path/to/storage-definition-filename 
    storage-gen http://url/of/file/to/download 

**Example:** Show a minimal script output, no messages 

    storage-gen --commentsoff --quiet new-typical 

**Example:** Live tree output with `watch` 

This is useful for split screen editing, with the storage definition file open 
in an editor on one side and this live view in another. 

    watch -cn1 `storage-gen --quiet --tree new-typical` 

**Example:** Save script file and show output together 

    storage-gen new-typical > my-storage-script 

or 

    storage-gen new-typical --output my-storage-script 

See the storage-gen/templates subdirectory for a list of prepared storage 
template files. 

## Existing Partitions 

You may wish to keep an existing partition (and whatever it contains) as it is. 
For example, you might have an existing filesystem that you want to mount under 
/home. To do this, just add the --keep option 

    filesystem --mountpoint /home --keep 

or (equivalent) 

    partition --keep --label MyHome 
        filesystem --mountpoint /home 

To replace an existing partition with a new, equally sized/numbered partition, 
use the --replace option in the template: 

    partition --replace --label ExistingLabel 
        filesystem --mountpoint /home 

In both the above cases, if the script finds a partition with the partition 
label "MyHome" or "ExistingLabel", it automatically selects that partition for 
replacement. Storage-gen will try to match existing partitions based on field 
values that it can read from the system such as `--code`, `--label`, `--size`. 
If it cannot locate a matching partition (or too many matches result) it will 
prompt the user for selection of the correct entry (unless the `--yes` option 
has been supplied on the command line which takes the first best option in all 
interactive queries, or fails if too many options are present). 

If you wish to preserve all existing partitions on a drive regardless of having 
no `--keep` or `--replace` options in the template, simply add the `--keep` 
option to the storage-gen command line and no drive formatting commands will be 
output in the final script (and existing partitions will be respected).


## Command Line Options

`-a, --advanced `

Like --help, but includes advanced options.

`-c, --compact `

No comments in the final script. Turns off all comments and blank lines in the 
script output. Removing these does not change the functionality of the final 
script.

`-h, --help `

Show summary of help information. Use --usage for expanded help.

`-l, --list `

List builtin storage-gen templates.

`-q, --quiet `

No messages or warnings. Only display actual script (or tree) output, skip 
informational messages (the informational messages will still be output in the 
script itself unless the --commentsoff option is selected. With this option, a 
single error message will be output to standard error if the script fails.

`-s, --skip `

Skip checking the current system. Disregards the current systems drive and 
partition environment. Allows script to create output based on *only* the 
provded template (and any --drives option values provided on the command line). 
If you want a script that isn't valid for the current system (because either 
it's for an entirely different system configuration or because the target 
installation target is currently mounted), use this option.   Because the 
current system is being ignored, no interactive queries for missing values will 
be run in this mode.

`-t, --tree `

Print tree diagram of output. After evaluating the storage template, a tree 
relationship is displayed with colors indicating key values (unless --nocolors 
has been used). Useful in combination with the something like `watch -cn1'. 
This option precludes final script output unless either the --script or 
--output options have also been provided.

`-u, --usage `

Displays detailed usage information. Adds a summary of the types of options and 
arguments that are valid for each storage type entry. Redirect to a file or 
pipe through a pager such as less. See also the storage-gen README available in 
the project git repository.

`-y, --yes `

No interactive queries (pick the first 'best' answer for potential queries. For 
example, for missing drive entries, pick the first available, unmounted, 
internal drive). This option won't work with the --skip option since --yes 
still requires information about the current system environment.

`-A, --alpha `

Whistle on past the alpha warning.

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

Keep all existing partitions. Prevents disk formatting, even if no actual 
'--keep' or '--replace' options have been set on partitions in the template. 
Without at least one partition with one of those options OR this command line 
option, the final script will include a disk format command. With this option 
on the command line, no disk formatting will occur and existing partitions will 
be preserved (unless --replace has been specified for a partitio).

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

List of drive paths to use for drive items in the template (or *implied* drives 
- you don't need an actual drive entry). This is precisely like adding a drive 
--devpath /dev/sdX entry in the template (or adding a --devpath field to an 
existing drive entry). THIS OVERRIDES EXISTING --devpath VALUES IN THE 
TEMPLATE! Without this option or actual --devpath values in the template file, 
the user will be prompted to select drives from the live system (if then --just 
option hasn't been set and if --guess isn't set).

`-m, --mountroot '/MOUNT/ROOT/PATH'`

Preinstall mount root. Default is /mnt. Set mount root directory (if not set, 
use the default '/mnt'). Argument is a valid, absolute path to an existing 
directory. This path is used during system installation as a chroot  and is 
*not* recorded in any fstab.

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

These are the options (fields) that are acceptable for each of the storage types. 
The types listed are the only valid types (drive, partition, etc.).
* `drive:` devpath ssd
* `partition:` bootable code size partnum new keep replace label uuid
* `logical:` Not yet implemented
* `encryption:` luksformat luksopen pass label encrypt
* `swap:` label
* `filesystem:` fstype mountpoint label
* `subvolume:` mountpoint label

### Storage Definition File - Field Descriptions

`--bootable `

Identifies the storage item (drive, partition, filesystem) as a bootable 
device. May not always impact the item initialization.

`--encrypt `

Used on storage items that may be children of encryption to imply an encryption 
entry in the template file without adding it explicitly.

`--keep `

Keen an existing partition. The partition may be specified by --partnum (1), 
--devpath (/dev/sda1), or a matching unique value such as label, code, or even 
size (assuming and exact match of label or size). If using code to match an 
existing partition, either the sgdisk short code or the long code version of it 
may be used. Run __InitializeRulesets --listcodes to list all codes.

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
partitions preserved using --noclobber) such as '30%' (% IS CURRENTLY 
UNIMPLEMNTED). You may also enter 'max' as a value to use the maximum available 
(remaining) free space after all other partitions have been assigned. Using the 
value 'ram' will similarly set the partition size to match the install system 
memory, useful for partitions that contain swap devices. Note that if a 
partition does contain a swap device without a size value assigned to either, a 
size of 'ram' will be assigned by default.

`--uuid 'UUID'`

A uuid intended to match an existing partition; better to use another, simpler 
field such as code, size, label to match existing partitions, but this is here 
if you really like entering long hex code values (and for internal use).

