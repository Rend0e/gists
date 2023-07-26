# Fix AMD GPU high idle power MCLK (vram / memory clock) stuck at 96 MHz / 1000 MHz for high refresh rates on Arch Linux Wayland & Xorg

On certain resolutions & refresh rates or multi-monitor setups, you might have noticed that your GPU MCLK (vram / memory clock) is stuck at the highest clock frequency (1000 MHz) [[1](https://gitlab.freedesktop.org/drm/amd/-/issues/1403)] [[2](https://gitlab.freedesktop.org/drm/amd/-/issues/2646)] causing higher GPU idle power draw.  On Linux kernel 6.4.x, AMDGPU MCLK (vram/memory) clocks at the lowest, causing major FPS drops while gaming [[1](https://gitlab.freedesktop.org/drm/amd/-/issues/2657)] [[2](https://gitlab.freedesktop.org/drm/amd/-/issues/2611)]. This is likely due to a monitor not using Coordinated Video Timings (CVT) with a low V-Blank value for the affected resolutions & refresh rates.  The higher clocking behavior is due to:

> [Well, the reason the clocks get forced to max in some cases is to avoid the flickering you are seeing.  There is a certain latency required to change the mclk.  The hardware needs to hide that mclk switch during some blanking period of the display otherwise you would see flickering.  If the blanking periods are too short or not alignable (in the case of multiple displays), the driver has to use a fixed mclk to avoid the flickering.  In your case, it appears that the latency of switching to/from the 96Mhz mclk is right on the edge of what is possible given the monitors blanking periods.  **You may need to slightly lengthen the hblank or vblank periods in your display's modeline**.](https://gitlab.freedesktop.org/drm/amd/-/issues/1403#note_1190208)

The monitor's non standard timings values were likely set that way to keep its pixel clock values within display cable bandwidth constraints that are more relevant on higher resolutions & refresh rates, which is why this issue tends to occur on high resolutions & refresh rates.

### HDMI

| Version  | Limit        | Data rate | Bandwidth |
| -------- | ------------ | --------- | --------- |
| 1.0-1.2a | **165 MHz**  | 3.96 Gbps | 4.95 Gbps |
| 1.3-1.4b | **340 MHz*** | 8.16 Gbps | 10.2 Gbps |
| 2.0-2.0b | **600 MHz**  | 14.4 Gbps | 18 Gbps   |

### DisplayPort

Check https://www.monitortests.com/blog/common-pixel-clock-limits/

## Check for issue

You can check for this issue by monitoring your GPU MCLK/VRAM/Memory clock value with `nvtop`, `mangohud` or `cat /sys/class/drm/card0/device/pp_dpm_mclk` *(Note: try card1 if file is not found)*.  If it's stuck at the highest clock (1000 MHz), you would also notice a higher idle power draw. If the values are stuck and do not adapt to the GPU workload, try lowering the display resolution refresh rate to see if it resolves the issue. If the issue only occurs in specific resolutions & refresh rates, then it's likely that the monitor's EDID has a too low v-blank value for the specific resolution & refresh rate. 

## Fix for Xorg [[1](https://gitlab.freedesktop.org/drm/amd/-/issues/2646#note_1970946)]

1. Use [cvt_modeline_calculator_12](https://github.com/kevinlekiller/cvt_modeline_calculator_12) to obtain modeline values for your resolution & refresh rate with CVT v1.2 reduced blanking timings:

   ```shell
   curl https://raw.githubusercontent.com/kevinlekiller/cvt_modeline_calculator_12/master/cvt12.c --output cvt12.c
   gcc cvt12.c -O2 -o cvt12 -lm -Wall
   ./cvt12 1920 1080 144 -b
   ```

   *Note: change 1920 1080 144 to your resolution and refresh rate, mine is 1920x1080@144.*

   <u>Output Example:</u>

   ```shell
   # 1920x1080 @ 144.000 Hz Reduced Blank (CVT) field rate 144.000 Hz; hsync: 166.608 kHz; pclk: 333.22 MHz
   Modeline "1920x1080_144.00_rb2"  333.22  1920 1928 1960 2000  1080 1143 1151 1157 +hsync -vsync
   ```

2. Use xrandr to add the custom resolution modeline:
   ```
   xrandr --newmode "MCLK-fix" 333.22  1920 1928 1960 2000  1080 1143 1151 1157 +hsync -vsync
   xrandr --addmode HDMI-2 "MCLK-fix"
   ```

   *Note: replace values after "MCLK-fix" with the modeline values output from step 1 and replace HDMI-2 with your connector (run `xrandr`)*.

3. Select the resolution and check if MCLK (VRAM / Memory) clock frequency properly adjusts to GPU workloads.

## Fix for Wayland

Before proceeding, try temporarily switching to xorg to test if the xorg solution works. You can skip steps 1-3 if you find your resolution & refresh rate in [Some CVT-RB v2 EDID values (can use this to skip steps 1-3)](#some-cvt-rb-v2-edid-values).

1. Use [cvt_modeline_calculator_12](https://github.com/kevinlekiller/cvt_modeline_calculator_12) to obtain modeline values for your resolution & refresh rate with CVT v1.2 reduced blanking timings:

   ```shell
   curl https://raw.githubusercontent.com/kevinlekiller/cvt_modeline_calculator_12/master/cvt12.c --output cvt12.c
   gcc cvt12.c -O2 -o cvt12 -lm -Wall
   ./cvt12 1920 1080 144 -b
   ```

   *Note: change 1920 1080 144 to your resolution and refresh rate, mine is 1920x1080@144Hz*:

   <u>Output Example:</u>

   ```shell
   # 1920x1080 @ 144.000 Hz Reduced Blank (CVT) field rate 144.000 Hz; hsync: 166.608 kHz; pclk: 333.22 MHz
   Modeline "1920x1080_144.00_rb2"  333.22  1920 1928 1960 2000  1080 1143 1151 1157 +hsync -vsync
   ```

2. Parse the modeline to edid-generator to see its values in EDID binary format:
   ```shell
   git clone https://github.com/akatrevorjay/edid-generator.git
   cd edid-generator
   yay -S zsh edid-decode-git automake dos2unix # Required dependencies for edid-generator
   zsh # Need to enter zsh shell
   ./modeline2edid - <<< 'Modeline "MCLK-fix" 333.22  1920 1928 1960 2000  1080 1143 1151 1157 +hsync -vsync'
   make
   ```

   *Note: replace the values after "MCLK-fix" with the modeline values output from step 1.*

3. Use wxedid to open the generated edid to take note of its values:

   ```shell
   yay -S wxedid
   ```

   1. Open the MCLK-Fix.bin file with wxEDID (`/usr/bin/wxedid`).
   2. You may get an error message, so follow its prompt: Options->"Ignore EDID Errors" and then "Reparse EDID buffer" to view the data anyway.
   3. Find DTD: Detailed Timing Descriptor.
   4. Take note of the values or take a screenshot.
      *Example:*
      ![3.3](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/3.3.png)

   

4. Obtain your monitor's EDID binary file: 
   ```shell
   ls /sys/class/drm/ # Use xrandr, wlr-randr, gnome-randr-rust to find the correct connector.
   cat /sys/class/drm/card1-HDMI-A-2/edid # Check for an output (monitor name may be visible). 
   cp /sys/class/drm/card1-HDMI-A-2/edid edid.bin # Replace card1-HDMI-A-2 with your connector.
   ```

   ![4](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/4.png)

5. Use wxEDID to edit mointor's EDID to MCLK-fix's values:

   1. Open the edid.bin file with wxEDID (`/usr/bin/wxedid`)

   2. Find DTD: Detailed Timing Descriptor

   3. Click the DTD Constructor Tab
      

      ![5.3](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/5.3.png)
      
   4. Check if X-res, V-res and V-Refresh matches your resolution and refresh rate; if not, go back to the EDID tab, then keep checking another DTD  until you find it:
      ![5.4](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/5.4.png)

   5. Go back to the EDID tab and change the values to match the ones in MCLK-Fix.bin:
   
      | ![5.5-Original](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/5.5-Original.png) | ![5.5-Fixed](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/5.5-Fixed.png) |
      | ------------------------------------------------------------ | ------------------------------------------------------------ |
   
   6. Click Option on the panel -> Assemble EDID:
      
      
      ![6](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/6.png)
      
   7. Click File on the panel -> Save EDID binary (overwrite edid.bin):
      
      
      ![7](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/7.png)
   
6. Load EDID at boot for the particular monitor:

   1. Place the edid file in `/lib/firmware/edid`
      ```shell
      sudo mkdir -p /lib/firmware/edid
      sudo cp edid.bin /lib/firmware/edid/edid_modified.bin
      ```

   2. Include the custom EDID file in the initramfs
      ```shell
      sudoedit /etc/mkinitcpio.conf
      
      FILES=(/lib/firmware/edid/edid_modified.bin)
      ```

   3. Add the kernel parameter `drm.edid_firmware=HDMI-A-2:edid/edid_modified.bin` (replace HDMI-A-2 with your connector).

7. Reboot and check if MCLK (VRAM / Memory) clock frequency properly adjusts to GPU workloads.

## Some CVT-RB v2 EDID values

#### 1920x1080@60Hz

`Modeline "1920x1080_60.00_rb2"  133.32  1920 1928 1960 2000  1080 1097 1105 1111 +hsync -vsync`

![1920x1080_60.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/1920x1080_60.00_rb2.png)

#### 1920x1080@120Hz 

`Modeline "1920x1080_120.00_rb2"  274.56  1920 1928 1960 2000  1080 1130 1138 1144 +hsync -vsync`

![1920x1080_120.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/1920x1080_120.00_rb2.png)

#### 1920x1080@144Hz 

`Modeline "1920x1080_144.00_rb2"  333.22  1920 1928 1960 2000  1080 1143 1151 1157 +hsync -vsync`

![1920x1080_144.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/1920x1080_144.00_rb2.png)

#### 1920x1080@160Hz 

`Modeline "1920x1080_160.00_rb2"  373.12  1920 1928 1960 2000  1080 1152 1160 1166 +hsync -vsync`

#### ![1920x1080_160.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/1920x1080_160.00_rb2.png)

#### 1920x1080@240Hz

#### `Modeline "1920x1080_240.00_rb2"  583.20  1920 1928 1960 2000  1080 1201 1209 1215 +hsync -vsync`

![1920x1080_240.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/1920x1080_240.00_rb2.png)

#### 2560×1440@60Hz 

`Modeline "2560x1440_60.00_rb2"  234.59  2560 2568 2600 2640  1440 1467 1475 1481 +hsync -vsync`

![2560x1440_60.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/2560x1440_60.00_rb2.png)

#### 2560×1440@120Hz 

`Modeline "2560x1440_120.00_rb2"  483.12  2560 2568 2600 2640  1440 1511 1519 1525 +hsync -vsync`

![2560x1440_120.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/2560x1440_120.00_rb2.png)

#### 2560×1440@144Hz 

`Modeline "2560x1440_144.00_rb2"  586.59  2560 2568 2600 2640  1440 1529 1537 1543 +hsync -vsync`

![2560x1440_144.00_rb2](https://rend0e.github.io/gists/Fix_MCLK_Clock_Stuck_Arch_Linux_Guide/images/2560x1440_144.00_rb2.png)
