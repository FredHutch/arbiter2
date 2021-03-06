# Installing Arbiter2
Arbiter2 is designed for CentOS 7 systems at the University of Utah Center for High Performance Computing (CHPC). It can run in other environments, provided the operating system uses a version of systemd with cgroups v1 (or a "hybrid" hierarchy, which will work with some functionality restrictions). In general, the software can be installed by

1. installing a recent version of Python 3 (version 3.6 or higher for the current release);
2. installing the required Python packages (such as matplotlib, toml, and requests);
3. configuring a user to run the service, such as root or a user with limited superuser privileges;
4. verifying the configuration file and any site-specific variables or functions for correctness;
5. testing the configurations and installations by running the scripts directly; and
6. setting up a service to run the scripts.

This entire process is facilitated by the setup.sh script, which interactively asks you questions. Although it is fully functional, we recommend it be used only as a reference which can be modified or pulled from to install Arbiter2.

## Installing requisite software
To install Arbiter2, you must first install some additional software (including Python 3, toml, and matplotlib). The following examples assume you are using a RHEL-like Linux distribution. Other distributions should also work, provided they are using systemd, with minor adjustments to installation procedures.

### Installing Python
Python 3 (version 3.6 or higher) is required to run Arbiter2.

#### Option 1: Installing from source
There may be more recent versions of Python available. Check the [Python website](https://www.python.org/) before proceeding.

```bash
# Get the source for Python 3.7 (instructions updated January 2019; check for updates)
# Arbiter2 is also compatible with Python 3.6
wget https://www.python.org/ftp/python/3.7.2/Python-3.7.2.tar.xz
tar xJf Python-3.7.2.tar.xz  # Creates ./Python-3.7.2
cd Python-3.7.2
# Specify the installation directory for Python
./configure --prefix=/absolute/path/to/where/you/want/python
make -j 4
make install
```

#### Option 2: Installing from repositories
It's likely a recent version of Python 3.6 (or a more recent version of Python 3) is available as a package for your system. Note that the name of the executable may be different in this case and will need to be updated in setup.sh.

```bash
yum install python36  # Executable will likely be named python36
```

### Installing external modules
The `matplotlib` and `toml` external modules are required for Arbiter2 to function. At CHPC, we use `requests` in etc/integrations.py to fetch non-default email addresses. `matplotlib` is used to create plots of usage in email messages, while `toml` is used to read configuration files.

The executable for Python may differ (depending on the installation method above). Try, for example, `python3` and `python36`.

```bash
python3 -m ensurepip --default-pip
pip3 install matplotlib toml requests  # requests may not be necessary
```

_Note: User-based Python module installations will also work (e.g. using the --user flag with pip), provided that the pip command is run by the user Arbiter2 will run under (See below for details on that user)_

### Acquiring the Arbiter2 source files
Git is the primary tool used to acquire the source for Arbiter2. Optionally, you can download the source files directly from the repository in a web browser, though updates will be more difficult. (The process can be streamlined by setting up logs and other transient files outside the Arbiter2 directory. Configuration files, however, will still need to be maintained manually.)

```bash
yum install git
```
Acquire the source of Arbiter2 with Git.

```bash
git clone https://gitlab.chpc.utah.edu/arbiter2/arbiter2.git optional-destination-directory
```

## Setting up a user to run the script
Arbiter2 requires limited access to files generally managed by root. In the interest of security, a system account with special sudo permissions is recommended for deployment (Arbiter2 can still be run as root without any additional runtime flags). This can be used via the `User=` option in the systemd servce for running the script. If the user is not root, the `-s`, or `--sudo`, flag is required to indicate to Arbiter2 that it should make sudo calls to modify things (see below on how to set this up). On CHPC systems, the "arbiter" user is already present and can be used for running the script. On other systems, a new user should be created.

```bash
# Create user with:
# -M (no home dir)
# -r (create system account)
# -s /bin/false (cannot log in)
useradd -M -r -s /bin/false -c "System account for Arbiter2" "arbiter"
```

### Allowing cgroup files to be edited without root via sudo
Arbiter2 can get around permanently being root by assuming that it can execute certain required calls, allowed by the etc/sudoers and etc/sudoers.d/\* files (and indicated to Arbiter2 with the `-s` or `--sudo` flag). These allowed calls are mostly `chmod` or `chgrp` commands that change the permissions on the cgroup hierarchy to allow Arbiter2 to write to requried files. The sudoers file (which can be directed to /etc/sudoers.d/arbiter2 for the sake of organization) can be generated by the tools/make_sudoers.py tool.

The make_sudoers.py script has several arguments. First, the `--user` (`-u`) flag can be used to set the user who will be able to run the commands. This should be the user you set up above to be associated with the Arbiter2 service. Second, the `--group` (`-g`) flag is used to set the group that will be set as the owner of cgroup files to allow them to be modified. Third, the `--num-digits` (`-n`) flag sets the maximum length of a uid that is allowed. This is defaulted to 15 if the flag isn't provided. Finally, the `--run-uid` (`-r`) flag sets the uid that will be used to turn on cgroup accounting if the `-a` flag is used (see below for details on turning on accounting). If it is not included, the corresponding commands will not be included in the sudoers file, thus the `-a` flag cannot be used. If it is included but no value is set, accounting can be turned on with any user. If it is included and a uid is set, only the command corresponding to the provided uid will be permitted.

```bash
cd tools
python3 make_sudoers.py -u arbiter -g arbiter > /etc/sudoers.d/arbiter2
```

_Note: One drawback of using sudoers is that the /etc/sudoers file does not support regular expressions (and only supports very simple character matching like `[0-9]`). Because of this, sudo expressions like `/bin/chmod user-*.slice` are vulnerable to injection, since they allow for anything to run where the star is. To avoid this, the script generates many lines of the same command, with the only difference being the number of digit patterns (for different sized uids). This makes more liberal wildcards like `*` unnecessary and ensures only approved commands can be run._

## Setting the right permissions for Arbiter2 files
The source files for Arbiter2 should be set to read and execute-only for the user running the scripts. The user should not be able to write to the files. The logs and databases are the only files that must be written to (and the permissions for the logs/ directory should be handled automatically by setup.sh). It is also recommended that most users cannot read the logs/statuses.db database (its location is configurable), as it contains details regarding all users' badness scores.

## Turning on cgroups accounting
On most systems, per-user accounting (which is needed for Arbiter2 to function correctly) can be permanently turned on via the `systemctl set-property user-$UID.slice CPUAccounting=true MemoryAccounting=true` commmand (or until there are no users logged in or the machine is rebooted with `--runtime`), where $UID is a uid of an active user. This command simply turns on accounting for a single user, which also implicitly turns on accounting for its siblings and parents. Please note that existing processes don't automatically land in the new per-user cgroups created by turning on accounting. The corralling processes section will cover this.

### Required workaround for systemd versions below v240 (RHEL7/CentOS7)
The permanent nature of the command above [seems to be partially broken](https://github.com/systemd/systemd/issues/9512) on systemd versions below v240, e.g. CentOS 7/RHEL 7. On affected machines, accounting will turn off if there are no users logged in or if the machine is rebooted (as if `--runtime` is provided). To get around this, you can have two options:

1. _(Recommended)_ Set up the Arbiter2 systemd service to run as a user in `user.slice` and enable accounting for itself, implicitly turning on accounting for all other users. This is partially done in the example `arbiter2.slice` (You need to add arbiter's uid in the service) and won't affect users who are using systemd v240+.
2. Use the `--accounting UID` (`-a UID`) flag, which creates a permanent user cgroup for the given uid with accounting enabled for it, implicitly turning on accounting for all other users. Because of the transient nature of systemd user cgroups, this user cannot log out (provided they log in) or else the cgroup will disappear, leading to no users being on the machine and accounting being turned off. This should be used over enabling accounting in the systemd service in cases where Arbiter2 is not run as a systemd service. This requires that the user Arbiter2 is run under has a sudoers entry for the corresponding command (generated via the -r flag in `tools/make_sudoers.py`), assuming that user isn't root.

### Corralling processes
When you first turn on cgroups accounting for user slices (for most setups, this is done with the `CPUAccounting`/`MemoryAccounting` lines in Arbiter2's service file), you effectively tell systemd to start placing new user sessions into their own user cgroup (after this is done, child processes spawned by that session are automatically put in the cgroup by the kernel). This will always be done so long as accounting is kept on. That being said, systemd doesn't move existing user sessions and their processes to their proper user cgroup when you enable accounting. **Such processes are effectively invisible to Arbiter2** (and as such will not accrue badness: child processes of the shells, which are not associated with particular cgroup slices, will not be counted). [A tool to "corral" processes](tools/allusers_corraller.sh) is provided to move existing user sessions and their processes to the appropriate cgroup. Another approach can be to restart the node, but make sure the Arbiter2 service is enabled to start again on reboot (or another mechanism will turn on accounting on startup).

```bash
cd tools/
sudo ./allusers_corraller.sh  # Must be root to move all the processes
```

## Checking the configuration
The configuration files used to tweak Arbiter2 are [toml](https://github.com/toml-lang/toml) files. The documentation for configuration is located at [CONFIG.md](CONFIG.md). A default configuration is also provided at ../etc/config.toml. The configuration can be checked with the tools/cfgparser.py tool, which allows you to either check your configurations (they cascade, allowing for simple overrides), or print the resulting config. Please check and modify ./etc/config.toml (and any other configuration files that are relevant) and verify that the options are correct using [CONFIG.md](CONFIG.md). Other changes to site-specific functions (such as email lookups) may be necessary; see below.

### Testing emails
To ensure the configured mail server and administrator emails are set up properly, you can use the with [tools/test\_email.py](tools/test_email.py) tool. The email configuration-testing tool requires the `-e` flag (the etc/ directory) and the `-g` flag (the configuration files). By default, the configurations will be used to determine the message headers and contents. It is possible to overwrite most options with command-line arguments (see them with `test_email.py --help`).

```bash
python3 test_email.py -e ../etc/ -g ../etc/config.toml  # from the tools/ directory
```

## Integrating things a bit more
There are some aspects of Arbiter2 that can be easily changed, such as getting proper email addresses and formatting emails, without diving into the entire codebase. These tweaks are in etc/integrations.py. The functions in the Python file are called inside of Arbiter2 and used for various things, such as the formatting of emails.

## Running the script
Arbiter2 should be run as a system service. This simplifies management and logging. There is an example service file called [arbiter2.service](arbiter2.service). The key parts to change are shown in the service with the "TODO:" prefix. Part of this is making sure that the [arguments](#notable_args) are correct. If a system account is used, that user should be assigned via the `User=` setting and the `Slice=` uid should be changed, as well as making sure that the `-s` or `--sudo` flag is provided (and sudoers is set up). Notably, there is also a `CAP_SYS_PTRACE` [capability](http://man7.org/linux/man-pages/man7/capabilities.7.html) assigned to the service, which allows Arbiter2 to read /proc/\<pid\>/smaps to get PSS memory values. See below for more details.

When the service file has been created and the files are in place, run `systemctl daemon-reload` and then `systemctl start arbiter2` to start the service. (You can also `enable` it when you've finished testing.) Check the status with `systemctl status arbiter2`.

### <span id="notable_args"></span>Notable arguments for Arbiter2
If you want to change the location of etc/, you can do so with the `-e` flag. Logging can be controlled with the `-p`/`-q`/`-v` flags. See the logging section below for details. Most importantly, the `-g` flag controls the configuration(s) provided to Arbiter2. See `python3 arbiter.py --help` for all options.

#### Exiting when a file is touched
The purpose of the optional `--exit-file` flag is to allow the easy updating of Arbiter2 across multiple nodes. When the specified file (which must be owned by the Arbiter2 groupname [the groupname is set in the config]) is touched after Arbiter2 starts, Arbiter2 will exit with a specific error code that can be caught in each Arbiter2 systemd service across nodes and force a restart of the service. If set up correctly, multiple Arbiter2 instances can look for the same file on a network file system and all restart. This enables adminstrators (and potentially user services teams if permissions are set correctly) to change things like whitelists, or even the entire code, and have all Arbiter2 instances across different nodes update without having to log into each of them individually and restart them there.

### Running directly (for testing purposes)
Arbiter2 should be run directly from the default directory structure by changing to the ./arbiter directory and running `python3 arbiter.py` with appropriate flags (e.g. `python3 arbiter.py -s -a 1019 -g /path/to/config.toml`, which turns on accounting using uid 1019, uses sudo and sets the location of the configuration files).

### Using PSS vs RSS for PID memory collection
Arbiter2 can use either PSS or RSS as its process memory metric (PSS can be used to correctly account for shared memory but requires access to /proc/\<pid\>/smaps). In order to use PSS (which is more accurate), the program or service must have a `CAP_SYS_PTRACE` ambient [capability](http://man7.org/linux/man-pages/man7/capabilities.7.html) (or be run as root), as well as having it enabled in the config `pss = true`. This can be done for the systemd service by adding `AmbientCapabilities=CAP_SYS_PTRACE` to the `[Service]` section of the systemd service file. Ambient capabilities (which are required because Arbiter2 is not a binary) are explained [in this article](https://lwn.net/Articles/636533/). Further resources on why we need ambient capabilities for Python can be found [in this question-and-answer page](https://stackoverflow.com/questions/38321220/script-with-cap-net-bind-service-cant-listen-on-port-80).

## RHEL 7/CentOS 7 Issues
At CHPC, Arbiter2 is deployed on CentOS 7 (kernel 3.10). This causes some problems because there are several bugs and nuances with cgroups. One is the disabling of kernel memory accounting by Red Hat. This prevents Arbiter2 from getting accurate memory data: memory.usage_in_bytes includes kernel memory. Because the feature is disabled, there's no kernel accounting data to subtract and get only the user memory data. As a result, the `--rhel7-compat` flag is available. It causes Arbiter2 to pull memory data from the PIDs inside of the cgroup and pretend that it's from the cgroup.

Other bugs include the `systemctl set-property user-$UID.slice ...` command—which Arbiter2 uses to turn on accounting—that is supposed to turn set cgroup accounting permanently, but in actual use, only turns accounting on until the machine is rebooted, or until there are no users. To fix this, accounting is turned on in Arbiter2's systemd service (or by using the `-a` flag).

## Testing Arbiter2 in a permissive mode
Arbiter2 has a mode called `debug_mode` that stops limits and quotas from being set, as well as sending emails to only the configured administrators. This is useful when Arbiter2 is first deployed for testing out the limits you may want to set. In this mode, any user can run Arbiter2 without any special permissions, as long as `pss = false` (since that requires a `CAP_SYS_PTRACE` capability). PSS can easily be disabled by appending the ../etc/_noperms.toml partial config to the end of the config list (`-g` or `--config`) when starting Arbiter2.

## Logging
By default, Arbiter2 automatically logs various things like its startup configuration and permission checks to stdout. Arbiter2 also logs all non-startup logging events to the rotating debug file and sends violation logging to the rotating service logs (e.g. who has nonzero badness and their corresponding badness score) at `log`. The location of these log files is configurable, but defaults to ../logs/ (with respect to arbiter/arbiter.py). If desired, non-startup logging can also be sent to stdout using the `-p` flag (without any additional flags, this sets the minimum logging level to INFO). The verbosity of this can be controlled with the `-v` and `-q` flags, which sets the minimum logging level to DEBUG and CRITICAL, respectively. If Arbiter2 is run in a service, this output can be viewed using `journalctl`.

### Levels (and what types of messages appear with that level)
| Level | Types of messages | Example |
| --- | --- | --- |
| DEBUG | Anything of significance. | 1000 is new and has status: Status(...) |
| INFO | Operational messages. | User 1000 was put in: penalty2 |
| WARNING | Problems that Arbiter2 is having, but can safely deal with. | Image could not be found or attached to email |
| CRITICAL | Not used. | Not applicable. |
| ERROR | Full stop issue. Arbiter2 will probably exit. | Arbiter2 does not have sufficient permissions to read /proc/\<pid\>/smaps (requires root or CAP_SYS_PTRACE capabilities) with pss = true |

## Debugging issues
If the system service fails (`systemctl status arbiter2`), start with the service output (`journalctl -u arbiter2`). If this failure occurs quickly during startup, it is likely Arbiter2 is failing because there is an issue with the config or with not sufficient permissions. This will be indicated there with the `arbiter_startup` logger printed out to stdout. If that isn't the case, look at the debug logs to try and see what logging took place before exiting. The stacktrace of a failure is typically sent to stderr, meaning `journalctl` is the likely place to look for that. If all else fails, you can try running the program youself. You can also see if it's a permissions issue by checking that it works in debug mode (you can just append ../etc/\_debug.toml to the end of the -g config list) and by disabling PSS (you can also append ../etc/noperms.toml).

In addition to those steps, you may also use the `tools/badsignal.py` tool to check that Arbiter2 is seeing badness on the node. This tool is essentially a mini Arbiter2 that contains just the core collection and badness calculations functions and by default, will print out any badness it sees over a internal. You may also find the `--debug` flag to be useful as it prints out the usage that Arbiter2 is seeing for every user.

```
cd tools/
python3 badsignal.py --help
python3 badsignal.py -g ../etc/config.toml --debug
```

## Viewing action logs
The [logsearch](tools/logsearch/logsearch.md) utility is bundled with the Arbiter2 source and can be used to explore throttling events. It requires [Flask](http://flask.pocoo.org/).

The log viewer is not required to use Arbiter2.
