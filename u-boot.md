#Intel Edison

##Using Intel's Stuff
Best walk-through is [here](http://shawnhymel.com/585/creating-a-custom-linux-kernel-for-the-edison/)

## Device Data

### Partition Table
```
boot > part list mmc 0

Partition Map for MMC device 0  --   Partition Type: EFI

Part    Start LBA       End LBA         Name
        Attributes
        Type GUID
        Partition GUID
  1     0x00000800      0x000017ff      "u-boot0"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   d117f98e-6f2c-d04b-a5b2-331a19f91cb2
  2     0x00001800      0x00001fff      "u-boot-env0"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   25718777-d0ad-7443-9e60-02cb591c9737
  3     0x00002000      0x00002fff      "u-boot1"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   8a4bb8b4-e304-ae48-8536-aff5c9c495b1
  4     0x00003000      0x000037ff      "u-boot-env1"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   08992135-13c6-084b-9322-3391ff571e19
  5     0x00003800      0x00003fff      "factory"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   333a128e-d3e3-b94d-92f4-d3ebd9b3224f
  6     0x00004000      0x0000ffff      "panic"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   f20aa902-1c5d-294a-9177-97a513e3cae4
  7     0x00010000      0x0001ffff      "boot"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   db88503d-34a5-3e41-836d-c757cb682814
  8     0x00020000      0x0031ffff      "rootfs"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   012b3303-34ac-284d-99b4-34e03a2335f4
  9     0x00320000      0x0049ffff      "update"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   faec2ecf-8544-e241-b19d-757e796da607
 10     0x004a0000      0x00747fde      "home"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        guid:   f13a0978-b1b5-1a4e-8821-39438e24b627
```
