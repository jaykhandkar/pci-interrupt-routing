# PCI(e) Interrupt Routing
The PCI local bus specification defines four active low, level trigerred interrupt signals - ```INT[A-D]#``` per device. 
On x86 machines, you have the IOAPIC and legacy 8259 interrupt controllers. The 8259 has effectively 15 interrupt pins, and the
IOAPIC usually has somewhere between 19-24 pins, with the possibility of having more than one in a chipset. To install an
interrupt handler for a PCI device, you need to find out which pin the device is using, and to which pin on the 8259 or IOAPIC
it is connected. How would you do this?

On very old, pre-ACPI systems you had the traditional PCI BIOS - access is provided to the routines through real mode interrupt
vectors - ```int 1AH```. I'll skip over this since it is irrelevant today, but the essence is that there is are routines to
get and set interrupt routing information given a PCI bus and device number. This method is defined only for 8259 controllers,
possibly because it predates the existence of the IOAPIC. Note that you can _set_ interrupt information - meaning the interrupt
pin of the 8259 that the interrupt is routed to is configurable - this is true even for the legacy 8259s included in today's
chipsets. You can find the details of this method in the PCI firmware specification.

For systems with ACPI, the DSDT includes a device for every host bridge/root complex present on the system. Under this device,
you have a method called ```_PRT```, or PCI Routing Table, that includes interrupt information for devices on the root bus
corresponding to this particular host bridge. The information returned by this method varies depending on whether you want
routing information for the legacy 8259 PIC or for the IOAPIC(s). To declare which one you are looking for, execute the
```_PIC``` method. Usually this method saves your preference in a global variable and then this variable is used when calling
```_PRT```. Let us look at a part of the information returned by ```_PRT``` when you have already called ```_PIC``` requesting
information for the IOAPIC:

```
Package (0x04)
{
    0x001CFFFF, 
    0x00, 
    0x00, 
    0x10
}, 

Package (0x04)
{
    0x001CFFFF, 
    0x01, 
    0x00, 
    0x11
}, 

Package (0x04)
{
    0x001CFFFF, 
    0x02, 
    0x00, 
    0x12
}, 

Package (0x04)
{
    0x001CFFFF, 
    0x03, 
    0x00, 
    0x13
}, 
```
This is ACPI AML code, but you don't need to understand much to see what's going on here. The ACPI specification defines a 
format for the information returned by ```_PRT``` - each package must have:
- A DWORD for the address of the device - for PCI devices the upper word contains the device number and the lower word contains
the function number. A function number of ```0xFFFF``` (an invalid function number as PCI defines only 8 functions per device)
indicates 'any device'.
- A Byte indicating which pin - ```0x0``` is ```INTA#``` and ```0x3``` is ```INTD#```.
- Either a NamePath or a Byte. If it is a NamePath, it indicates a device in the ACPI namespace which whill allocate the
interrupt - a programmable interrupt router. If it is a Byte, it has to be zero and indicates that the interrupt is fixed
and cannot be configured.
 - A DWORD that is defined only if the Byte in the previous field is zero. This indicates the GSI (global system interrupt)
that the interrupt is routed to. The ACPI specification defines GSIs as follows - to get the pin of an IOAPIC that the
interrupt is connected to, subtract the GSI base for the IOAPIC (can be found in the MADT) from the GSI. Meaning, if you have
two IOAPICs with 24 pins, they might be indicated in the MADT as having GSI bases of 0 and 24, and so when you encounter a 
GSI of 40, it really means pin 16 of the second IOAPIC.

In the above piece of code, take the first package - if you apply the above format you get to know that the ```INTA``` pin
of device ```0x1c``` (bus 0 in this case since I took this out of the ```_PRT``` of the only host bridge on this system) is
connected to GSI ```0x10```. On this system, the MADT indicated that there is only one IOAPIC with a GSI base of zero, so this
is really pin 16 of the only IOAPIC of the system. So you can program the corresponding redirection entry in the IOAPIC's
configuration registers with the vector and LAPIC ID of your choice and install your interrupt handler for that vector, and 
your job is done. Your interrupt handler will be called everytime this device drives its ```INTX#``` pin.

What if, for whatever reason, you want to use the legacy 8259 PIC? The package returned by ```_PRT``` will look like this - 
```
Package (0x04)
{
    0x001CFFFF, 
    0x00, 
    \_SB.LNKA, 
    0x00
}, 

Package (0x04)
{
    0x001CFFFF, 
    0x01, 
    \_SB.LNKB, 
    0x00
}, 

Package (0x04)
{
    0x001CFFFF, 
    0x02, 
    \_SB.LNKC, 
    0x00
}, 

Package (0x04)
{
    0x001CFFFF, 
    0x03, 
    \_SB.LNKD, 
    0x00
}, 
```
Applying the format defined by the ACPI spec to the first package, we get to know that pin ```INTA#``` of device ```0x1c```
is configurable and is allocated by a device called ```\_SB.LNKA``` in the ACPI namespace. I'll include the definition of
this device below:
```
    Device (LNKA)
        {
            Name (_HID, EisaId ("PNP0C0F") /* PCI Interrupt Link Device */)  // _HID: Hardware ID
            Name (_UID, 0x01)  // _UID: Unique ID
            Method (_STA, 0, NotSerialized)  // _STA: Status
            {
                If (!VPIR (\_SB.PCI0.LPC.PIRA))
                {
                    Return (0x09)
                }
                Else
                {
                    Return (0x0B)
                }
            }

            Name (_PRS, ResourceTemplate ()  // _PRS: Possible Resource Settings
            {
                IRQ (Level, ActiveLow, Shared, )
                    {3,4,5,6,7,9,10,11}
            })
            Method (_DIS, 0, NotSerialized)  // _DIS: Disable Device
            {
                \_SB.PCI0.LPC.PIRA |= 0x80
            }

            Name (BUFA, ResourceTemplate ()
            {
                IRQ (Level, ActiveLow, Shared, _Y00)
                    {}
            })
            CreateWordField (BUFA, \_SB.LNKA._Y00._INT, IRA1)  // _INT: Interrupts
            Method (_CRS, 0, NotSerialized)  // _CRS: Current Resource Settings
            {
                Local0 = (\_SB.PCI0.LPC.PIRA & 0x8F)
                If (VPIR (Local0))
                {
                    IRA1 = (0x01 << Local0)
                }
                Else
                {
                    IRA1 = 0x00
                }

                Return (BUFA) /* \_SB_.LNKA.BUFA */
            }

            Method (_SRS, 1, NotSerialized)  // _SRS: Set Resource Settings
            {
                CreateWordField (Arg0, 0x01, IRA2)
                FindSetRightBit (IRA2, Local0)
                Local1 = (\_SB.PCI0.LPC.PIRA & 0x70)
                Local1 |= Local0--
                \_SB.PCI0.LPC.PIRA = Local1
            }
        }
```
It looks quite a bit more complex, but here's how it works - you call the ```_PRS``` (Possible Resource Settings) method of
this device to get the list of all possible IRQs of the 8259 that you can route it to. In this case, you can see that it's
IRQs 3,4,5,6,7,9,10,11. If you want to route it to one of these IRQs, call ```_SRS``` (Set Resource Settings) with that IRQ
as an argument. To see which IRQ is currently configured, call ```_CRS``` (Current Resource Settings). To disable the link,
call ```_DIS``` and to get the status of the link, call ```_STA```. This is what these methods do. To understand how they do
it, and what all that code above means, keep reading.

You will notice that all this information is bus relative, since every package contains only a device number. Every PCI bridge
on the system will also have a a ```_PRT``` method to provide routing information for devices behind that bridge. In case it 
doesn't, or the bridge doesn't exist at all in the ACPI namespace, it means that the interrupt pins for devices behind that
bridge are mapped to interrupt pins on the bridge itself in a specific manner. This is the interrupt _swizzle_ defined in the
PCI-PCI bridge specification - 

![swizzle](https://user-images.githubusercontent.com/23404671/178135531-f4a27edd-9651-4f19-a70d-4d3f5e296e8a.png)

So, given that you have interrupt routing information for the bridge itself, you can get the routing for devices behind the bridge.
By now, you should know how to get interrupt routing information for a particular PCI device and exactly how its ```INTX#``` pins are
routed to the systems interrupt controller. So if your goal is to get PCI interrupts working on your operating system, you should be
good to go.

I will now take the example of a specific chipset and show _why_ the routing information provided is what it is. I will use a Thinkpad
T420 - it contains a Sandy Bridge Intel Core i5-240M with a Cougar Point Intel C200 PCH chipset. The datasheets for both of these are
publicly available, and so I can show you exactly how you can make sense of the interrupt routing information you will find on this 
machine. First, there are some differences you should understand since this system contains a PCIe fabric and not PCI, although PCIe
is compatible with PCI from a software perspective.

PCIe is a point to point interconnect as opposed to the shared bus nature of PCI. To remain compatible with PCI in software, consistency
is maintained with the PCI topology as follows - the point to point link is represented as a bus, and the PCIe endpoint at the end of it 
is device 0. This is why you will notice that on most machines, ```lspci``` will display a lot of devices with number 0 and on different 
buses. A root port presents itself as a PCI-PCI bridge to software, with the bus number of its secondary side (the PCIe link) freely 
configurable just as in PCI. You will also see devices with non zero device numbers - these are root complex integrated devices - they are
present in the chipset or processor package itself and are enumerated as PCI devices, with a hardcoded device number. PCIe also eliminates
the four sideband interrupt signals in favour of message signaled interrupts. For software compatibility, a special TLP (Transaction Layer
Packet, PCIe is a layered protocol) type is defined - ```Assert_INTX#``` and ```Deassert_INTX```. The device generates these transactions
if it has been configured to use legacy interrupts. The root complex must keep track of these messages and translate them into an interrupt
event at a pin of one of its interrupt controllers.

Now, about the Thinkpad T420 I shall be talking about - the Core i5-240M has a memory controller integrated into the CPU package. This
means that the silicon that generates memory transactions on DRAM, generates PCIe configuration transactions, maps address ranges to 
specific targets is on the processor package itself - functions that were traditionally part of the Northbridge external chipset. This
Thinkpad contains the C200 series PCH, or platform controller hub - this is the modern equivalent of the Southbridge and contains various
I/O such as xHCI, SATA, PCIe ports etc. The PCH is linked to the processor through DMI (Direct Media Interface). You can think of the DMI
as just a logical extension of PCI bus 0. Look at the output of ```lspci``` on this system - 

![t420-lspci](https://user-images.githubusercontent.com/23404671/178156363-38dc8197-d610-45b4-911d-5c9823a881c9.png)

Devices 0 and 2 on PCI bus 0 are part of the processor package, and devices 0x16 through 0x1f are part of the PCH.
