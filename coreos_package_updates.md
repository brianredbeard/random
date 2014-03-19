#Package identification and modification inside of CoreOS

## Important locations and understanding changes to packages
All unmodified portage packages are located in the directory:

    ~/trunk/src/third_party/portage-stable

Any packages which receive patches before being built into the distribution will be here:

    ~/trunk/src/third_party/coreos-overlay

This is helpful for indexing/auditing packages which will be placed into the distro.  Additionally we can diff the ebuilds between what is in portage and what is in the CoreOS overlay.  As a walkthrough of this process let's use `sys-apps/shadow` as an example.

First identify the newest ebuild in **portage-stable**:

```
(cr) ((87de0fa...)) bharrington@mightyrobot ~/trunk/src/scripts $ ls -1 ~/trunk/src/third_party/portage-stable/sys-apps/shadow/*.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.4.2-r6.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.4.3.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.5.1.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.5.1-r1.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.5.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.5-r1.ebuild
/home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.5-r2.ebuild
```

Next, identify the newest ebuild in **coreos-overlay**:

```
    (cr) ((87de0fa...)) bharrington@mightyrobot ~/trunk/src/scripts $ ls -1 ~/trunk/src/third_party/coreos-overlay/sys-apps/shadow/*.ebuild
    /home/bharrington/trunk/src/third_party/coreos-overlay/sys-apps/shadow/shadow-4.1.4.3-r101.ebuild
```

Diff the ebuilds of the same version to see what patches were made:

```
(cr) ((0fa3489...)) bharrington@mightyrobot ~/trunk/src/third_party/coreos-overlay/sys-auth/pambase $ diff /home/bharrington/trunk/src/third_party/portage-stable/sys-apps/shadow/shadow-4.1.4.3.ebuild /home/bharrington/trunk/src/third_party/coreos-overlay/sys-apps/shadow/shadow-4.1.4.3-r101.ebuild
14c14
< IUSE="audit cracklib nls pam selinux skey"
---
> IUSE="audit cracklib nls pam selinux skey symlink-usr"
101c101,103
<     dosym /bin/passwd /usr/bin/passwd
---
> 	if ! use symlink-usr ; then
> 		dosym /bin/passwd /usr/bin/passwd
> 	fi

```

In this case we are making two modifications:

* modifying the [IUSE](http://devmanual.gentoo.org/ebuild-writing/variables/index.html#iuse) flags to add **symlink-usr** (All non-special use flags used by the ebuild, as seen in `emerge -pv` for example).
* Adding a symbolic link **from** /usr/bin/passwd **to** /bin/passwd in the event that the /bin is not symbolically linked to /usr/bin. 

##To update a package in the CoreOS distro which is not patched:

First look at the available packages:

    equery list -o -p pambase

Update the package in **portage-stable**:
 
    ~/trunk/src/scripts/update_ebuilds sys-auth/pambase

Emerge the new file:

    emerge-amd64-usr sys-auth/pambase

Assuming that the build worked correctly, commit the new ebuild file created in `~/trunk/src/third_party/portage-stable/sys-auth/pambase/` to git and generate a pull request.  Optionally one can run `update-ebuilds --commit` to commit in line.


## To update a package in the CoreOS distro which has/needs a patch:

Retrieve the package from rsync (rsync://rsync.gentoo.org/gentoo-portage) or cvs (:pserver:anonymous@anoncvs.gentoo.org:/var/cvsroot) that you wish to update in **coreos-overlay**, then manually update the ebuild file in the corresponding package location within **coreos-overlay**.  You will need to manually apply any patches/changes to the ebuild file to ensure that they are merged with the upstream code. 

If the ebuild downloads source tarballs the `Manifest` will need to be regenerated with `ebuild filename.ebuild digest`.

Test the build by running the command:

    ebuild filename.ebuild

Assuming a successful build commit the changes to git and submit a pull request.


