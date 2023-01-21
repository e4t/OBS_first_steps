#  OBS First Steps #

This document provides some of the most relevant commands of the Open Build
Service OBS CLI. This is all you need to get modify test and commit packages
in OBS.

## Create an Account ##

In order to interact with OBS (other than thru the web interface) you will
need an account. In case you haven't done so already, you will have to create
one. The easiest way is to go to the [Build Service home
page](https://build.opensuse.org) and follow the button on the top right
corner labelled
[Sign Up](https://idp-portal.suse.com/univention/self-service/#page=createaccount) .
You will have to chose a user name and password.

## Get the CLI Client ##

The OBS client `osc` is available for a number of Linux distributions,
though not for the Mac OS.
The command to use to install it on your distribution depends on the
distribution you use. Here we show it for a SUSE based distribution.

```
zypper install osc
```
For a number of other distributions installation repositories are
available at:

```
 https://download.opensuse.org/repositories/openSUSE:/Tools/<distribution>
```
You will have to use your distros package management tool to add this
repository to your installation source and install `osc`

Next, you will have to let `osc` know about your account OBS account
and credentials. See next chapter.
Note that `osc` has a build in help. Type `osc help` to get a list of
all commands. To learn more about an individual command, run
`osc <command> --help`.

## Basic Setup ##

In oder to access OBS thru the CLIm you need to create a file in your
home directory `.oscrc` containing:
```
[general]
apisrv = https://api.opensuse.org/
# In case you want to cache package for local builds in a different directory
#packagecachedir = /var/tmp/osbuild-packagecache
# In case you want to test build packages in different path
#build-root = /var/tmp/build-root
# Useful to have packages for debugging in a build chroot
#extra-pkgs = gdb strace less
[https://api.opensuse.org/]
user=<your user name>
pass=<your password>
```

## General Workflow ##

Packages are hosted in so call 'Devel Projects'. Projects in OBS are denoted
by their path (each path element separated by a colon `:`).
Like: `network:cluster`.
With the CLI client `osc` you can check out a package to your local directory:
```
osc co network:cluster/singularity_ce
```
This will create the directory/file `network:cluster/singularity_ce` into
your current directory.
If you want to edit a package - especially if you are not listed as maintainer
of this package, you will have to create a local branch of it in your OBS
'home'. The project path of your home is `home:<username>`.
To create a branch of a package, run `osc branch <project>/<package>` - ie:
```
osc branch network:cluster/singularity_ce
```
Now you can check out the package to your local disk:
```
osc co home:<username>:branches:network:cluster/singularity_ce
```
The files in this directory can now be modified, the package can be built
locally and the modifications can be checked back into the same project
on OBS.

## Package Modification Workflow ##

Perform any modification to the spec file, other files, add or remove files
in the local copy of a package.
To remove files, run
```
osc rm <filename>
```
to add a new file, run
```
osc add <filename>
```
To check whether a file has already been added, run `osc status`.
Once you're done editing, you may want to:
- Test build the results locally:
  run `osc build <repository> <arch>`. `<repository>` refers to the repository
  to build against (usually a distribution version), <arch> the architecture
  to build for (if omitted it will use the architecture you are currently
  running). Note that unless cross building is enabled, you may be able to
  build only for related architecures - like i586 on x86_64.
  To obtain a list of repositories and architectures enabled for your project,
  run `osc results`. This will also show the status of the build of a specific
  project or package (depending if executed from the project or package
  directory).
- Add a changelog entry:
  Since each submission to openSUSE:Factory will require a changelog entry,
  it is a good idea to create one when you are done editing your changes
  - right before checkin. In OBS changelog entries are not added to the
  `%changelog` section of a spec file directly, instead they go to the file
  `<packagename>.changes`. To simplify this, run:
  ```
  osc vc
  ```
  This will open the changes file in your preferred editor adding your name
  and a time stamp. If you want to edit an existing entry (ie not add a new
  time stamp), run
  ```
  osc vc -e
  ```
  When you have updated the package to a new upstream version, you should
  add what has changed compared to the old version. Usually you find this
  information in a file like 'NEWS' or 'CHANGELOG' in the sources.
- Check your results back into OBS:
  ```
  osc ci [-m "Commit Message"]
  ```
  If the commit message is ommitted, `osc ci` will open an editor to add
  the commit message and pre-populate it with the additions to the changes
  file since the last checkin.
- Check build status:
  Once the changes have been checked back into the package, OBS will attempt
  to rebuild each repository/arch specified for the package.
  It's a good idea to check the build results and if the build failed, do
  the necessary fixes.
  Run
  ```
  osc results -v
  ```
  to get the build status. If it says `failed`, something went wrong.
  To find out what caused the failure, run:
  ```
  osc buildlog <repository> <architecture>
  ```
  The information what `<repository>` and `<architecture>` to use can
  be taken from the output of `osc results`: pass it to `osc buildlog`
  in the same order as printed.
  NOTE: The build may fail for repositories especially for older products
  due to missing dependencies. In this case, if you do not care to support
  this older product, you may savely ignore the failure.
  At a minimum, you should make sure the package builds for the
  `openSUSE_Tumbleweed` or `openSUSE_Factory` repository - whichever is
  listed.
  Also, a build may fail for some enterprise product repositories likeL
  `openSUSE_Backports_SLE-15-SP<N>` or `15.<N>`: this is due to the fact
  that a separate build for Leap/PackageHub is not allowed since the package
  is shipped on the enterprise product already. You will see a message like
  this on the output of `osc buildlog`:
  ```
  <package-name>: E: SUSE_Backports_policy-SLE_conflict (Badness: 10000)
  Package with name <package-name> already in SLE
  ```
  In this case, there is nothing to worry about if the corresponding build
  for `SLE_15_SP<N>` succeeds.
- Submit changes back to the development project:
  So far, the changes are only available in the branch (copy) of the package
  in your OBS home. Generally, packages are maintained in their 'devel project',
  only from there, they may be submitted to `openSUSE:Factory`. To submit a
  package back to the project you have forked it from, you may run:
  ```
  osc sr [-m 'Comment']
  ```
  Normally, `osc sr` takes a submission target as argument. In this case
  however, `osc` has recorded the project you have branched from and will
  use this as the default target.
  If you ommit the comment, an editor will be opened and you are able to
  add a comment yourself. If you have updated the changes file since the
  last submission, the editor will be pre-populated with the additions.

## Useful Hints ##

### Download Source Packages ###

You need to specify the name of the source tarball in the RPM spec file.
Optionally, you may specify this as a URL:
```
Source: https://github.com/sylabs/singularity/releases/download/v%{version}/%{name}-%{version}.tar.gz
```
(Here, `%name` is replaced by the argument of the tag `Name:` and `%version`
with the argument of the tag `Version:`. If you specify a URL, you are able
to use the OBS service `download_files`, to fetch the sources for you.
You need to install `download_files`:
```
zypper install obs-service-download_files
```
Once this is downloaded, run
```
zypper service disabledrun download_files
```
to download the source package. If you need to rename the source package
before saving it, you may add the new file name at the end, separated by
'#/':
```
Source: https://github.com/sylabs/singularity/releases/download/v%{version}/%{name}-%{version}.tar.gz#/Singularity_CE-%version.tar.gz
```

## Further Reading ##

This document only provides a few first steps. You will be able to find
more information on how to use OBS here:

  * https://openbuildservice.org/help/manuals/obs-user-guide/art.obs.bg.html
  * https://en.opensuse.org/openSUSE:Build_Service_Tutorial
