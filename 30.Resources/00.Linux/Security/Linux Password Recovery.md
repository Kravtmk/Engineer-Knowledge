# Linux Password Recovery via GRUB

**Tags:** #linux #security #grub #recovery  
**Tested on:** Kali Linux  
**Requires:** Physical access, unencrypted root filesystem

## 1. Open GRUB

1. Reboot the computer.
    
2. Select the Linux boot entry.
    
3. Press `e` to edit.
    
4. Find the line starting with `linux`.
    

## 2. Modify Boot Parameters

Replace `ro` with `rw` and append:

```
init=/bin/bash
```

Example:

```
rw quiet splash init=/bin/bash
```

Press `Ctrl + X` or `F10` to boot.

## 3. Reset Password

In the root shell:

```
passwd user
```

Enter the new password twice.

If the filesystem is read-only:

```
mount -o remount,rw /
passwd user
```

## 4. Reboot

```
sync
exec /sbin/reboot -f
```

If reboot fails, power off after `sync`.

## 5. Verify

Log in with the new password.

```
whoami
id
```

## Useful Commands

```
# List regular users
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $6}' /etc/passwd

# Check disk encryption
lsblk -f

# Check current user
whoami
```

## Security Notes

- Login passwords do not protect unencrypted disks against physical access.
    
- GRUB password protection can restrict boot parameter editing.
    
- LUKS2 protects data at rest when the encrypted volume is locked.
    
- Secure Boot does not replace disk encryption.
    
- The procedure may not work with encrypted or protected boot configurations.
    

**Key takeaway:** Physical access + unencrypted disk = potential root access.

**Use only on systems you own or are authorized to administer.**