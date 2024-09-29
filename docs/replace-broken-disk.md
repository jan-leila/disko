# ⚠️ WARNING ⚠️
While disko can run incrementally it is not recommended for users that do not have a good recovery option in the case that something fails due to some unforeseen edge case. If keeping your data intact is critical make sure you are following the 3-2-1 backup strategy before proceeding. Read **ALL** steps of any recovery plan you decide on using before implementing them. The Developers of Disko are not responsible for any data loss you may incur from a failed recovery proceed at your own risk.

# Single drive failure:
## Situation:
We are going to define a system configuration that sets up two two drives in a zfs mirror pool

`flake.nix`
```nix
{
  description = "Example ZFS recovery machine";
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

		disko = {
			url = "github:nix-community/disko";
			inputs.nixpkgs.follows = "nixpkgs";
		};
  };
  outputs = { self, nixpkgs, disko }: {
    nixosConfigurations.host = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        # Setup disks with disko
        disko.nixosModules.disko
        ./host.nix
      ];
    };
  };
}
```

`host.nix`
```nix
{ inputs, lib, config, ... }:
{
  imports = [
    ./disko-config.nix
  ];

  options.host.disko.disks = lib.mkOption {
    type = lib.types.listOf lib.types.path;
    default = [ "/dev/nvme0n1" "/dev/nvme1n1" ];
    description = lib.mdDoc "Disks formatted by disko";
  };

	config = {
		boot.loader.grub = {
			enable = true;
			efiSupport = true;
			efiInstallAsRemovable = true;
			devices = lib.mkForce [];
			mirroredBoots = lib.imap0 (i: device: lib.mkMerge [
				(lib.mkIf (i == 0) {
					path = "/boot";
				})
				(lib.mkIf (i != 0) {
					path = "/boot-fallback-${toString i}";
				})
				{
					devices = [ "nodev" ];
				}
			]) config.host.disko.disks;
		};

    networking.hostId = "c843aedb";

		system.stateVersion = "23.11";
	};
}
```

`disko-config.nix`
```nix
{ lib, config, disko, ... }:
{
	disko.devices = {

		disk = lib.mkMerge (
			lib.imap0 (i: device: (
				lib.genAttrs [device] (_: {
					name = lib.replaceStrings [ "/" ] [ "_" ] device;
					device = device;
					type = "disk";
					content = {
						type = "gpt";
						partitions = {
							boot = {
								size = "1M";
								type = "EF02"; # for grub MBR
							};
							ESP = {
								size = "1G";
								type = "EF00";
								content = lib.mkMerge [
									(lib.mkIf (i == 0) {
										mountpoint = "/boot";
										type = "filesystem";
										format = "vfat";
										mountOptions = [ "nofail" ];
									})
									(lib.mkIf (i != 0) {
										mountpoint = "/boot-fallback-${toString i}";
										type = "filesystem";
										format = "vfat";
										mountOptions = [ "nofail" ];
									})
								];
							};
							zfs = {
								size = "100%";
								content = {
									type = "zfs";
									pool = "zroot";
								};
							};
						};
					};
				})
			)) config.host.disko.disks
		);

		zpool = {
			zroot = {
				type = "zpool";
				mode = "mirror";
				# NOTE: for this demo do we want to just drop the options here to have a more stripped down easier to understand configuration with less noise in it
				options = {
					ashift = "12";
				};
				rootFsOptions = {
					acltype = "posixacl";
					atime = "off";
					compression = "zstd";
					mountpoint = "none";
					xattr = "sa";
					"com.sun:auto-snapshot" = "false";
				};
				datasets = {
					root = {
						type = "zfs_fs";
						mountpoint = "/";
						options.mountpoint = "legacy";
					};
				};
			};
		};
	};
}
```

In this simulated system one of our administrators causes a catastrophic failure and wiped our primary boot drive by running `wipefs -a /dev/nvme0n1` while on a live usb system. This isn't too bad because we still have our backup boot partition `/boot-fallback-1` that we can boot into and all of our data is still here on our second mirrored drive, but now we need to recover our missing drive and get our pool back to a health state.

## Recovery:
The first thing that needs to be done is to regenerate the disk partitions on our drive.

If this was a situation where the existing drive was damaged and needed to be replaced you would first edit your system configuration to reflect the drives configuration of the drives that you are planning running with.

After your drive configuration is how you want it you can run the disko cli tool to format your drives to match your defined configuration:
`nix run github:nix-community/disko -- --mode format --flake .#host`

(NOTE: have we tried rebooting at this point instead of doing this detach attach maneuver???)
After this is done disko will attempt to reattach your drive to the zpool but this will fail due to the drives not being mounted correctly.
To amend this get the name of the new partition and detach it from the zpool and then attach the proper partition name:

```
[root@localhost:~]# zpool status
  pool: zroot
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
        invalid.  Sufficient replicas exist for the pool to continue
        functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-4J
config:

        NAME                       STATE     READ WRITE CKSUM
        zroot                      DEGRADED     0     0     0
          mirror-0                 DEGRADED     0     0     0
            disk-_dev_nvme0n1-zfs  ONLINE       0     0     0
            DISK_NAME_HERE   			 UNAVAIL      0     0     0  was /dev/nvme1n1
[root@localhost:~]# zpool detach zroot DISK_NAME_HERE
[root@localhost:~]# zpool attach zroot disk-_dev_nvme0n1-zfs nvme1n1p3
```

After the pool disk has been reattached to the pool reboot your device and everything should mount correctly.

Then the last step is to rebuild your system configuration to update GRUB on all of your existing drives.
