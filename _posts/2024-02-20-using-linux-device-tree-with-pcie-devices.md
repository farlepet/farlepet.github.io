---
layout: post
title:  "Using Linux device-tree with PCIe devices"
date:   2024-02-20 17:45:00 -0600
categories: linux
---
Recently for work, we have been going through the process of removing any use
of the deprecated `/sys/class/gpio` paths for interacting with GPIOs. This has
been accomplished by defining these GPIOs in the device-tree in association with
some driver - using a pre-existing one like gpio-leds or gpio-keys, extending an
existing driver, or creating an entirely new driver.

Most of these have been pretty straightforward, they have been devices that already
are typically found on the device tree, mainly platform devices (devices that
are more-or-less independent of any bus). But recently I needed to do the same
for some devices that don't typically have device-tree entries: PCIe devices.
These are auto-enumerated by the kernel, so most of the time they don't need any
help for detection. But in this case, I needed to find a way to represent these
devices in the device-tree so I could add configuration values to them.

I tried looking online for any documentation on doing this, but it seems that
it's a relatively uncommon need. Most posts/questions/articles I could find
on the subject seemed to be about setting up the PCIe ports (root complex ports?)
that are directly on an SoC. In my case, this much was already done, and these are
commonly provided by the vendor (though in some instances they may need to be
modified) if the SoC has good support. I was able to find
[one Stack Overflow question](https://stackoverflow.com/questions/54367498/creating-a-device-tree-for-the-hardware-on-a-pci-device)
that gave me some clues on how to go about it. I was also able to find a little
more in the kernel documentation, but again most of that was about setting up
root ports/bridges.

Initially, I more-or-less got it working on a couple products through
trial-and-error. I always just kept the `regs` values as all zeros, and didn't include
the `ranges` values. On these products, it was a lot simpler since the PCIe
devices were connected directly to the SoC. In the end, I was able to get away
with something similar to the following:

```
&pcie0 { /* Typically defined in a dtsi for the SoC */
    status = "okay";

    pcie@0,0 { /* PCIe bridge/root */
        #address-cells = <3>;
        #size-cells    = <2>;
        reg = <0 0 0 0 0>;

        device_name: pcie@0 {
            compatible = "pci<vendor ID>,<device ID>";

            reg = <0 0 0 0 0>;

            /* Your stuff goes here */
        }
    }
}
```

And this worked great for those simpler products, but we also had a product
that uses a PCIe switch on one of the busses. I attempted to get it working
through dumb trial-and-error again, but this time I wasn't having any luck.
I went through `/sys/bus/pci/devices` to find the hierarchy, which got me
a good bit of the way there, but the driver still was not picking up the entries.

Eventually I remembered seeing [this LKML patch](https://lwn.net/Articles/917999/)
that I had initially dismissed as I didn't fully read it and didn't know it
was in mainline and usable on our platform. So I enabled the kernel config
option (`PCI_DYNAMIC_OF_NODES` - Create Device tree nodes for PCI devices)
and loaded the new kernel on my device. At first, this didn't
fully work - it seems that it doesn't always fully auto-generate the entries
if there are any conflicting entries in the device-tree. So I temporarily
removed those entries, and then it fully generated the PCIe device-tree
(apart from the devices). In order to read it in a sensible way, I copied
the contents of `/sys/firmware/devicetree` to my computer, and used `dtc`
to convert it into `dts` format (`dtc -I fs -O dts <path> > tree.dts`).

At this point, I just tried replicating the same structure it had. I think
the most important part I was missing was proper values for the `reg` field.
The first bridge has them as all zeros, so it makes sense that that also
worked for me. But the switch/nested bridges had non-zero values for these.
After adding those, it still wasn't working correctly. After adding some
debug prints to show which items in the PCIe hierarchy properly picked up
fwnode items, I realized that it was trying to use another node I hadn't moved
yet for one of the bridges, and not the node that actually had the device as
a child. (My current guess is that for bridges where they are alone at the
same location in the hierarchy, a `reg` of all zeros will work, but it might
not if the bridge has neighbors at the same level, or if it's device ID is not
zero. I'm just glad to have gotten it working, and don't really feel like
trying a bunch of permutations to figure out all the rules.)

So in the end, I ended up with a device-tree closer to this:

```
&pcie0 {
    status = "okay";

    pcie@0,0 { /* Root */
        #address-cells = <3>;
        #size-cells = <2>;
        reg = <0 0 0 0 0>;

        pcie@0,0 { /* Switch */
            #address-cells = <3>;
            #size-cells = <2>;
            reg = <0x10000 0x00 0x00 0x00 0x00>;

            pcie@1,0 { /* Switch */
                #address-cells = <3>;
                #size-cells = <2>;
                reg = <0x20800 0x00 0x00 0x00 0x00>;

                device_0: pcie@0 { /* Device */
                    compatible = "pci<pid>,<vid>";
                    reg = <0 0 0 0 0>;

                    /* Properties go here */
                };
            };

            pcie@3,0 { /* Switch */
                #address-cells = <3>;
                #size-cells = <2>;
                reg = <0x21800 0x00 0x00 0x00 0x00>;

                device_1: pcie@0 { /* Device */
                    compatible = "pci<pid>,<vid>";
                    reg = <0 0 0 0 0>;

                    /* Properties go here */
                };
            };
        };
    };
};

```

And in case it might be useful to anyone else, this is the code I used to
see what PCIe bus nodes were picking up fwnodes:

```c
strict pci_dev *dev;

...

struct pci_bus *p_bus = dev->bus;
while (p_bus && (p_bus->parent != p_bus)) {
    if (p_bus->dev.fwnode) {
        const char *name = fwnode_get_name(p_bus->dev.fwnode);
        dev_warn(&dev->dev, "PCI parent bus has fwnode: %s", name);
    } else {
        dev_warn(&dev->dev, "PCI parent bus has no fwnode");
    }

    p_bus = p_bus->parent;
}

if (dev->dev.fwnode) {
    const char *name = fwnode_get_name(dev->dev.fwnode);
    dev_warn(&dev->dev, "Device has fwnode: %s", name);
} else {
    dev_warn(&dev->dev, "Device has no fwnode");
}
```
