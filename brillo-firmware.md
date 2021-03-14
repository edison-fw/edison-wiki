# Brillo

Brillo is Intel and Google's combined effort to run Android on Edison. For this a limited amount of devices can be found in the wild with a different firmware, which provides fastboot.

The Brillo files can be found [here](https://dl.google.com/dl/androidthings/edison/devpreview/2/androidthings_edison_devpreview_2.zip).

### Provisioning Edison for Brillo

First Edison must be provisioned with a new firmware before Brillo can be installed:

```
xfstk-dldr-solo --gpflags 0x80000007 --fwdnx edison_dnx_fwr.bin --fwimage edison_ifwi-dbg-00.bin
```

The flashing will appears to hang, but be patient. After a minute or so the command will complete. Edison will appear to hang and needs a manual reset.

This loads a new firmware into Edison with its own built-in shell.

### Brillo IFWI
When provisioned for Brillo the IFWI version is **00ED.1D0E**.

This is an older version as the one supplied by [meta-intel-edison](https://github.com/edison-fw/meta-intel-edison/commit/70525b810cc3e4e368be7334a2423ffaefb49a7c) but it appears to be a debug version with its own shell.

The shell gives access to various registers, lists PCI devices, list SFI tables and so on.

[meta-intel-edison](https://github.com/edison-fw/meta-intel-edison/commit/4688789e42c97baf988ca44ae895778dbd954a70) provides a newer one **00ED.1D0F** 

### Running meta-intel-edison image on Brillo provisioned Edison
U-boot will start and will allow booting a kernel. Some U-Boot commands will hang, like:
```
=> pci
No such bus
=> pci enum
PCI: Failed autoconfig bar 18
(hangs)
```

### Going back to standard Edison
To run meta-intel-edison image or other vanilla kernel configured for Edison Brillo supplied IFWI must be replaced by the latest version 00ED.1D0F. This can be found under `meta-intel-edison/utils/flash/ifwi/edison/`.

Run the same command as above `xfstk-dldr-solo --gpflags 0x80000007 --fwdnx edison_dnx_fwr.bin --fwimage edison_ifwi-dbg-00.bin --osdnx edison_dnx_osr.bin --osimage u-boot-edison.img`
