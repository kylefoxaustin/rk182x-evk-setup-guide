# Firefly RK182X EVK — From Unboxing to Running an LLM on the NPU

A pedantic, no-prior-knowledge-assumed setup guide for the **Firefly RK182X 3D RAM Stacking Development Kit** (AIO-GS1N2 carrier board + Core-3588JD4 main SoM + RK1820/RK1828 SO-DIMM AI accelerator).

This guide takes you from *"I just opened the box and I have no idea what any of this is"* all the way to *"I am typing a question and a large language model is answering me, and the answer is being computed by a chip the size of a stick of laptop RAM."*

> **Note:** This guide is deliberately slow and explicit. If you already know Rockchip boards, skim the headings. If you have never touched one, read every line — nothing is assumed.

---

## Table of Contents

1. [What is this kit and what are we building toward](#1-what-is-this-kit-and-what-are-we-building-toward)
2. [What you need before starting](#2-what-you-need-before-starting)
3. [Unboxing and physical assembly](#3-unboxing-and-physical-assembly)
4. [Option A: First boot with the preinstalled OS](#4-option-a-first-boot-with-the-preinstalled-os-path-of-least-resistance)
5. [Option B: Flashing a fresh Firefly Debian 12 image](#5-option-b-flashing-a-fresh-firefly-debian-12-image)
6. [Verifying the RK1828 is alive](#6-verifying-the-rk1828-is-alive)
7. [Installing RKLLM-Toolkit on the host PC](#7-installing-rkllm-toolkit-on-the-host-pc)
8. [Running the DeepSeek-R1-Distill-Qwen-1.5B hello-world demo](#8-running-the-deepseek-r1-distill-qwen-15b-hello-world-demo-end-to-end)
9. [Where to go next](#9-where-to-go-next)
10. [Glossary](#10-glossary)
11. [Useful links](#11-useful-links)

---

## 1. What is this kit and what are we building toward

This kit is a small computer designed to run AI models — specifically large language models (the kind of AI that reads and writes text, like a chatbot) — very fast and using very little power, right on your desk with no internet or cloud account required. It is made of three stacked parts: a big green base board (the **carrier board**, model **AIO-GS1N2**) that supplies power and all the ports; a small plug-in computer-on-a-card (the **main module**, or **SoM**, model **Core-3588JD4**) that runs a normal Linux operating system; and a second plug-in card that looks exactly like a stick of laptop memory (the **AI accelerator**, model **RK1820** or **RK1828**) whose only job is to crunch AI math extremely quickly. By the end of this guide you will have assembled these parts, booted Linux on the main module, confirmed the AI accelerator is working, and run a real large language model on it that types answers back to you faster than you can read them.

---

## 2. What you need before starting

Before you touch the board, gather everything below. Missing one item (especially the right cable) is the single most common reason people get stuck on step one.

### 2.1 The kit itself

Make sure your box contains all three pieces. If you bought the kit assembled, the modules may already be seated — that is fine, we will verify in Section 3.

| Part | Model | What it is |
| --- | --- | --- |
| Carrier board | AIO-GS1N2 | The large base board with all the ports |
| Main module (SoM) | Core-3588JD4 | RK3588 octa-core computer-on-module that runs Linux |
| AI accelerator | RK1820 **or** RK1828 | SO-DIMM card that runs the LLM. RK1820 = 2.5 GB on-chip RAM (LLMs up to ~3B params), RK1828 = 5 GB on-chip RAM (LLMs up to ~7B params). Both deliver 20 TOPS (INT8). |

> **Note:** This guide assumes you have the **RK1828** (the 5 GB version), because that is what most kits ship with and it can run the 7B-class models we point you to in Section 9. Everything also works with the RK1820 — you are simply limited to smaller models.

### 2.2 Power

- The **power supply** that came in the box. The carrier board accepts **24 V DC** through a barrel jack, or **12 V / 48 V** through an ATX-style connector. Use the adapter Firefly shipped; do not substitute a random laptop charger.

> **Warning:** Plugging in a power supply with the wrong voltage or polarity can permanently destroy the board. If your kit shipped without a supply, check the [specification PDF](#11-useful-links) for the exact voltage and barrel-jack polarity before buying one.

### 2.3 Cables and peripherals

- **One HDMI cable** and an **HDMI monitor or TV** — so you can see the desktop on first boot (Option A).
- **A USB keyboard and mouse** — plug into any USB 3.0 port.
- **One USB-A to USB-C cable** (data-capable, not charge-only) — this connects the board to your host PC for flashing firmware (Option B). The USB-C end goes into the board's **OTG / upgrade** port.
- **One USB-to-TTL serial adapter (3.3 V)** *(optional but strongly recommended)* — lets you watch the boot messages on the **debug UART** when nothing appears on HDMI. This is your lifeline when something goes wrong.
- **An Ethernet cable** plugged into your router — far easier than fighting with Wi-Fi on first setup.
- *(Optional)* A **microSD (TF) card**, 16 GB or larger, if you prefer the SD-card flashing method.

### 2.4 A host PC (for Option B and for the AI toolkit)

You need a separate "host" computer — your normal laptop or desktop — to download firmware, run the AI model-conversion toolkit, and talk to the board.

- **Strongly preferred:** a PC running **Ubuntu 20.04 or 22.04** (x86-64). The Rockchip flashing tool (`upgrade_tool`) and the AI toolkit (`RKLLM-Toolkit`) are built and tested for Linux. This guide's commands assume Ubuntu.
- **Windows** works for *flashing only*, using the GUI tool **RKDevTool** (noted as an alternative in Section 5). The AI toolkit still wants Linux.
- At least **30 GB of free disk space** (model weights are large) and a working internet connection.

### 2.5 Accounts to create

- A **Hugging Face account** (free) at <https://huggingface.co> — you will download the DeepSeek model weights from there in Section 8.
- *(Optional)* A **GitHub account** if you want to clone Rockchip's repositories over SSH; HTTPS cloning works without one.

> **Tip:** Do all the downloading on your host PC over a wired or fast connection. The model weights for even a "small" 1.5B model are several gigabytes.

---

## 3. Unboxing and physical assembly

We will now physically build the kit. Work on a hard, non-carpeted surface. Touch a grounded metal object (or wear an anti-static strap) before handling the boards — static electricity can silently kill these chips.

> **Warning:** Every step in this section must be done with the **power supply unplugged**. Never insert or remove a module while the board has power.

### Step 3.1 — Confirm power is disconnected

Look at the barrel jack and ATX connector on the carrier board. Make sure nothing is plugged in and the power LED is off.

**How you know it worked:** No lights are on anywhere on the board.

### Step 3.2 — Seat the main module (Core-3588JD4 SoM)

The main module is the larger plug-in board. It connects to the carrier through high-density board-to-board connectors (rows of tiny pins), **not** a slot you slide into at an angle.

1. **Step 3.2.1** — Orient the module so its connectors line up directly above the matching connectors on the carrier board. The mounting screw holes on the module should line up with the standoffs on the carrier.
2. **Step 3.2.2** — Hold the module flat and parallel to the carrier, then press straight down evenly over the connectors with your thumbs. You will feel and hear a firm *click* as the connectors mate.
3. **Step 3.2.3** — Fasten the module to the standoffs with the screws provided. Snug, not gorilla-tight.

**How you know it worked:** The module sits flat and level (no corner lifted), there is no visible gap at the board-to-board connectors, and the screws hold it firmly in place.

> **If this didn't work:** If one corner is higher than the others, the connectors are not fully mated. Unscrew, lift straight up, re-align, and press down evenly again. Never force it crooked — bent connector pins are not repairable.

### Step 3.3 — Seat the RK182X AI accelerator (SO-DIMM)

The AI accelerator (RK1820 or RK1828) is the small card shaped exactly like a stick of laptop RAM. It goes into a **SO-DIMM slot**, which is the angled-insertion type — the same motion as installing laptop memory.

1. **Step 3.3.1** — Find the SO-DIMM slot on the board. Note the small notch (key) in the card's gold edge connector; it only lines up one way with the bump in the slot. Do not force the card in backwards.
2. **Step 3.3.2** — Hold the card at roughly a **30-degree angle** to the board and slide the gold edge fully into the slot. When seated correctly, the gold contacts almost disappear into the slot.
3. **Step 3.3.3** — Press the top edge of the card **down toward the board** until the two metal retaining clips on the sides snap up and lock into the notches on the card's edge.

**How you know it worked:** Both side clips have clicked closed and are holding the card, and the card is now lying nearly flat and level instead of sticking up at an angle.

> **If this didn't work:** If the clips won't snap, the card isn't pushed deep enough into the slot. Open the clips, remove the card, and re-insert it more firmly at the angle before pressing down. The card should slide in with light, even pressure — if it fights you, check the notch orientation.

### Step 3.4 — Attach the heatsink / fan

The AI accelerator gets warm under load, and the main module gets *hot*. A heatsink (and usually a fan) is included.

1. **Step 3.4.1** — Peel the protective film off any thermal pad on the underside of the heatsink. The pad must make direct contact with the chip; missing this is a common mistake.
2. **Step 3.4.2** — Place the heatsink squarely on top of the module/accelerator and secure it with the spring-loaded screws or clips provided.
3. **Step 3.4.3** — If there is a separate fan, plug its connector into the **fan header** on the carrier board.

**How you know it worked:** The heatsink does not wobble, and the fan (if present) is plugged into a header.

> **Warning:** Running these modules — *especially* the RK1828 during LLM inference — without a heatsink will cause thermal throttling within seconds and can damage the silicon. Do not skip the heatsink.

### Step 3.5 — Connect the peripherals

With everything still powered off:

1. **Step 3.5.1** — Plug the **HDMI cable** from the board into your monitor.
2. **Step 3.5.2** — Plug in the **USB keyboard and mouse**.
3. **Step 3.5.3** — Plug the **Ethernet cable** into the board and your router.
4. **Step 3.5.4** — *(Optional)* Connect the **USB-to-TTL serial adapter** to the debug UART pins (see the wiki's interface diagram for which three pins are GND / TX / RX).

**How you know it worked:** Everything is physically connected and the power supply is still unplugged.

### Step 3.6 — Connect power (but do not turn on yet)

Plug the Firefly power supply into the barrel jack, then into the wall.

**How you know it worked:** A standby/power LED on the board lights up, indicating the board has power but has not been told to boot.

---

## 4. Option A: First boot with the preinstalled OS (path of least resistance)

Your kit ships with a working OS already flashed onto the main module. The fastest way to confirm the hardware is alive is simply to turn it on. **Do this first**, before any flashing — it proves the board works out of the box.

### Step 4.1 — Power on

Press the **power button** on the carrier board (consult the wiki's interface diagram if you are unsure which button is power versus reset versus recovery).

**How you know it worked:** Within a few seconds the power LED is solid, the fan spins up, and after 20–40 seconds a Linux desktop (or login prompt) appears on your HDMI monitor.

> **Tip:** If the HDMI screen stays black but the fan is spinning, the board is probably booting fine and HDMI just didn't negotiate. Try a different HDMI cable/monitor, or watch the boot text on the serial console (Section 5.6 explains how to open a serial console).

### Step 4.2 — Log in

If you reach a graphical desktop, you may be logged in automatically. If you hit a text or graphical login prompt, use Firefly's default Debian credentials:

```
username: firefly
password: firefly
```

**How you know it worked:** You are looking at a desktop or a shell prompt that accepts commands.

> **If this didn't work:** If the default credentials are rejected, the image on your board may differ. Check the exact default login for your image's release notes on the [Firefly download page](#11-useful-links). The `root` password on many Firefly images is also `firefly`.

### Step 4.3 — Open a terminal and confirm you are on the RK3588 main module

Open a terminal (right-click desktop → Open Terminal, or use the serial console) and run a command that prints the CPU/SoC info. This confirms you are talking to the Core-3588JD4 main module.

```
cat /proc/cpuinfo
```

**How you know it worked:** You see eight CPU cores listed (the RK3588 is octa-core) and Rockchip/ARM identifiers.

If your board boots and logs in, the hardware is healthy. You can now either stay on the preinstalled OS, or proceed to **Option B** to flash a known-clean Debian 12 image (recommended if you want a setup that exactly matches this guide and the Firefly AI documentation).

---

## 5. Option B: Flashing a fresh Firefly Debian 12 image

Flashing means erasing the main module's storage and writing a fresh operating system image to it. We do this over the USB cable from your Linux host PC using Rockchip's command-line tool, `upgrade_tool`. (Windows users: skip to the RKDevTool note in Section 5.5.)

> **Warning:** Flashing **erases everything** currently on the board's eMMC storage. If you saved anything during Option A, back it up first.

### 5.1 Understand the two special boot modes (read this before flashing)

To flash, the board must not be running its normal OS — it has to be sitting in a special "listen for commands" mode. There are two such modes, and knowing the difference saves a lot of grief:

- **Loader mode** — The board's bootloader is intact and *running*; it pauses and waits for flashing commands from the host. This is the **normal, preferred** mode for a routine firmware update. It is easiest to enter and works whenever the existing bootloader is healthy.
- **MaskRom mode** — A tiny program baked into the chip's silicon takes over *before* any bootloader runs. You use this as a **rescue** mode: when the bootloader is corrupted, when a flash failed halfway, or when the board won't enter Loader mode. MaskRom can always recover a board that is otherwise bricked.

> **Note:** Rule of thumb — try **Loader mode** first for everyday flashing. Fall back to **MaskRom mode** only if Loader mode won't cooperate or the board is unresponsive.

### 5.2 Download the firmware image

1. **Step 5.2.1** — On your host PC, open the official Firefly download page for this kit:

   <https://en.t-firefly.com/doc/download/369.html>

2. **Step 5.2.2** — Find the **"RK182X Development Kit -- Firmwares"** section and download the latest **Debian 12** image for the **AIO-GS1N2 / Core-3588JD4**. It will arrive as a compressed file (for example a `.img.gz` or `.7z`/`.zip` archive).

3. **Step 5.2.3** — Extract it until you have a single `update.img` (or a similarly named `.img`) file. Note its full path.

**How you know it worked:** You have a single, several-gigabyte `.img` file on disk and you know where it is.

> **If this didn't work:** If the download page looks unfamiliar, use the navigation to find the AIO-GS1N2-RK182X product, then its "Firmwares" / "Resources" section. The matching documentation index is <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/index.html>.

### 5.3 Install `upgrade_tool` on the Linux host

`upgrade_tool` (also called the "Linux Upgrade Tool") is Rockchip's command-line flashing utility. Firefly distributes it from the same download page, under **Tools**.

1. **Step 5.3.1** — Install the USB library it depends on:

   ```
   sudo apt update
   ```

   ```
   sudo apt install -y libusb-1.0-0 libusb-1.0-0-dev
   ```

   **How you know it worked:** `apt` finishes with no errors.

2. **Step 5.3.2** — Download the **Linux_Upgrade_Tool** package from the [Tools section of the download page](https://en.t-firefly.com/doc/download/369.html) and extract it. Inside you will find a binary named `upgrade_tool`.

3. **Step 5.3.3** — Install it system-wide so you can call it from anywhere (adjust the path to where you extracted it):

   ```
   sudo cp ./Linux_Upgrade_Tool/upgrade_tool /usr/local/bin/
   ```

   ```
   sudo chmod +x /usr/local/bin/upgrade_tool
   ```

4. **Step 5.3.4** — Confirm it runs:

   ```
   upgrade_tool -v
   ```

   **How you know it worked:** It prints a version number instead of "command not found."

> **Tip:** Always run `upgrade_tool` with `sudo`. Without root, it cannot claim the USB device and you will see "device not found" even when the board is connected correctly.

### 5.4 Enter the flashing mode and flash

1. **Step 5.4.1 — Connect the USB cable.** Plug the USB-A end into your host PC and the USB-C end into the board's **OTG / upgrade** port (the one wired for flashing). Leave the board's power supply connected.

2. **Step 5.4.2 — Enter Loader mode.** The board has a small **RECOVERY** (also called **MASKROM/RECOVERY**) button and a **RESET** button — check the silkscreen labels next to the buttons, or the wiki interface diagram. To enter Loader mode:
   - Press and **hold the RECOVERY button**.
   - While still holding it, **briefly press and release RESET** (or, if the board is off, tap the power button).
   - **Keep holding RECOVERY for about 2–3 seconds** after the reset, then release.

3. **Step 5.4.3 — Confirm the host sees the board.** On the host, ask `upgrade_tool` to list connected devices:

   ```
   sudo upgrade_tool LD
   ```

   **How you know it worked:** The output lists one device and labels it **`Loader`** (for example: `DevNo=1 ... Mode=Loader`). You can cross-check at the USB level — Rockchip devices appear under vendor ID `2207`:

   ```
   lsusb | grep -i 2207
   ```

   If you see a line containing `ID 2207:` , the host is talking to the board.

   > **If this didn't work:** If `LD` shows nothing, the board isn't in a flashing mode or the cable is charge-only. Try a different (data-capable) USB cable, make sure you used `sudo`, and repeat the button sequence. If Loader mode never appears, use **MaskRom mode** instead: hold the **MASKROM** button (some boards label the recovery button as MASKROM) while applying power/reset, then run `sudo upgrade_tool LD` again — it should now report `Mode=Maskrom`.

4. **Step 5.4.4 — Flash the full firmware image.** With the device showing in `LD`, write the image (replace the path with your actual `update.img`):

   ```
   sudo upgrade_tool UF /path/to/update.img
   ```

   `UF` means "Upgrade Firmware" — it writes the complete image (bootloader, partition table, and root filesystem) in one shot.

   **How you know it worked:** A progress percentage climbs to 100%, you see messages like `Download Firmware Data...` and finally `Upgrade firmware ok`, and the board automatically reboots into the new OS.

   > **If this didn't work:** If flashing stalls or errors out partway, the board may now have a half-written (corrupt) bootloader. This is exactly what MaskRom mode is for: enter MaskRom mode (Step 5.4.3 fallback) and run the `UF` command again. A MaskRom flash recovers a board that won't otherwise boot.

### 5.5 Windows alternative: RKDevTool

If you must flash from Windows, use Rockchip's GUI tool **RKDevTool** (download it from the **Tools** section of the [Firefly download page](https://en.t-firefly.com/doc/download/369.html)), together with the **DriverAssistant** package to install the Rockchip USB driver.

1. Install the Rockchip USB driver via **DriverAssistant** (run `DriverInstall.exe` → *Install Driver*).
2. Put the board into Loader or MaskRom mode using the same button sequence as Step 5.4.2/5.4.3.
3. Open **RKDevTool**. At the bottom it should say **"Found One LOADER Device"** (or **"Found One MASKROM Device"**).
4. Go to the **Upgrade Firmware** tab → **Firmware** button → select your `update.img` → click **Upgrade**.

**How you know it worked:** The log pane shows download progress and ends with "Upgrade Done" / "Success," then the board reboots.

### 5.6 First boot, login, and network setup

1. **Step 5.6.1 — Watch it boot.** After the flash, the board reboots on its own. Watch your HDMI monitor (or, better, the serial console).

   To open the serial console from your host (if you wired up the USB-TTL adapter), install a terminal program and connect at 1,500,000 baud (Rockchip's standard debug baud rate):

   ```
   sudo apt install -y minicom
   ```

   ```
   sudo minicom -D /dev/ttyUSB0 -b 1500000
   ```

   **How you know it worked:** You see kernel boot messages scrolling, ending at a login prompt.

2. **Step 5.6.2 — Log in** with the default Debian credentials:

   ```
   username: firefly
   password: firefly
   ```

   **How you know it worked:** You reach a shell prompt or desktop.

3. **Step 5.6.3 — Bring up the network.** With the Ethernet cable plugged in, confirm you have an IP address:

   ```
   ip addr
   ```

   If you do not see an address on the `eth0` interface, request one:

   ```
   sudo dhclient eth0
   ```

   Then verify you can reach the internet:

   ```
   ping -c 3 8.8.8.8
   ```

   **How you know it worked:** `ip addr` shows an IPv4 address on `eth0`, and `ping` reports replies with `0% packet loss`.

   > **If this didn't work:** If there is no IP, check the cable and that your router's DHCP is on. For Wi-Fi instead, use `nmtui` (a text menu): run `sudo nmtui`, choose *Activate a connection*, and pick your network.

4. **Step 5.6.4 — Update the package list** so you can install tools later:

   ```
   sudo apt update
   ```

   **How you know it worked:** `apt` fetches package lists from the Debian and Firefly repositories without network errors.

---

## 6. Verifying the RK1828 is alive

Booting Linux proves the *main module* works. Now we confirm the *AI accelerator* (the RK1820/RK1828 SO-DIMM) is detected and healthy. **Run these commands on the board** (over HDMI terminal or serial console), not on the host PC.

The primary tool is **`rknn-smi`** — think of it as the "task manager" for the AI accelerator (analogous to `nvidia-smi` for NVIDIA GPUs). On the Firefly Debian image it is preinstalled.

### Step 6.1 — Check the rknn-smi software version

```
sudo rknn-smi -v
```

**How you know it worked:** It prints a software/driver version string instead of an error.

### Step 6.2 — Read the accelerator's hardware info

```
sudo rknn-smi info -l
```

**How you know it worked:** It lists your RK1820/RK1828 accelerator and its hardware version — confirming the SO-DIMM is detected and enumerated.

### Step 6.3 — Check its current status

```
sudo rknn-smi info -w
```

**How you know it worked:** It reports a live status for the device (idle/working) — proof the driver is communicating with the chip in real time.

### Step 6.4 — (Optional) Read power and set performance mode

Check power draw:

```
sudo rknn-smi info -t power
```

Put the accelerator into its high-performance mode before heavy inference:

```
sudo rknn-smi set -t work_mode -s 2
```

**How you know it worked:** The power command reports a wattage figure; the work-mode command returns without error.

### Step 6.5 — Cross-check the kernel logs

The kernel's boot log records hardware it found. Search it for the rknn driver bringing up the accelerator:

```
sudo dmesg | grep -i rknn
```

**How you know it worked:** You see driver initialization lines mentioning `rknn` and no fatal errors.

> **Note:** On this kit the RK1828 is attached as a coprocessor managed by the Rockchip `rknn3` driver and `rknn-smi`, **not** as a standard PCIe card. So `lspci` may not list it the way a desktop GPU would — `rknn-smi` is the authoritative "is it alive?" check. If you do want to inspect the PCI bus anyway, run `lspci` and look for any Rockchip (vendor `1d87`) device, but treat `rknn-smi` as the source of truth.

> **If this didn't work:** A very common failure is `rknn-smi` reporting **"Failed to initialize rknnsmi"** right after boot, because the `rknn-smi` service starts before the accelerator is fully ready. The fix (per Firefly's documentation) is to add a short startup delay to the service. Edit the service file:
>
> ```
> sudo nano /lib/systemd/system/rknn3.service
> ```
>
> Add this line inside the `[Service]` section:
>
> ```
> ExecStartPre=/bin/sleep 3
> ```
>
> Then reload and reboot:
>
> ```
> sudo systemctl daemon-reload
> ```
>
> ```
> sudo reboot
> ```
>
> After reboot, re-run `sudo rknn-smi info -l`. **How you know it worked:** the accelerator now enumerates without the initialization error.

---

## 7. Installing RKLLM-Toolkit on the host PC

To run an LLM on the accelerator we cannot just download a model and run it — the model first has to be **converted and quantized** into Rockchip's `.rkllm` format. That conversion happens on your **host PC** (it is too heavy for the board), using the **RKLLM-Toolkit** Python package. The board then runs the resulting `.rkllm` file with the lightweight **RKLLM Runtime**.

RKLLM-Toolkit is fussy about its environment: it wants **Python 3.8** specifically. The clean way to give it exactly that without disturbing your system Python is **Conda** (Miniconda). This is the "conda + Python 3.8 dance."

> **Note:** Do everything in this section **on your host PC** (Ubuntu 20.04/22.04 x86-64), *not* on the board.

### Step 7.1 — Install Miniconda (if you don't already have conda)

1. **Step 7.1.1** — Download the Miniconda installer for Linux x86-64:

   ```
   wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
   ```

2. **Step 7.1.2** — Run the installer and accept the defaults:

   ```
   bash Miniconda3-latest-Linux-x86_64.sh
   ```

3. **Step 7.1.3** — Reload your shell so the `conda` command becomes available:

   ```
   source ~/.bashrc
   ```

   **How you know it worked:** Running `conda --version` prints a version number, and your shell prompt now shows `(base)`.

### Step 7.2 — Create the Python 3.8 environment

Create an isolated environment named `RKLLM-Toolkit` pinned to Python 3.8:

```
conda create -n RKLLM-Toolkit python=3.8
```

Activate it:

```
conda activate RKLLM-Toolkit
```

**How you know it worked:** Your prompt now starts with `(RKLLM-Toolkit)`, and `python --version` reports `Python 3.8.x`.

### Step 7.3 — Get the RKLLM source and the toolkit wheel

The toolkit and its examples live in Rockchip's official `rknn-llm` repository.

1. **Step 7.3.1** — Clone the repository:

   ```
   git clone https://github.com/airockchip/rknn-llm.git
   ```

2. **Step 7.3.2** — Enter it:

   ```
   cd rknn-llm
   ```

   **How you know it worked:** `ls` shows folders including `rkllm-toolkit`, `rkllm-runtime`, `examples`, and `doc`.

### Step 7.4 — Install the RKLLM-Toolkit wheel

The installable Python package (a `.whl` file) ships inside the repo under `rkllm-toolkit/`. Filenames change with each release — list the folder to find the exact name:

```
ls rkllm-toolkit/
```

You will see a wheel such as `rkllm_toolkit-1.2.x-cp38-cp38-linux_x86_64.whl` (`cp38` confirms it is the Python 3.8 build, which is why we pinned 3.8). Install it by its real filename:

```
pip3 install ./rkllm-toolkit/rkllm_toolkit-1.2.1-cp38-cp38-linux_x86_64.whl
```

> **Tip:** Replace `1.2.1` with whatever version `ls` actually showed. As of late 2025 the toolkit was at the **v1.2.x** series (the repo's v1.2.3 release added Qwen3-VL and DeepSeek-OCR support). Newer is generally better — match the runtime on the board to the toolkit version you used.

**How you know it worked:** `pip3` reports `Successfully installed rkllm_toolkit-...`, and this import prints the version with no error:

```
python -c "from rkllm.api import RKLLM; print('RKLLM-Toolkit OK')"
```

> **If this didn't work:** A `is not a supported wheel on this platform` error almost always means your environment isn't Python 3.8 (check `python --version`) or you're on the wrong CPU architecture (the wheel is x86-64 only — it will not install on the ARM board). Re-activate the `RKLLM-Toolkit` env and try again.

---

## 8. Running the DeepSeek-R1-Distill-Qwen-1.5B hello-world demo end-to-end

This is the payoff. We will take a real, off-the-shelf large language model — **DeepSeek-R1-Distill-Qwen-1.5B** — convert it to `.rkllm` on the host, copy it to the board, and chat with it running on the NPU. We pick the 1.5B model because it is small enough to convert quickly and runs comfortably on both the RK1820 and RK1828.

The two-machine split is the whole mental model here:

- **Host PC (Python 3.8 conda env):** download weights → convert/quantize → produce a `.rkllm` file.
- **Board (the RK1828):** receive the `.rkllm` file → run the compiled inference demo → answer your questions.

### Step 8.1 — (Host) Download the model weights from Hugging Face

1. **Step 8.1.1** — With your `RKLLM-Toolkit` env active, install the Hugging Face downloader and git-lfs (large files live in LFS):

   ```
   pip3 install huggingface_hub
   ```

   ```
   sudo apt install -y git-lfs
   ```

   ```
   git lfs install
   ```

2. **Step 8.1.2** — Download the model into a local folder:

   ```
   huggingface-cli download deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B --local-dir ./DeepSeek-R1-Distill-Qwen-1.5B
   ```

   **How you know it worked:** The folder `./DeepSeek-R1-Distill-Qwen-1.5B` now contains `config.json`, a tokenizer, and the weight files (`*.safetensors`). This is several gigabytes — give it time.

   > **If this didn't work:** If the download stops with an auth error, run `huggingface-cli login` and paste a token from your Hugging Face account settings, then retry.

### Step 8.2 — (Host) Convert and quantize the model to `.rkllm`

The `examples/rkllm_api_demo` folder contains the canonical export script that loads a Hugging Face model with RKLLM-Toolkit, quantizes it, and writes a `.rkllm` file.

1. **Step 8.2.1** — Go to the export example:

   ```
   cd ~/rknn-llm/examples/rkllm_api_demo
   ```

   **How you know it worked:** `ls` shows a Python export script (commonly `export_rkllm.py`) and a C/C++ source demo (commonly under `src/`).

2. **Step 8.2.2** — Open the export script and set three things near the top: the path to the model folder you downloaded, the **target platform**, and the **quantization type**.

   - **Target platform:** set it to your accelerator — `"RK1820"` or `"RK1828"`.
   - **Quantization type:** use **`"w4a16"`** — 4-bit weights, 16-bit activations. This is the recommended setting for these 20-TOPS accelerators because it shrinks the model enough to fit in the on-chip RAM while keeping good quality. (`w8a8` — 8-bit weights and activations — is the alternative if you need different accuracy/speed trade-offs.)

   In code the call looks roughly like this (your script may differ slightly — match its existing structure):

   ```
   ret = llm.load_huggingface(model="./DeepSeek-R1-Distill-Qwen-1.5B")
   ret = llm.build(do_quantization=True, optimization_level=1, quantized_dtype="w4a16", target_platform="RK1828")
   ret = llm.export_rkllm("./DeepSeek-R1-Distill-Qwen-1.5B.rkllm")
   ```

3. **Step 8.2.3** — Run the conversion:

   ```
   python export_rkllm.py
   ```

   **How you know it worked:** The toolkit prints progress through loading, quantizing, and building, then writes a file named something like `DeepSeek-R1-Distill-Qwen-1.5B.rkllm`. Confirm it exists and is roughly 1–2 GB:

   ```
   ls -lh *.rkllm
   ```

   > **If this didn't work:** Out-of-memory during conversion means your host ran low on RAM — close other apps, or use a machine with more memory (8B+ models can need 32 GB+ to convert). A "platform not supported" error means the `target_platform` string is wrong; it must exactly match `RK1820` or `RK1828`.

### Step 8.3 — (Host → Board) Copy the `.rkllm` file to the board

Use `scp` over the network. Replace `BOARD_IP` with the board's IP address (from `ip addr` in Step 5.6.3):

```
scp ./DeepSeek-R1-Distill-Qwen-1.5B.rkllm firefly@BOARD_IP:/home/firefly/
```

When prompted, the password is `firefly`.

**How you know it worked:** `scp` shows a 100% transfer progress bar and returns to your prompt. On the board, `ls -lh ~/` now lists the `.rkllm` file.

> **Tip:** If `scp` can't connect, make sure the SSH server is running on the board (`sudo systemctl start ssh`) and that both machines are on the same network.

### Step 8.4 — (Board) Build the inference demo

The C/C++ runtime demo has to be compiled (or you can use the prebuilt binary if Firefly's image includes one). Build it **on the board**:

1. **Step 8.4.1** — Get the same repo onto the board so you have the demo sources and the runtime library:

   ```
   git clone https://github.com/airockchip/rknn-llm.git
   ```

   ```
   cd rknn-llm/examples/rkllm_api_demo
   ```

2. **Step 8.4.2** — Run the build script provided in the demo (it compiles against the aarch64 RKLLM runtime library):

   ```
   bash build-linux.sh
   ```

   **How you know it worked:** The script finishes without compiler errors and produces an executable (commonly under a newly created `build/` or `install/` directory, named like `llm_demo`).

   > **If this didn't work:** If the compiler isn't installed, run `sudo apt install -y build-essential cmake` and try again. If it cannot find the runtime `.so`, confirm `librkllmrt.so` exists under `rkllm-runtime/.../aarch64/` in the repo and that the build script points at it.

### Step 8.5 — (Board) Run inference on-device

Run the demo, pointing it at the `.rkllm` file you copied over. The exact arguments are printed if you run the binary with no arguments; a typical invocation is:

```
./llm_demo /home/firefly/DeepSeek-R1-Distill-Qwen-1.5B.rkllm 2048 4096
```

(The two numbers are typical context/token-limit parameters; use the values the demo's usage text specifies.)

**How you know it worked:** The model loads (you'll see it initialize on the NPU), and you get an interactive prompt. Type a question, press Enter, and watch the answer stream back token-by-token.

### Step 8.6 — What output to expect

After loading you should see something like an interactive loop. Try:

```
**User:** Write a haiku about a tiny AI chip.
```

The model will "think" briefly and then stream a response, for example:

```
**Robot:** Silicon whispers,
Five gigabytes hold a mind—
Stick of RAM dreams big.
```

Because DeepSeek-R1 is a *reasoning* model, you may also see a `<think>...</think>` section where it reasons before the final answer. The exact words will differ every run — that's normal; LLM output is not deterministic.

**How you know the whole thing worked:** Text streams out at a brisk pace (on this hardware, tens to over a hundred tokens per second for a 1.5B model). You are now running a large language model entirely on the RK1828's NPU — no cloud, no internet, just the stick-of-RAM-shaped chip on your desk. 🎉

> **If this didn't work:** If the demo crashes with a memory error, the model is too big for the accelerator's on-chip RAM — re-export with `w4a16` quantization (Step 8.2.2) if you didn't already, or pick a smaller model. If it loads but produces garbage text, the toolkit version that converted the model and the runtime version on the board are mismatched — rebuild both from the same `rknn-llm` release.

---

## 9. Where to go next

You ran a 1.5B model. The RK1828's 5 GB of on-chip RAM can do a lot more. From here:

- **Bigger text models — Qwen2.5 / Qwen3 7B:** The 7B-class models are the headline use case for the RK1828. Follow the same convert-and-deploy flow with the larger model. Start from Firefly's AI guide and Rockchip's example zoo: <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/ai_rk182x.html>
- **Vision-language models (VLMs):** Models like Qwen2-VL, InternVL, and MiniCPM-V let the board *see* images and talk about them. The multimodal demo lives at <https://github.com/airockchip/rknn-llm/tree/main/examples/multimodal_model_demo>
- **The RKNN3 model zoo** — ready-made examples (LLMs, VLMs, and classic vision models) specifically targeting the RK1820/RK1828 with the RKNN3 toolkit: <https://github.com/airockchip/rknn3-model-zoo>
- **Rockchip's RKLLM home base** — release notes, supported-model list, and runtime updates: <https://github.com/airockchip/rknn-llm>
- **Firefly's full Rockchip AI documentation:** <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/ai_rockchip.html>

> **Note:** A reality check from independent benchmarks (see the CNX Software writeup in the links): the RK1820/RK1828 shine at **LLM/VLM** workloads (roughly **59–180 tokens/s** depending on model), but for classic computer-vision CNNs like YOLOv5s or ResNet50 they offer **no advantage** over the RK3588's own 6-TOPS NPU. Use the accelerator for language and multimodal models; keep ordinary vision work on the main SoC.

---

## 10. Glossary

- **RK182X** — Firefly/Rockchip's umbrella name for this generation of AI-accelerator modules, covering the **RK1820** (2.5 GB on-chip DRAM, ~3B-parameter LLMs) and **RK1828** (5 GB on-chip DRAM, ~7B-parameter LLMs). Both provide 20 TOPS at INT8.
- **SoM (System on Module)** — A complete tiny computer (CPU, RAM, storage) on one small board that plugs into a larger carrier board. Here, the **Core-3588JD4** is the SoM; it runs Linux.
- **SO-DIMM (in this context)** — The physical connector/form factor — the same edge-card slot used for laptop memory. Rockchip reused this convenient form factor for the RK1820/RK1828 AI accelerator card, so the accelerator *looks like* a RAM stick but is actually a coprocessor.
- **NPU (Neural Processing Unit)** — A chip specialized for the matrix math that neural networks need. It does AI inference far faster and more efficiently than a general-purpose CPU.
- **MaskRom mode** — A recovery boot mode driven by code permanently baked into the chip's silicon. It runs before any bootloader, so it can rescue a board whose bootloader is corrupted. Use it to un-brick.
- **Loader mode** — A flashing mode entered by the board's *existing* bootloader, which pauses to accept firmware-update commands from a host. The normal mode for routine flashing when the bootloader still works.
- **RKLLM** — Rockchip's software stack for running Large Language Models on its NPUs. It has two halves: **RKLLM-Toolkit** (runs on a PC, converts/quantizes Hugging Face models into the `.rkllm` format) and the **RKLLM Runtime** (runs on the board, executes the `.rkllm` file).
- **RKNN vs RKNN2 vs RKNN3** — Successive generations of Rockchip's general neural-network toolkit (for converting and running models like image classifiers/detectors). **RKNN** was the original; **RKNN2** (the `rknn-toolkit2` package) targets RK3588-era chips; **RKNN3** is the newest generation used by the RK1820/RK1828 coprocessors and the `rknn3-model-zoo`. (RKLLM is the LLM-specific sibling of these.)
- **W4A16 vs W8A8** — Quantization schemes (how aggressively the model's numbers are compressed). **W4A16** = 4-bit weights, 16-bit activations — smaller and the recommended default for fitting big models in the accelerator's on-chip RAM. **W8A8** = 8-bit weights and 8-bit activations — a different size/speed/accuracy trade-off. Lower bit-widths use less memory and run faster but can slightly reduce answer quality.

---

## 11. Useful links

Every link below was verified to load at the time of writing.

**Firefly — product & store**
- Product page (store): <https://www.firefly.store/products/rk182x-3d-ram-stacking-development-kit>
- Specification PDF: <https://download.t-firefly.com/Spec/Suite/RK182X-Development-Kit_Specifications_EN.pdf>

**Firefly — documentation wiki (AIO-GS1N2-RK182X)**
- Documentation index: <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/index.html>
- Firmware upgrade / flashing guide: <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/upgrade_rockchip.html>
- AI on RK182X (rknn-smi, accelerator): <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/ai_rk182x.html>
- Rockchip AI tools (RKLLM, RKNN): <https://wiki.t-firefly.com/en/AIO-GS1N2-RK182X/ai_rockchip.html>
- Firmware & tool downloads: <https://en.t-firefly.com/doc/download/369.html>

**Rockchip — official GitHub**
- RKLLM (toolkit + runtime): <https://github.com/airockchip/rknn-llm>
- RKLLM examples (api / multimodal / server demos): <https://github.com/airockchip/rknn-llm/tree/main/examples>
- RKNN3 model zoo (RK1820/RK1828): <https://github.com/airockchip/rknn3-model-zoo>
- RKNN model zoo (RK3588-era vision models): <https://github.com/airockchip/rknn_model_zoo>

**Independent coverage**
- CNX Software — RK1820/RK1828 modules, devkits, and benchmarks: <https://www.cnx-software.com/2025/12/30/rockchip-rk1820-rk1828-so-dimm-and-m-2-llm-vlm-ai-accelerator-modules-devkits-and-benchmarks/>

---

<div align="center">

**TTA 🚀**

*Maintained by Kyle Fox*

</div>
