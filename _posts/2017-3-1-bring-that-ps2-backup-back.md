---
layout: post
title: Bring that PS2 backup back
---

{% include google_analytics.html %}

## PS2 memcard data on your PC, can you work with that?
I have my old PS2 memcards and I want to play on the [PCSX2](http://pcsx2.net) emu with the old savegames, how do I do that?

### The right devices and system
You are a hardcore [PlayStation 2](https://en.wikipedia.org/wiki/PlayStation_2) player (ok even [PS1](https://en.wikipedia.org/wiki/PlayStation_1) maybe) and you still have those old and rare Sony memory cards around (ok yes maybe cheap chinese replicas too). And they are loaded with savegames. You want those to come back to life, bring the clock back to the late 2000's and start that GTA Vice City Stories with close-to-100% percentage completion. Yes, you can. And it is not that crazy difficult, either.

First of all, how do we hook up the memory card to the computer? You need the adapter. Right, not just "some adapter". *Buy on eBay an original Sony memcard adapter*. I stress this because, for personal experience corroborated by [old forum threads](http://psx-scene.com/forums/f153/ps3-memory-card-adaptor-cechzm1-allows-free-mc-boot-install-88649-print/index23.html) cheap chinese replicas just won't work. Apparently the chipset around is missing some crypto detail and the data transfer fails. C hackers from the forum tried hard, unless you are the new Stallman I suggest you to but the real one. By the way it looks like [this](https://upload.wikimedia.org/wikipedia/commons/6/65/PS3-memory-card-adapter.jpg), don't get fooled on Tha Intahnet.

Also, this has been tried on Linux even if the util we are going to use is for Windows too. Nevertheless you might have to get yourself the [libusb](www.libusb.org/wiki/libusb-win32) driver.

### t00Lz

Download the [ps3mca-tool]({{ site.url }}attachments/ps3mca-tool-fmcb-1.93..zip). This is the memory card adaptor tool, created by ancient hackers. We will copy over the contents of the memcard as a binary image:
```./ps3mca-tool -img memcard.img```
This is just a binary data image, simple and plain.

Now use [ecc_check]({{ site.url }}attachments/ecc_check.zip) to add the ECC info to your dump:
```ecc_check.exe memcard.img memcard.ps2```
Basically the ps2 format is the binary dump with some error-correcting codes. Aha, we are clearly using Wine here.

### h3mu
Install PCSX2 and configure the BIOS. You can simply install it from the software center (and you should). Unless your computer is blazing fast and new, as in OpenGL 3+, don't even bother. Go get a powerful box and come back to this later, we'll wait for you.
Ok. Configure the BIOS now. I am not hosting the BIOS file "because w4r3z" but you can find many on eMule, or well maybe you have flashed yours already. In the CDVD menu select "No disc", unless you have one, ie you didn't listen when I told you to go get a real computer.

Drop the ps2 files into the `memcards` folder and mount them in from the PCSX2 configuration.
Use this [PCSX2-modded uLaunchELF]({{ site.url }}attachments/uLE_v4.40h_PCSX2.zip)ded and boot your emu - there is no real difference between the regular and compressed binary. Enter the first two folders, these are the memcards, make sure your data is there.

Load the iso and go! The memory card will be recognized and usually the latest savegame start.

### Troubleshooting

If it's like no memory card is detected in the game, it could be the region, savegame's are also region bounded, like the optica. But fixing this is pretty easy. If everything is working fine already, skip this section.

Download [myMC]({{ site.url }}attachments/mymc_2.6.g2.dist.7z) and a hex editor. Open `mymc-gui.exe` and export the savegame to convert. Download a savegame in the target region from [GameFAQs](https://www.gamefaqs.com/ps2/938211-grand-theft-auto-vice-city-stories) or so. So say your current savegame is PAL but the .iso of the game is NTSC, you want to convert the PAL save to the NTSC then, so download a US save from the website.

Backup your PCSX2 `memcards` directory, that is just duplicate it. Now in myMC select this `memcards` folder and the desired memcard, then select the savegame to modify and export it. Open the exported `.psu` file with the hex editor, and open the one by GameFAQs too. Replace any `BESLES-XXXXX` occurrence in the first, so our savegame to convert, with the `BESLUS-YYYYY` you find in the second - or vice versa; it could be many instances or just one.

Save the modified file and rename it, so `SLES-XXXXX` to `SLUS-YYYYY`. Finally, import this new `.psu` into the memcard with myMC. You can keep the old savegame if you have enough space of course.

### Play on!

Now you are all set up to bring those old achievements back to life, congrats! Unless your computer has rockstar specs, you may experience lag - no matter that PAL vs NTSC framerate thing. PCSX2 does help in this sense, set the speed hacks to say level four in the configuration. In GTA:VCS this sensibly improved the playability, from map and radio lag to fluid, or better regular, game dynamics.

About where to find games, see question for the BIOS above, but you can use your old PS2 DVDs if you still have them, or turn to [eMule](www.emule-project.net/) for some fresh ISOs.

That's all! Play on!
