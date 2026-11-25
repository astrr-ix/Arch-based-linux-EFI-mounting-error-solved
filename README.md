# Arch-based-linux-EFI-mounting-error-solved
Updated Arch and then used timeshift to restore a capture from before the update. Error while rebooting, stuck in emergency mode. 

Disclaimer: I am a beginner, this happened to me one week into linux so I might explain some things vaguely or wrongly. Take it with a grain of salt.
# Probably gonna require live USB of the distro (unless it works before that step)
(Done on EndeavourOS but probably works similarly on other Arch based distros)

Issue? 
    Updating Arch updates kernel, timeshift restores former kernel but not the old kernel image in the EFI partition (necessary to boot)

Solution:
    Restoring the former kernel image using Chroot (live boot)

# Steps (troubleshoot and solve):
Boot screen:
  [FAILED] Failed to mount /efi.
  
  [DEPEND] Dependency failed for Local File Systems
  
"You are in emergency mode."

    systemctl status efi.mount
  
    mount -v /dev/nvme0n1p4 /efi
  
    mount: /efi: unknown filesystem type 'vfat'
    
    journalctl -b -0 -u efi.mount
  
    blkid
  
  here you can see the partitions with their labels, types, UUIDs, etc. 
    find the partition labeled EFI (if dual booted on one disk like me, find the one associated with the distro partition, it was right underneath for me) and write down UUID
      
       nano /etc/fstab
    
   this opens the file with each partition. look for the EFI, it should have the same UUID as the one written down from before (if the last command showed two EFI partitions, check it again to see which one's in fstab. Thats the EFI for this distro boot.
      
   if it's wrong write down (in case it is correct) the current UUID and try changing it in fstab to the right one.
      
      ctrl+O and then enter to save edit
      
      citrl+X to exit nano
      
    reboot
    
  If error persists, go back to nano /etc/fstab and change the UUID back to the original one

      nano /etc/fstab
      
TEMPORARY fix to boot:

  Look for the line with the EFI partition
  
  Put a # right before the UUID to deactivate that partition without deleting it (so you can undo it easily later)
    
    reboot
  
  it should reboot normally but wireless internet won't work

Fix:    

    ip a
  
  the only output will probably be "lo: ..."/loopback network interface, no wlan listed (no internet interface)
    
    sudo nano /etc/fstab
  
  remove the # but don't reboot yet
    
    uname -r && ls /usr/lib/modules
  
  outputs kernel version and kernel image version (i think, i might have the terminology wrong mb) -> check if they match (they're probably different) 
    
  note the second version listed
    
    ls /var/cache/pacman/pkg/linux-*
  
  look for that same kernel package version in the cache
    
    sudo pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
  
  replace <version> with the version number (ie: linux-7.3.4.arch1-1-86_64.pkg.tar.zst)
    
  Personally I got an error here: 
    
      error: failed to init transaction (unable to lock databases)
      
      error: could not lock database: File exists
      
         if you're sure a package manager is not already running, you can remove /var/lib/pacman.db.lck
         
  close any tab running yay or another package manager
    
      sudo rm /var/lib/pacman.db.lck
      
  (deletes the lockfile that was frozen in the Timeshift capture bcs a package manager was running during the capture)
        
  repeat the kernel installation from cache
        
          sudo pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
          
  press Y to proceed with the installation even if it says you already have it installed
            
  it doesnt know that the more recent version is still running in /efi, reinstalling the old version makes sure everything runs the same old version
              
      sudo reboot
  
  if it works then thats great
  
  it didnt work for me, same efi issue as before, back to emergency mode
  
          uname -r
          
  check if the version is the old or new one this time. It still showed the new version for me, it booted that version from the unmounted partition. You can try the next step before trying the live USB method, it didn't work for me tho
            
          mount /efi
          
  if theres no output it worked, keep going. Otherwise keep reading, skip the next few steps
      
          pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
          
  same command as before, we re trying to reinstall the old kernel
            
          reboot
          
  If mount /efi fails  because of vfat:
# Live boot with USB
1. Live boot (in case you forgot: poweroff, plug in the USB and open the boot menu, select the USB option)
2. Check partition names, specifically the linux root partition (btrfs or ext4 type probably) and linux efi partition (vfat type)

       lsblk -f
   
IF ROOT PARTITION TYPE IS ext4:

  4.     sudo mount /dev/nvme0n1pX /mnt
     replace X with the number from the root partition

         sudo mount /dev/nvme0n1pY /mnt/efi
     replace Y with the number from the EFI partition (ie: nvme0n1p4)
  6.     sudo arch-chroot /mnt
  7.     pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
  8.     exit
         sudo reboot
     I can't help after this step if it still doesn't work, I use btrfs

IF ROOT PARTITION TYPE IS btrfs:
  
  4.     sudo mount -o subvol=@ /dev/nvme0n1pX /mnt
       replace X with the number from the root partition
  5.     sudo mount /dev/nvme0n1pY /mnt/efi
         sudo arch-chroot /mnt
       this connects the EFI partition to /mnt/efi (bcs the live USB/s vfat actually works) and then gets you in Chroot mode which lets you affect the distro from the healthy live boot
  6.     pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
  7.     exit
         sudo reboot
Now pray and hope it works LMAO ok no im kidding, I got an error while mounting: /mnt/efi: mount point does not exist.

8.       ls /mnt
   no output? if btrfs type:

        sudo mount -o subvol=@ /dev/nvme0n1pX /mnt
         ls /mnt
     outputs folders like bin, boot, efi, etc, home, usr and var
10.     sudo mount /dev/nvme0n1pY /mnt.efi
         sudo arch-chroot /mnt  
11.     pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
        error: '/var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst': could not find or read package
        exit
        sudo mount -o subvol=@cache /dev/nvme0n1pX /mnt/var/cache
        sudo arch-chroot /mnt
        pacman -U /var/cache/pacman/pkg/linux-<version>-x86_64.pkg.tar.zst
    thanks to my crazy bad luck, I got another error:

              Running in a chroot
              ...
              fatal library error, lookup self
            reinstall-kernels
            ls -la /efi/loader/entries/
    make sure there's a .conf file with the right arch version in filename

12.     exit
        sudo reboot
      unplug USB when screen turns black

  war is over

# To confirm everythings ok
    uname -r && ls /usr/lib/modules

  both versions should be identical
  
    findmnt /efi

  should show /dev/nvme0n1pY mounted as vfat without any errors
  
    systemctl --failed

  should output 0 loaded units listed



time to fall into a 72h long coma and eat ice cream bcs wtaf was that

  
  
