---
title: Gen8 HP Microserver vs TrueNAS SCALE
date: 2022-11-20 11:00:00
categories: [Tech]
tags: [microserver,truenas,grub,gen8]
img_path: /assets/20221120-userver/
---

![Banner image](banner.png)

TL;DR: TrueNAS SCALE on a Gen8 HP Microserver is a bit of a faff. This is what worked for me.

## Well, d'uh?

I picked up a cheap Gen 8 Microserver recently, viewing it as a way to move up a step from cheap underpowered Lenovo and Synology RAID boxes, as well as to get onto something less vendor-locked, hence TrueNAS. 

I also wanted to move to a platform capable of lightweight virtualisaton and container hosting, further suggesting TrueNAS as a potential option. 

By default the Microserver comes with four HDD bays, and an optical drive sandwiched in the top. There's also an internal microSD slot along with an internal USB header, which together are _suggestive_ of flexible boot options.

However, I'd heard vague, non-specific, rumours suggesting that the BIOS on the Gen8 Microserver was a touch... _funky_. How bad could it possibly be?

## Round 1

I started fiddling with this months ago, but hit a brick wall, so put it aside for _some time_. Some of these recollections are fuzzy. I don't write everthing down as I do it like Jeff Geerling.

The initial sequence of ~~disappointment~~ events was as follows:
1. Acquire server
1. Rip out the optical drive, replace with a smallish SATA SSD, complete with molex-to-SATA power adapter
1. Upgrade the weak-sauce CPU to something Xeon-y
1. Buy the wrong sort of RAM upgrade, stick with the 4GB it came with
1. Grab TrueNAS SCALE[^1] & burn to a USB stick
1. Fiddle with the BIOS to ensure USB booting was a thing
1. Boot!
1. Install!
1. Reboot!
1. \<_fails to boot_\> Oh.

[^1]: I tried TrueNAS CORE first, but it didn't like the funky 10GBE + M.2 PCI card[^2] I'd thrown into the box for grins. 

[^2]: A QNAP something-or-other.

That funky BIOS? Yeah. In summary:

- Boot off USB thumb drive attached to internal or externa ports? Absolutely.
- Boot off internal microSD card? Absolutely.
- Boot off something large on the end of a USB-to-SATA converter? Nope. 
- Boot off something large on the end of a USB-to-M.2/mSATA converter? Nope. 
- Boot off something large on the end of the SATA optical drive port? Nuh-uh. 

The _innernet_ claims that it will happily boot off the first drive in the HDD bays though, so all good, right? 

Wrong: if you're using something like TrueNAS, it'll (reasonably) want that drive all to itself, so you lose that slot to hosting the OS.  Further consider that the first drive bay is one of only two 6Gbps ports, and... also no.

The innernet _also_ claims that the route forward is to
1. Install TrueNAS SCALE to the place you really want it running from - in my case, the SATA SSD on the optical drive SATA port
1. Install a GRUB bootloader on something the Gen8 box _will_ boot off (internal or external USB, or the microSD card) and have GRUB chain to the real bootloader. 

That seems entirely sensible but, at the time, piecing together multiple pages of infonuggets on the topic, AND understanding how to write GRUB config, AND understanding what the heck HP were doing... well, I put it aside for a while.  Most of a year, in fact - I had better things to do[^3].

[^3]: Like persuading a swanky car dealer to take back a used car after a year and 10,000 miles without leaving me significantly out of pocket. I won, but it took a bit of time and persistence

## Round 2

Initial frustration a memory, I had anoter go around at getting the thing to boot from the device of my choosing.  This time around, I stubled over some (german language) advice for writing GRUB configs that _might_ work. 

With this in mind, my starting point was:
1. A 250GB SSD in on the optical drive SATA connection, "air mounted" in the top of the Microserver
1. A small microSD card in the internal slot
1. The latest TrueNAS SCALE image burnt to a 32GB USB thumb drive
1. An Ubuntu 22.something image burnt to _another_ USB thumb drive to be used as a Live distro 

### The install process

> First of all, everything here is derived from the German instructions at <https://www.bastelbunker.de/hp-microserver-gen8-booten-von-den-odd-port/>, which were about getting Windows booting of a drive connected to the optical drive port, and got me 90% of the way with TrueNAS.

Prep the microSD card

> I erased it on my Mac, and specified MBR as the partition table type (others may work, but will cause it to vary from the instructions I followed to get here, and I'm not sure if something different will upset the funky HP BIOS)
{: .prompt-info }

Then stuff it in the Microserver - everything else will happen there. 

Next, install TrueNAS scale from its USB installer thumb drive. Remove the installer thum drive and let it reboot - whereupon it'll fail to find something to boot from). Not documented here because (a) it's well documented elsewhere (b) it's _done_ already and (c) I didn't take notes[^4].

[^4]: By the time you've been through about 20 POST-install-reboot-POST-bootfail-POST-fiddle-reboot-POST-bootfail loops... you'd quit taking notes too

Now swap installer thumb drives - and boot from the Ubuntu installer and select the option to Try Ubuntu.

Once in Ubuntu, grab a shell, figure out which device corresponds to your microSD card. Knowing how big it is will help here. I brute force likely devices using:

```console
dmesg | fgrep sd
```

We'll call the microSD device `/dev/sdX`.

Partition the card:

```console
fdisk /dev/sdX
->'n' Create a new partition
Set --->'p' to Primary (default)
--->'1' Select the first partition (default)
---> In the first sector, we leave everything as it is
--->'+128M' creates a 128MB partition
->'a' To set the start flag
--->'1' Choose the first partition
->'w' Save and apply everything
```

Create a filesystem:

```console
mkfs -t ext2 /dev/sdX1
```

Mount the SD card somewhere:

```console
mkdir /tmp/usb
mount /dev/sdX1 /tmp/usb
```

Make the folder where GRUB will live:

```console
mkdir /tmp/usb/boot
```

Install GRUB:

```console
grub-install --boot-directory=/tmp/usb/boot /dev/sdX
```

This will populate `/tmp/usb/boot/grub` and put a bootblock on the microSD card.

Create the GRUB configuration:

```
vi /tmp/usb/boot/grub/grub.cfg
```

Fill it with:

```
set default='3'
set timeout='10'

menuentry 'TrueNAS SCALE hd5' {
	set root=(hd5)
	chainloader +1
}

menuentry 'TrueNAS SCALE hd4' {
	set root=(hd4)
	chainloader +1
}

menuentry 'TrueNAS SCALE hd3' {
	set root=(hd3)
	chainloader +1
}

menuentry 'TrueNAS SCALE hd2' {
	set root=(hd2)
	chainloader +1
}

menuentry 'TrueNAS SCALE hd1' {
	set root=(hd1)
	chainloader +1
}
```


> **Beware:** The `hdX` references WILL NOT line up with `/dev/sdX` references from `dmesg` once Linux has booted.
{: .prompt-warning }
> This is all from the BIOS's perspective.  What this gives me, in my circumstances, is a way to boot from the SSD on the optical drive SATA connector, whether I've got 0, 1, 2, 3 or 4 drives in the not-actually-hot-swap SATA bays.  You can use the `default` variable to pick which one will be chosen once the 10 second `timeout` expires, and I _assume_ it'll fall back to a prompt if, say, you've added or removed a drive causing the TrueNAS drive to change number.  
{: .prompt-info }

Save that file, and unmount the microSD card:

```console
cd / 
umount /tmp/usb
```

Finally, reboot (remembering to yank out the Ubuntu installed USB thumb drive so you don't go back down the rabbit hole).

Now you should find that the Microserver boots off the microSD card and lets you pick one of five options that will _then_ boot Linux off the TrueNAS SCALE drive on the optical drive port. 

Good luck.

## The end!

Hope that helps anyone on the same journey I was on, and also perhaps lacking time (willpower, experiences, whatever) to figure out the fine-grain details of GRUB vs the Gen8 Microserver BIOS.

Thanks to [IdleBit](https://www.bastelbunker.de/author/shojo/) for writing the [instructions](https://www.bastelbunker.de/hp-microserver-gen8-booten-von-den-odd-port/) that got me nearly all the way here.

---

