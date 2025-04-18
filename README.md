## **4. Vivado Project Customisation**
1. Once inside Vivado, navigate to the "sources" box and navigate as such `pcileech_squirrel_top` > `i_pcileech_pcie_a7 : pcileech_pcie_a7` then double click on the file with the yellow square labelled `i_pcie_7x_0 : pcie_7x_0`.

![image](https://github.com/Silverr12/DMA-CFW-Guide/assets/89455475/5617a8f8-6d5a-44af-8f88-703bc7d1f101)

2. You should now be in a window called "Re-customize IP", in there, press on the `IDs` tab and enter all the IDs you gathered from your donor board, also note that the "SubSystem Vendor ID" Is just the same as your Vendor ID. _(If your donor board is different from a network adapter you may have to adjust some settings in the "Class Code" section below as well.)_

![image](https://github.com/Silverr12/DMA-CFW-Guide/assets/89455475/4b0584ec-9dda-4a2a-a5e1-a6e2eb28c6d1)

To check the class code of your donor card go back to Arbor > scan if needed, else > PCI config > set PCI view to Linear. Your card should be highlighted in green. There will also be a column header called **Class**. Match that with your card.

![image](https://github.com/Silverr12/DMA-CFW-Guide/assets/89455475/24131586-03d6-4b70-9000-16448a4d8944)

3. Also go into the "BARs" tab and set the size value you gathered in step 2, note that the Hex Value shown is not meant to be the same as your bar address. You cannot edit this value.

![image](https://github.com/Silverr12/DMA-CFW-Guide/assets/89455475/1942fa3c-71cf-4466-a9a6-a33b5b38e54d)

the size of my bar was 16kb so 16kb is what you set it as

If the size unit is different change the size unit to accommodate the unit of the bar size



4. Press OK on the bottom right then hit "Generate" on the new window that pops up and wait for it to finish.<br />
![image](https://github.com/Silverr12/DMA-CFW-Guide/assets/48173453/df292771-63c0-4013-9eaf-bb2c39e52539)

5. We will lock the core so that when Vivado synthesises and/or builds our project it will not overwrite some things and allow us to edit some things manually we could only do through the interface before, to do this, navigate to the "Tcl Console" located in the top right of the bottom box and enter into there `set_property is_managed false [get_files pcie_7x_0.xci]`, (to unlock it in the future for any purposes use `set_property is_managed true [get_files pcie_7x_0.xci]`.)


---
# **Steps 5 and 6** are currently under research and revision, so they are not yet complete or final. Please proceed with caution.



## **5. Other Config Space Changes**
> [!TIP]
> Please have a look at [shadow_cfg_space.md](https://github.com/Silverr12/DMA-CFW-Guide/blob/main/Shadow_cfg_space.md), It is a much easier way of doing what is described below

  1. In Vivado, navigate to `pcie_7x_0_core_top` as shown in the image, and use the magnifying glass in the top left of the text editor to search for these different lines to match them to your donor card

![image](https://github.com/Silverr12/DMA-CFW-Guide/assets/48173453/c018b760-cb8f-4c08-9efc-e5a3cdd8ed8d)

#### - Here is a list of variable names in the manual Vivado IP core config correlating to values we have confirmed to **not** break your firmware that you could change to match your donor cards that we've been able to match by name from Arbor. matched by capability, there is: <br />
  - (PM) `PM_CAP_VERSION`, `PM_CAP_D1SUPPORT`,`PM_CAP_AUXCURRENT`, `PM_CSR_NOSOFTRST`
  - (MSI) `MSI_CAP_64_BIT_ADDR_CAPABLE`, 
  - (PCIe) `PCIE_CAP_DEVICE_PORT_TYPE`, `DEV_CAP_MAX_PAYLOAD_SUPPORTED`, `DEV_CAP_EXT_TAG_SUPPORTED`, `DEV_CAP_ENDPOINT_L0S_LATENCY`, `DEV_CAP_ENDPOINT_L1_LATENCY`, `LINK_CAP_ASPM_SUPPORT`, `LINK_CAP_MAX_LINK_SPEED`, `LINK_CAP_MAX_LINK_WIDTH`, `LINK_CTRL2_TARGET_LINK_SPEED`
  - Fields that can be changed in different files or a GUI that I do not yet know about. <br />
    - (PM) `cfg_pmcsr_powerstate`
    - (PCIe) `corr_err_reporting_en`, `non_fatal_err_reporting_en`, `fatal_err_reporting_en`, `no_snoop_en`, `Link Status2: Current De-emphasis`


#### - It is also advised that you change the block locations of some of the capabilities, this can be done by changing the following variables:
  - Capability NEXT Pointers:`CAPABILITIES_PTR`, `MSI_CAP_NEXTPTR`, `PCIE_CAP_NEXTPTR`, `PM_CAP_NEXTPTR` and
  - Capability Pointers: `MSI_BASE_PTR`, `PCIE_BASE_PTR`, `PM_BASE_PTR`

On default pcileech firmware you can locate: **PM at 0x40, MSI at 0x50, and PCIe at 0x60**, The example will be changing them to **PCIe at 0x40, PM at 0xC8 and MSI at 0xD0**, but you can have them at any location really (e.g PCIe at 0x80, PM at 0xD0 and MSI at 0x90) since our computers can and will jump over the empty blocks, all you have to do is make sure the `NEXTPTR`'s line up to the next capability as explained below and that you take note of the capabilities sizes so they don't try to overlap.
- You need your NEXTPTRs lined up starting from your header at 0x00 and going up in the config blocks, for example:
  - If I were to change my capabilities blocks around to `PCIe: 0x40 | PM: 0xC8 | MSI: 0xD0` I would simply assign their associated `BASE_PTR` variables as such to the same value. Always make to start at or above 0x40 as our header ends just before it and also make sure your base ptrs always end on 0, 4, or 8 such as 40, 44 68.
  - Secondly, I would also have to have my header capability pointer `CAPABILITIES_PTR` point to 40 (which it is by default) since it's our lowest/first to be read in this case, then the `PCIE_CAP_NEXTPTR` will point to C8, `PM_CAP_NEXTPTR` to D0 and `MSI_CAP_NEXTPTR` to 00 to finalise it out, and always make sure it's in order from top to bottom as if you try to point backward in the config space your firmware will not work in some cases. (Extended capabilities such as AER, DSN, LTR, etc also require this configuration if you decide to put them in. But you do not point the regular capabilities into them as they are a separate 'set', besides that they follow the same pointer format as your regular capabilities.)


> [!IMPORTANT]
> Once you have completed steps 1-5, you **should, with 98% confidence**, be good to go for systems with lower-end detections
> Your best bet to get past higher-end detection systems is lots of trial and error, many will not reveal their methods unless they want it detected so you are mostly on your own there unfortunately.

  
## **6. TLP Emulation**
**For now, see:**
1. [Ekknod's bar controller config](https://github.com/ekknod/pcileech-wifi/blob/main/src/pcileech_tlps128_bar_controller.sv#L850) between line 850-896 for an example
2. [One of Yxlnq's bar controllers](https://github.com/yxlnqs/diviner-full-emu-v2/blob/5a177e34ae5dae94bb2c023e38301af425ca6e4b/src/pcileech_tlps128_bar_controller.sv#L850)

Notes to consider:
 

1. You have two options for obtaining the register addresses for the device you're emulating, your options are:
- Searching to see if your driver is already open source and looking through it, a good starting point is this [Wikipedia](https://en.wikipedia.org/wiki/Comparison_of_open-source_wireless_drivers) page that lists open-source/reverse-engineered wireless drivers that you could take values from for your firmware
- **[Hard]** Using a reverse-engineering program of your choice to find the details of your driver for your donor card, you can find the location of the installed driver by navigating to your device in the device manager, going to Properties>Driver>Driver Details, and it should normally be the only .dll file in there. (Mind you unless you are already well versed in the area of reverse engineering, don't commit your time to this)

### Resources for general understanding & TLP emulation
1. https://fpgaemu.readthedocs.io/en/latest/emulation.html
2. https://docs.xilinx.com/v/u/en-US/pcie_blk_plus_ug341
3. https://www.fpga4fun.com/PCI-Express4.html
4. https://www.xillybus.com/tutorials/pci-express-tlp-pcie-primer-tutorial-guide-1
5. https://ctf.re (<-amazing one)

## **7. Interrupts**
Please see [https://github.com/kilmu1337/VGK-DMA-BYPASS](https://github.com/kilmu1337/VGK-DMA-BYPASS) for now

## **8. Building, Flashing & Testing**

1. Run `source vivado_build.tcl -notrace` in the tcl console to generate the file you'll need to flash onto your card<br />
   - You'll find the file in `pcileech_squirrel/pcileech_squirrel.runs/impl_1` named "pchileech_squirrel_top.bin"<br />
2. Follow the steps on the [official LambdaConcept guide for flashing](https://docs.lambdaconcept.com/screamer/programming.html) **<sub>REMINDER: ONLY FOR SQUIRREL</sub>**
3. Run a DMA speed test tool from your second computer <sub>(There is a link and download in the discord server)</sub> to verify your firmware is working and reading as it should be.
4. Dump and compare the config space of your new firmware to the **known** signed pcileech default seen below to see if it's overly similar. You should most definitely be right about some values being the same, you have to think about the fact that apart from the serial number and maybe bar address, the configuration space of one type of (for example) network card is going to be the same across all of them. GLHF

This is the signature one of the lower-end detection systems supposedly scan for in the config space of the PCIe device:
[More info here](https://dma.lystic.dev/anticheat-evasion/detection-vectors)<br>
     `40: 01 48 03 78 08 00 00 00 05 60 80 00 00 00 00 00`<br />
     `60: 10 00 02 00 e2 8f XX XX XX XX XX XX 12 f4 03 00`<br />
     ("XX" are bytes that they do not care about)

Another form of detection that may or may not be implemented that could be blocking your firmware is reading your device history, this can be cleaned by following [this](https://dma.lystic.dev/anticheat-evasion/clearing-device-history) post.

> [!CAUTION]
> **There is a good chance that on your first flash if you went about some of the more 'harder' to navigate steps it will mess something up, don't worry, and look at the troubleshooting below.**<br />

### Flashing troubleshooting
- If you mess up your CFW and your game PC won't fully "boot", be because of bios hang or other reasons, you will be able to flash new firmware onto it from your second computer if the card is still powered (normally indicated by LEDs). If your main computer won't stay powered on, you have to buy a PCIe riser that will allow you to power your DMA card without it 'communicating' **(NOT RECOMMENDED: if a riser is unavailable you can hotplug the dma card in after your computers fully booted then flash the card, be warned however as there have been rare reports of motherboard corruptions due to this)**
- There are flat-out some motherboards that will be incompatible with some firmware, in some cases this can be resolved my making a manual memory map.

### 'Dysfunctional' firmware troubleshooting
- If your speed test prompts something along the lines of `tiny PCIe algorithm`, you have made a mistake somewhere in your configuration space. Your card *will* still function but reads will be slower than they should be which can severely impact performance.
- Changing some functions below acceptable bounds most likely named something including payload/size/speed **can** also slow down the reading speed of your card. The best course of action is to set max read request/payload sizes to 4KB/highest available
- Some motherboards will simply be incompatible with some firmware, most reports have been on gigabyte mobos.
- Sometimes your firmware will allow your device to work but cause a massive slowdown then BSOD your computer if it tries to read it with Arbor or Device Manager. Unfortunately, I don't know exactly where you need to go wrong for this to happen so I recommend re-doing your whole firmware. I suggest keeping a stable firmware that works on your second computer in case this happens.
- Are your changes not saving when making a new .bin file? Try deleting your `pcileech_squirrel.runs` & `pcileech_squirrel.cached` folder or even making and working in a new copy of the stock pcileech-fpga folder every new firmware as good practice


### Once you've read through all this,
If you have any questions, problems with your firmware or suggestions, feel free to join my Discord for support.
[![Discord Banner](https://discord.com/api/guilds/1268006421002453124/widget.png?style=banner2)](https://discord.gg/m9zmBwn8Y5)


### Additional Credits
Ulf Frisk for [pcileech](https://github.com/ufrisk/pcileech) <br />
Ekknod for his [custom pcileech config](https://github.com/ekknod/pcileech-wifi)<sub>(You may be able to use this as a base to start off of as well!)</sub> <br />
Garagedweller's [Unknown Cheats thread](https://www.unknowncheats.me/forum/anti-cheat-bypass/613135-dma-custom-firmware-guide.html) that inspired me to make this in the first place and whom I credit my interest in this topic to.

### Sponsor this project
If you feel this guide has helped you enough to warrant a monetary donation, feel free to donate here: <br />
USDT/trc20: `TDa8PUwAdD9rg84ythjXjN52s8UeaejnFN` <br />
![usdtaddr](https://github.com/Silverr12/DMA-CFW-Guide/assets/48173453/36a8a6d6-1edd-4289-96b9-a9003a7c4a26)<br />

LTC: `MMxWW2n5pTbWoY9EakDaTiQ7HKBJy7sxDh`<br />
![ltcaddr](https://github.com/Silverr12/DMA-CFW-Guide/assets/48173453/e243973f-7b84-42a9-b78a-19a7a12aac98)<br />
or just starring the repo helps **immensely** too <3 <br />
<sub>also sponsor the [man who's making this all possible](https://github.com/ufrisk)<br />

