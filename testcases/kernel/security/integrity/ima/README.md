# IMA + EVM testing

## IMA tests

Mandatory kernel configuration for IMA:
```
CONFIG_INTEGRITY=y
CONFIG_IMA=y
```

### IMA measurement tests
`ima_measurements.sh` require builtin IMA tcb policy to be loaded
(`ima_policy=tcb` kernel parameter).
Although a custom policy, loaded via dracut, systemd or manually from user
space, may contain equivalent measurement tcb rules, detecting them would
require `IMA_READ_POLICY=y` therefore ignore this option.

### IMA key import test
`ima_keys.sh` requires a x509 public key, by default in `/etc/keys/x509_ima.der`
(defined in `CONFIG_IMA_X509_PATH` kernel config option).
The key must be signed by the private key you generate. Follow these instructions:
https://manpages.ubuntu.com/manpages/disco/man1/evmctl.1.html#generate%20trusted%20keys

The test cannot be set-up automatically because the x509 public key must be
built into the kernel and loaded onto a trusted keyring
(e.g. `.builtin_trusted_keys`, `.secondary_trusted_keyring`).

As well as what's required for the IMA tests, the following are also required
in the kernel configuration:
```
CONFIG_IMA_READ_POLICY=y
CONFIG_IMA_X509_PATH="/etc/keys/x509_ima.der"
CONFIG_SYSTEM_TRUSTED_KEYRING=y
CONFIG_SYSTEM_TRUSTED_KEYS="/etc/keys/ima-local-ca.pem"
```

Test also requires loaded policy with `func=KEY_CHECK`, see example in `keycheck.policy`.

### IMA kexec test

`ima_kexec.sh` requires loaded policy which contains `measure func=KEXEC_CMDLINE`,
see example in `kexec.policy`.

The test attempts to kexec the existing running kernel image.
To kexec a different kernel image export `IMA_KEXEC_IMAGE=<pathname>`.

## EVM tests

`evm_overlay.sh` requires a builtin IMA appraise tcb policy (e.g. `ima_policy=appraise_tcb`
kernel parameter) which appraises the integrity of all files owned by root and EVM setup.
Again, for simplicity ignore possibility to load requires rules via custom policy.

Mandatory kernel configuration for IMA & EVM:
```
CONFIG_INTEGRITY=y
CONFIG_INTEGRITY_SIGNATURE=y
CONFIG_IMA=y
CONFIG_IMA_APPRAISE=y
CONFIG_EVM=y
CONFIG_KEYS=y
CONFIG_TRUSTED_KEYS=y
CONFIG_ENCRYPTED_KEYS=y
```

Example of installing IMA + EVM on openSUSE:

* Boot install system with `ima_policy=tcb|appraise_tcb ima_appraise=fix evm=fix` kernel parameters
  (for IMA measurement, IMA appraisal and EVM protection)
* Proceed with installation until summary screen, but do not start the installation yet
* Select package `dracut-ima` (required for early boot EVM support) for installation
  (Debian based distros already contain IMA + EVM support in `dracut` package)
* Change to a console window and run commands to generate keys required by EVM:
```
# mkdir /etc/keys
# user_key=$(keyctl add user kmk-user "`dd if=/dev/urandom bs=1 count=32 2>/dev/null`" @u)
# keyctl pipe "$user_key" > /etc/keys/kmk-user.blob
# evm_key=$(keyctl add encrypted evm-key "new user:kmk-user 64" @u)
# keyctl pipe "$evm_key" >/etc/keys/evm.blob
# cat <<END >/etc/sysconfig/masterkey
MASTERKEYTYPE="user"
MASTERKEY="/etc/keys/kmk-user.blob"
END
# cat <<END >/etc/sysconfig/evm
EVMKEY="/etc/keys/evm.blob"
END
# mount -t securityfs security /sys/kernel/security
# echo 1 >/sys/kernel/security/evm
```

* Go back to the installation summary screen and start the installation
* During the installation execute the following commands from the console:
```
# cp -r /etc/keys /mnt/etc/ # Debian based distributions: use /target instead of /mnt
# cp /etc/sysconfig/{evm,masterkey} /mnt/etc/sysconfig/
```

This should work on any distribution using dracut.
Loading EVM keys is also possible with initramfs-tools (Debian based distributions).

Of course it's possible to install OS usual way, add keys later and fix missing xattrs with:
```
evmctl -r ima_fix /
```

or with `find` if evmctl is not available:
```
find / \( -fstype rootfs -o -fstype ext4 -o -fstype btrfs -o -fstype xfs \) -exec sh -c "< '{}'" \;
```
Again, fixing requires `ima_policy=tcb|appraise_tcb ima_appraise=fix evm=fix` kernel parameters.
