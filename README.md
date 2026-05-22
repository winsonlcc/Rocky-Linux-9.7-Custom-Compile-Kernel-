# Rocky-Linux-9.7-Custom-Compile-Kernel-
This procedure describes how to build and install a custom kernel based on the Rocky Linux 9.7 default kernel source RPM, instead of downloading and building an upstream kernel from kernel.org.

This method is recommended because:
•	the Rocky Linux 9.7 kernel source is aligned with the Rocky/RHEL platform
•	it preserves Rocky kernel packaging and release naming behavior
•	it reduces compatibility risk
•	it allows controlled kernel configuration overrides
•	it allows a custom kernel naming convention such as .<yourNamingConvention>

This procedure has been validated successfully with a working result:
5.14.0-611.55.1.<yourNamingConvention>.el9.x86_64

Important Notes
•	Perform RPM build steps using a normal user account.
•	Use root or sudo only for:
•	package installation
•	repository enablement
•	installing the finished kernel RPMs
•	Do not remove the stock Rocky kernel.
•	Keep at least one known-good stock kernel for rollback.
•	This procedure uses the Rocky kernel SRPM and RPM packaging workflow.
•	Do not use kernel.org upstream tarballs for this task.
•	zonefs was tested and caused a compile failure in this build path, so it was left disabled.
•	Future optimization: exclude rt, rt-debug, and debug builds unless specifically required

-----------------------------------------------------------------------------------------------
Step 1: Verify Rocky Linux version and running kernel
-----------------------------------------------------------------------------------------------

Check OS Version:
cat /etc/os-release

Check running kernel:
uname -r
Example working source/target baseline:
5.14.0-611.47.1.el9_7.x86_64

-----------------------------------------------------------------------------------------------
Step 2: Install build tools and prerequisites
-----------------------------------------------------------------------------------------------

Run as root:
dnf -y groupinstall "Development Tools"

Install required packages:
dnf -y install \
  dnf-plugins-core \
  rpm-build rpmdevtools \
  ncurses-devel openssl-devel \
  elfutils-libelf-devel \
  python3 bc dwarves flex bison \
  wget git perl rsync cpio \
  grubby dracut \
  pesign

Enable CRB:
dnf config-manager --set-enabled crb

Enable Rocky source repository needed for kernel SRPM download:
dnf config-manager --set-enabled baseos-source

Optional verification:
dnf repolist --enabled | grep source

-----------------------------------------------------------------------------------------------
Step 3: Prepare RPM build tree
-----------------------------------------------------------------------------------------------

Run as normal user: 
rpmdev-setuptree

This creates:
~/rpmbuild/

Expected directories:
•	BUILD
•	RPMS
•	SOURCES
•	SPECS
•	SRPMS

-----------------------------------------------------------------------------------------------
Step 4: Download matching Rocky kernel source RPM
-----------------------------------------------------------------------------------------------

Run as normal user:
cd ~/rpmbuild/SRPMS
dnf download --source kernel

Verify the file exists:
ls -lh ~/rpmbuild/SRPMS/kernel-*.src.rpm

Example validated build source:
kernel-5.14.0-611.55.1.el9_7.src.rpm

-----------------------------------------------------------------------------------------------
Step 5: Install the kernel source RPM into the rpmbuild tree
-----------------------------------------------------------------------------------------------

Run as normal user:
rpm -ivh ~/rpmbuild/SRPMS/kernel-5.14.0-611.55.1.el9_7.src.rpm

Warnings about mockbuild / mock can be ignored if installation completes.
Verify:
ls ~/rpmbuild/SPECS
ls ~/rpmbuild/SOURCES | head

Expected:
•	kernel.spec exists
•	source/config files exist under SOURCES

-----------------------------------------------------------------------------------------------
Step 6: Install build dependencies from the kernel spec
-----------------------------------------------------------------------------------------------

Run as root:
dnf builddep -y /home/<user>/rpmbuild/SPECS/kernel.spec

Example:
dnf builddep -y /home/prouser/rpmbuild/SPECS/kernel.spec

-----------------------------------------------------------------------------------------------
Step 7: Back up the original spec file
-----------------------------------------------------------------------------------------------

Run as normal user:
cp ~/rpmbuild/SPECS/kernel.spec ~/rpmbuild/SPECS/kernel.spec.bak

-----------------------------------------------------------------------------------------------
Step 8: Set custom Proteus build ID in the spec
-----------------------------------------------------------------------------------------------

Inspect current buildid lines:
grep -n "buildid" ~/rpmbuild/SPECS/kernel.spec
grep -n "specrelease" ~/rpmbuild/SPECS/kernel.spec

Edit:
vi ~/rpmbuild/SPECS/kernel.spec

Change: 
# define buildid .local
To:
%define buildid .proteus

Verify:
grep -n "buildid" ~/rpmbuild/SPECS/kernel.spec

Expected:
%define buildid .proteus

This ensures the final kernel release includes .proteus

-----------------------------------------------------------------------------------------------
Step 9: Prepare the kernel source tree
-----------------------------------------------------------------------------------------------

Run as normal user:
cd ~/rpmbuild/SPECS
rpmbuild -bp --target=$(uname -m) kernel.spec

Verify the build tree was created:
ls ~/rpmbuild/BUILD

Example:
kernel-5.14.0-611.55.1.el9_7

-----------------------------------------------------------------------------------------------
Step 10: Enter the extracted kernel source tree
-----------------------------------------------------------------------------------------------

Run:
cd ~/rpmbuild/BUILD/kernel-5.14.0-611.55.1.el9_7
ls
cd linux-*
pwd

Expected kernel tree example:
/home/<user>/rpmbuild/BUILD/kernel-5.14.0-611.55.1.el9_7/linux-5.14.0-611.55.1.proteus.el9.x86_64

Step 11: Create a temporary base config from the running kernel

Run inside the kernel source tree:
cp /boot/config-$(uname -r) .config
make olddefconfig

This is useful for temporary inspection and config testing.

-----------------------------------------------------------------------------------------------
Step 12: Temporary interactive config review (optional but useful)
-----------------------------------------------------------------------------------------------

Run:
make menuconfig

For this validated build, the intended settings were:
Enable
CONFIG_BLK_DEV_ZONED=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_BLK_DEV_NVME=m
CONFIG_BLK_DEV_DM=m
CONFIG_DM_ZONED=m
CONFIG_FUSION_FC=m

Disable
CONFIG_USB_SERIAL_FTDI_SIO is not set

Not used** in final validated build
CONFIG_ZONEFS_FS

**zonefs was attempted but caused a compile failure and was therefore disabled in the final working build.

After saving, the .config may be backed up for reference, but it is not the final source of truth for RPM build config overrides.

Optional backup:
cp .config ~/proteus-kernel.config
cp .config ~/rpmbuild/SOURCES/config-proteus-$(date +%F)

-----------------------------------------------------------------------------------------------
Step 13: Use kernel-local for authoritative config overrides
-----------------------------------------------------------------------------------------------

The Rocky kernel spec merges a local override file during build:
~/rpmbuild/SOURCES/kernel-local

Edit this file:
vi ~/rpmbuild/SOURCES/kernel-local

For the validated working build, set it to exactly:
CONFIG_BLK_DEV_ZONED=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_BLK_DEV_NVME=m
CONFIG_BLK_DEV_DM=m
CONFIG_DM_ZONED=m
# CONFIG_ZONEFS_FS is not set
CONFIG_FUSION_FC=m
# CONFIG_FUSION_LAN is not set
# CONFIG_USB_SERIAL_FTDI_SIO is not set

Verify:
sed -n '1,20p' ~/rpmbuild/SOURCES/kernel-local

Important notes
•	CONFIG_FUSION_LAN had to be explicitly set to not set because enabling CONFIG_FUSION_FC required that companion option to be explicitly resolved by the config checker.
•	CONFIG_ZONEFS_FS was left disabled because enabling it caused a compile failure in this Rocky kernel tree.

-----------------------------------------------------------------------------------------------
Step 14: Build the binary RPMs
-----------------------------------------------------------------------------------------------

Run as normal user:
cd ~/rpmbuild/SPECS
rpmbuild -bb --target=$(uname -m) kernel.spec 2>&1 | tee ~/kernel-build.log

This may take a long time. (Estimate 2 to 4 hours)
The validated build produced RPMs successfully and ended with exit 0.
Example with exit 0:
<img width="940" height="182" alt="image" src="https://github.com/user-attachments/assets/74c7b553-a0b6-48ea-b156-f0f1ffa8ab25" />

-----------------------------------------------------------------------------------------------
Step 15: Verify generated RPMs
-----------------------------------------------------------------------------------------------

Run:
find ~/rpmbuild/RPMS/$(uname -m) -maxdepth 1 -type f | sort

Validated successful output included standard kernel packages such as:
kernel-5.14.0-611.55.1.proteus.el9.x86_64.rpm
kernel-core-5.14.0-611.55.1.proteus.el9.x86_64.rpm
kernel-modules-core-5.14.0-611.55.1.proteus.el9.x86_64.rpm
kernel-modules-5.14.0-611.55.1.proteus.el9.x86_64.rpm
kernel-modules-extra-5.14.0-611.55.1.proteus.el9.x86_64.rpm

Many debug and RT RPMs may also be produced, but they are not required for normal installation.

-----------------------------------------------------------------------------------------------
Step 16: Install the standard Proteus kernel RPMs
-----------------------------------------------------------------------------------------------

Run as root:
dnf install -y \
  /home/<user>/rpmbuild/RPMS/x86_64/kernel-5.14.0-611.55.1.proteus.el9.x86_64.rpm \
  /home/<user>/rpmbuild/RPMS/x86_64/kernel-core-5.14.0-611.55.1.proteus.el9.x86_64.rpm \
  /home/<user>/rpmbuild/RPMS/x86_64/kernel-modules-core-5.14.0-611.55.1.proteus.el9.x86_64.rpm \
  /home/<user>/rpmbuild/RPMS/x86_64/kernel-modules-5.14.0-611.55.1.proteus.el9.x86_64.rpm \
  /home/<user>/rpmbuild/RPMS/x86_64/kernel-modules-extra-5.14.0-611.55.1.proteus.el9.x86_64.rpm

Important
kernel-modules-core must be included, otherwise the install will fail with dependency errors.

-----------------------------------------------------------------------------------------------
Step 17: Verify installation before reboot
-----------------------------------------------------------------------------------------------

Run:
rpm -qa | grep '^kernel' | sort

Check Proteus boot files:
ls -lh /boot | grep proteus

Check boot entries:
grubby --info=ALL | grep -E '^index=|^kernel='

Check default kernel:
grubby --default-kernel

Validated successful result:
/boot/vmlinuz-5.14.0-611.55.1.proteus.el9.x86_64

-----------------------------------------------------------------------------------------------
Step 19: Validate booted kernel after reboot
-----------------------------------------------------------------------------------------------

Run:
uname -r
cat /proc/version

Validated working result:
5.14.0-611.55.1.proteus.el9.x86_64

-----------------------------------------------------------------------------------------------
Step 20: Validate final kernel config after reboot
-----------------------------------------------------------------------------------------------

Run:
grep CONFIG_BLK_DEV_ZONED /boot/config-$(uname -r)
grep CONFIG_MQ_IOSCHED_DEADLINE /boot/config-$(uname -r)
grep CONFIG_BLK_DEV_NVME /boot/config-$(uname -r)
grep '^CONFIG_BLK_DEV_DM=' /boot/config-$(uname -r)
grep CONFIG_DM_ZONED /boot/config-$(uname -r)
grep CONFIG_FUSION_FC /boot/config-$(uname -r)
grep CONFIG_USB_SERIAL_FTDI_SIO /boot/config-$(uname -r)

Validated working result:

CONFIG_BLK_DEV_ZONED=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_BLK_DEV_NVME=m
CONFIG_BLK_DEV_DM=m
CONFIG_DM_ZONED=m
CONFIG_FUSION_FC=m
# CONFIG_USB_SERIAL_FTDI_SIO is not set

-----------------------------------------------------------------------------------------------
Step 21: Optional runtime module checks
-----------------------------------------------------------------------------------------------

Example:
lsmod | grep -E 'nvme|dm_|fusion|ftdi'

In the validated boot, dm_mod was loaded.

-----------------------------------------------------------------------------------------------
Step 22: Backup build artifacts for reuse
-----------------------------------------------------------------------------------------------

Do not back up the entire rpmbuild directory unless needed, as it can be very large.
Recommended backup items:
•	built RPMs in ~/rpmbuild/RPMS/x86_64/
•	source RPM in ~/rpmbuild/SRPMS/
•	modified spec file ~/rpmbuild/SPECS/kernel.spec
•	original spec backup ~/rpmbuild/SPECS/kernel.spec.bak
•	config override file ~/rpmbuild/SOURCES/kernel-local
•	saved config ~/proteus-kernel.config
•	build log ~/kernel-build.log

Example backup folder:
mkdir -p ~/proteus-kernel-backup
cp -a ~/rpmbuild/RPMS/x86_64 ~/proteus-kernel-backup/
cp -a ~/rpmbuild/SRPMS ~/proteus-kernel-backup/
cp -a ~/rpmbuild/SPECS/kernel.spec ~/proteus-kernel-backup/
cp -a ~/rpmbuild/SPECS/kernel.spec.bak ~/proteus-kernel-backup/
cp -a ~/rpmbuild/SOURCES/kernel-local ~/proteus-kernel-backup/
cp -a ~/proteus-kernel.config ~/proteus-kernel-backup/
cp -a ~/kernel-build.log ~/proteus-kernel-backup/

Optional archive:
tar -czf ~/proteus-kernel-backup.tar.gz -C ~ proteus-kernel-backup

