---
layout: post
title:  "Building a Skylake system"
date:   2019-11-01
---

In 2010 I built a pair systems around
[Westmere](https://en.wikichip.org/wiki/intel/microarchitectures/westmere_(client))
CPUs. At the time those were the fastest consumer processors on the
market and for many years they were the main workhorses for my
computational projects.

But by 2019 they were feeling pretty poky, and I wanted more room than
their maximum of 24GB of RAM. So I built a new system from
scratch:

[![System photo](skylake.jpg)](skylake.jpg)

The key specs are:

- [Intel Core i9-9980XE
CPU](https://en.wikichip.org/wiki/intel/core_i9/i9-9980xe) -- Skylake
microarchitecture, 18 cores / 36 threads
- 128GB of DDR4-3600 RAM
- 2TB solid-state [M.2](https://en.wikipedia.org/wiki/M.2) drive for main storage
- 24TB of scratch disk space in a 3-disk array

Here's the [full parts
list](https://pcpartpicker.com/user/bchaffin76/saved/2FDWbv).

A lot has changed since 2010, and it took a lot of research and
tinkering to get the system running the way I wanted. Below are my
conclusions, for any other newbies who are trying to figure this out.

- Components
  - [Case](#case)
  - [Motherboard](#motherboard-take-one)
  - [Cooling](#cooling)
  - [Memory](#memory)
- [Measuring the system](#measuring-the-system)
- [Overclocking](#overclocking-the-i9-9980xe)
- [AVX settings](#avx-settings)
- [Useful MSRs](#useful-msrs)
- [BIOS settings](#complete-bios-settings)

## Case

#### [Phanteks P600S](http://www.phanteks.com/Eclipse-P600s.html), 5 stars

If you're intending to overclock, airflow is key. This is obvious, but
since every case boasts about its "optimal airflow design" it was hard
to tell from the marketing info and reviews just how much was needed.

I started off with the [Phanteks
P400S](http://www.phanteks.com/Eclipse-P400S-TemperedGlass.html),
which I liked for its sleek design. It's a great case for a smaller
core, but with that solid front panel, it only allows air intake
through small vents at the top and bottom. Even with fans on maximum
my CPU was overheating, and when I pulled the front cover off I could
hear the fans relax and the temperature immediately dropped 10C. So
back it went.

Instead I switched to the [Phanteks
P600S](http://www.phanteks.com/Eclipse-P600s.html), which is
unfortunately significantly larger and more expensive. But its top and
front panels have removable plates, and with those open it allows
great airflow. It also has room for multiple 3.5-inch drives and space
to top-mount a large CPU radiator. So far I've been very happy with
it.

## Motherboard, take one

#### [ASRock X299 Extreme4](https://www.asrock.com/MB/Intel/X299%20Extreme4/index.asp), 1 star

I have an unusual usage for my system -- lots of high-power
computation, and not much else. Systems which support these high-end
CPUs and overclocking are aimed at the gamer market, which means that
more expensive motherboards add features like support for multiple
graphics cards, high-quality sound, and multiple high-speed network
connections -- most of which I don't care about. It also means lots of
marketing fluff full of steel and jagged diagonal graphic design and
hype about "military-grade" this and "optimal" that, which you have to
sort through.

Thinking that I didn't need any of those fancy features, I went with
the entry-level [ASRock X299
Extreme4](https://www.asrock.com/MB/Intel/X299%20Extreme4/index.asp). It
got good reviews and has a decent-looking thermal solution on the
voltage regulator (VRM), with a heat pipe connecting to a second
heatsink around the corner of the board. (Early versions of the X299
boards got a lot of complaints about overheating VRMs.)

But this board was a bust. First of all, its BIOS has a lovely feature
where any time you change any overclocking settings, it automatically
raises the CPU voltage -- which is buried in another menu, which you
might never even look in if you don't intend to over-voltage your CPU
at all -- to something well above spec. High enough that it causes
system resets when running a stress test, and could well damage the
CPU if unnoticed for a long time.

Secondly, despite the good-looking double heatsink, it had persistent
problems with VRM overheating. It doesn't have a VRM temperature
sensor, so you can't determine this directly. But the symptom is that
all cores drop from turbo frequency to the rated frequency of the CPU
(3.0GHz for the i9-9980XE) and stay there for about 10 seconds, and
some experimentation with blowing air directly on the VRM pinpointed
it as the culprit. Initially this was happening every ~30 seconds,
which is a major performance impact. (Also, it's not good for
long-term reliabiltiy to run hard up against thermal limits, as we'll
see below.) After lots of trial and error with mounting extra finned
heatsinks and fans, I settled on two extra fans mounted directly on
top of the VRM heatsinks, along with a noisily high speed for all the
case fans. This helped a lot, but still didn't entirely prevent VRM
throttling under maximum load. (A shout-out to [Noctua
fans](https://noctua.at/en/products/fan), by the way, which are
incredibly quiet.)

Finally, after 6 months the board just died. It crashed a couple times
in some way that also took out the cable modem (despite no direct
connection between them), and then just failed to boot. Good bye and
good riddance.

## Motherboard, take two

#### [Gigabyte X299 Aorus Master](https://www.gigabyte.com/us/Motherboard/X299-AORUS-MASTER-rev-10), 4.5 stars

Having learned some things about what features I needed in a
motherboard, this time around I chose the [Gigabyte X299 Aorus
Master](https://www.gigabyte.com/us/Motherboard/X299-AORUS-MASTER-rev-10). It's
nearly double the price of the ASRock board, but it comes with lots
more features, including integrated heatsinks for the M.2 drives, and
a direct connection from one M.2 slot to the CPU rather than going
through the chipset. And -- crucially -- it has a finned heatsink on
the VRM (instead of an aluminum brick with lots of mass but little
surface area), and a heat pipe to a second heatsink with an integrated
fan.

Amazingly, under the same heavy load, the Gigabyte board draws 100W
less at the wall than the ASRock -- down to 460W from 560W! I don't
know where all that extra power was going; either the ASRock was
driving a higher than necessary voltage to the CPU, or it had
inefficient VRMs, or both. Both the CPU and the VRM run much cooler,
and I seem to be getting the same work done for 20% less power.

The integrated VRM fan does a good job of keeping it cool, but it
makes an unpleasant whine at high speeds. Since I had some extra fans
anyway, I mounted a 40mm Noctua fan in front of the main VRM heatsink,
which lets the integrated fan run slower and quieter.

As of this writing the board is only a week old, but so far I'm happy
with it. My only complaint is some strange fan control behavior when
using the FAN5/6 connectors -- I switched to different connectors and
it hasn't recurred so far.

Out of the box, the Ubuntu 18.04 lm-sensors package was unable to talk
to any of the on-board sensors to read temperature and fan speeds. It
seems that the `it87` driver was maintained by one volunteer, who
finally got fed up with everyone complaining about how he didn't
support their new board. There's still a [clone of his it87 driver
code](https://github.com/a1wong/it87) on GitHub. After some tinkering
with the source I found that I needed to use the
`ignore_resource_conflict` option to ignore an ACPI conflict which was
preventing it from connecting to one of the two sensor chips on the
Aorus Master. After that it seems to work fine.

The sequence to get it up and running was:

```
git clone https://github.com/a1wong/it87
cd it87
make
sudo make install  # copies to /lib/modules/<kernel version>/kernel/drivers/hwmon
sudo modprobe it87 ignore_resource_conflict
# Add it87 ignore_resource_conflict to /etc/modules
```

## Cooling

#### [Corsair H115i Pro](https://www.corsair.com/us/en/Categories/Products/Liquid-Cooling/Hydro-Series%E2%84%A2-H115i-PRO-RGB-280mm-Liquid-CPU-Cooler/p/CW-9060032-WW), 4 stars

I wanted a solid radiator to dissipate the CPU heat. A 140x280mm
radiator has nearly as much surface area as a 120x360mm, and is
cheaper and compatible with more cases. The [Corsair H115i
Pro](https://www.corsair.com/us/en/Categories/Products/Liquid-Cooling/Hydro-Series%E2%84%A2-H115i-PRO-RGB-280mm-Liquid-CPU-Cooler/p/CW-9060032-WW)
has worked fine.

I do have two complaints:
- The pre-applied thermal paste is not nearly sufficient for a large
CPU package like the LGA-2066. When I removed the cooler to replace
the motherboard I found the circle of thermal paste to be dry and
crumbly, and not nearly covering the whole package surface.
- The Corsair adjusts its fan speeds based on the coolant temperature,
but the pump only has three static speeds. To be able to run heavy
loads, I have it set to "performance mode", but that means it's
running at full speed all the time, and when the system is idle and
all the fans are slow, the pump noise is noticeable.

As with most components aimed at the gamer market, there is no
official Linux support whatsoever. Fortunately I found
[OpenCorsairLink](https://github.com/audiohacked/OpenCorsairLink). I
had to use the 'testing' branch (rather than 'master') to get access
to all the H115i's settings. Without being able to set the pump and
fans to higher speeds, I would not be able to run the system at full
load.

## Memory

#### [G.Skill Ripjaws V DDR4-3600](https://www.gskill.com/product/165/184/1536135595/F4-3600C19Q-64GVRBRipjaws-VDDR4-3600MHz-CL19-20-20-40-1.35V64GB-(4x16GB)), 5 stars

My main goal was to max out the 128GB capacity of the board. But I
also care about latency and bandwidth. At the time I bought it,
3600MHz was a clear knee in the curve of frequency vs. cost. The XMP
profile worked perfectly and there has been no problem since.

## Measuring the system

Before you can start overclocking, you have to be able to measure the
frequency, temperature, and performance of the system. That's a big
topic, and I don't claim to be an expert. But here are the tools I
found most useful:

- [turbostat](https://packages.ubuntu.com/bionic/linux-tools-generic) -- best
tool for showing frequency, processor temperature, sleep state
residency (thread, core, and package), and more. You need the
linux-tools-generic package and also linux-tools-\<your-kernel-version\>-generic.
- [OpenCorsairLink](https://github.com/audiohacked/OpenCorsairLink) -- allows
control of a Corsair liquid cooler.
- [it87 driver](https://github.com/a1wong/it87) -- if you're lucky,
updating this driver lets sensors-detect from the lm-sensors package
find your motherboard's on-board temperature and fan speed sensors.
- [i7z](https://packages.ubuntu.com/bionic/admin/i7z) -- alternate
tool for displaying frequency
- [prime95](https://www.mersenne.org/download/) -- best stress test for
AVX2, AVX3, and non-vector code.
- [Intel MKL linpack benchmark](https://software.intel.com/en-us/articles/intel-mkl-benchmarks-suite) --
a good vector benchmark to see whether you are hitting thermal or power
consraints when running vector code.
- [Intel memory latency checker](https://software.intel.com/en-us/articles/intelr-memory-latency-checker) --
measures memory bandwidth, idle latency, and loaded latency. Good for
verifying that your memory's XMP profile is actually doing something.

## Overclocking the i9-9980XE

The stock settings for the i9-9980XE allow two specific "favored cores"
to turbo up to 4.5GHz; the rest can turbo up to 4.4GHz. Those
frequencies are only achievable with one or two cores awake (the rest
have to be in core C6, the lowest power state, where the core is
entirely off). As more cores wake up, the frequency drops, down to
3.8GHz with all 18 cores running. These "turbo ratio limits" keep the
package close to its TDP (thermal design point) of 165W. Wikichip has
the [complete turbo
profile](https://en.wikichip.org/wiki/intel/core_i9/i9-9980xe#Frequencies).

Overclocking can mean different things. I don't actually want to run
my cores faster than they are rated, because I care a lot about
getting the right answer all the time. But I have a good cooling
solution, so I want to run all cores at their maximum frequency
simultaneously.

To do this, I overrode the turbo ratio limits to say that I can turbo
up to 4.5GHz with any number of cores. But by itself, that will run
*all* cores at 4.5GHz, which is faster than some of them can offically
handle. Fortunately the BIOS also supports setting per-core limits, so
I can set the two favored cores to 4.5 and the rest to 4.4.

The default maximum CPU temperature in the Gigabyte BIOS was 82C,
which is pretty low. I raised that to 100C to avoid throttling under
heavy load. There are also PL1 and PL2 power limits, which cap the
power drawn by the CPU. I raised those and the related current limit
to something really large to avoid any power throttling.

And with that, it works, for integer code. Two cores at 4.5 and the
rest at 4.4, steady with no jitter from turboing down and up.

## AVX settings

But of course it's never that simple. Vector code can draw a lot of
power, so the frequency drops more when running AVX code -- more for
AVX3 (512-bit vectors) than for AVX2 (256-bit vectors). There's an
entirely different profile of frequencies vs. number of active cores
when running AVX code. The BIOS exposes this as "AVX offset" and "AVX3
offset", which is how much the CPU ratio should drop by for each type
of code.

The first problem was that when running AVX3 code, the two favored
cores would drop to 700MHz, which is the lowest supported
frequency. All the other cores ran normally. There is a feature known
as "hardware P-states" or "HWP", which in my BIOS defaulted to
"native" mode. This allows the CPU to manage the frequency targets for
the cores internally. In general this works fine, but it doesn't seem
to play well with the per-core ratio limits. Setting it to "legacy"
mode, which lets the operating system request frequencies, fixed
that. As far as I can tell, this is a bug in the internal power
management algorithms of the CPU.

The second problem was that as I ran stress tests which used AVX3 on
more cores, the performance (and power consumption) would suddenly
drop -- even when the overall package was well below any thermal or
power limits. To test the effect I ran a
[linpack](https://software.intel.com/en-us/articles/intel-mkl-benchmarks-suite)
benchmark with 1-36 threads, at different AVX3 offsets which resulted
in frequencies between 3.5GHz and 4.2GHz. Here are the results:

[![Linpack auto load line
calibration](linpack-auto.png)](linpack-auto.png)

Up through 11 threads, the results are as you would expect -- it jumps
up and down because linpack likes the number of threads to be a power
of 2 (or at least even), but higher frequencies give higher
performance. (At 4.2GHz, the system reset when trying to run 11
threads. I guess that is Just Too Fast.)

But starting at 12 threads, the higher frequencies fall off a
cliff. Lower frequencies can handle more threads before dropping off,
but eventually they all suffer greatly reduced performance.

After a lot of experimentation I concluded that this is due to
internal throttling, in response to voltage droop caused by too much
current draw from the cores. Most motherboards have a setting for
"dynamic load line calibration", which adjusts voltage based on
current draw to try to maintain a constant power supply. Changing this
from the default to "extreme" mode -- which tries the hardest to keep
the voltage constant -- helped a lot.

Here's the same benchmark with "extreme" load line calibration:

[![Linpack extreme load line
calibration](linpack-extreme.png)](linpack-extreme.png)

The interesting thing here is that at first, higher frequency gives
better performance. But after 16 threads, the higher frequencies drop
off. It's not as dramatic as before, but 4.1GHz is about 15% slower
than 3.7GHz, so it's significant. 3.7GHz seems to be the sweet spot,
so that's where I left it. (AVX2 is set to run at 3.9GHz, which is the
Intel default. I don't expect to see a lot of AVX2 code.)

## Useful MSRs

While debugging all of this, I did a lot of Googling for details of
the MSRs which report power management state. You can access MSRs
using `sudo rdmsr` and `sudo wrmsr`, after loading the driver with
`sudo modprobe msr`.

Modifying the AVX offsets can be done without rebooting and changing
your BIOS settings. It uses a firmware mailbox which is not publicly
documented, but can be inferred from public snippets of BIOS
code. Here's a perl script which reads or writes the AVX2 and AVX3
offsets:

```perl
#!/usr/bin/perl -w

# Usage:
# avx-offsets [AVX2_offset AVX3_offset]
# With no arguments, prints current offsets

if ($#ARGV == 1) {
    $msr = ($ARGV[0] << 5) | ($ARGV[1] << 10);
    $hexmsr = sprintf("0x8000001b%08x", $msr);
    `sudo wrmsr 0x150 $hexmsr`;
}

`sudo wrmsr 0x150 0x8000001a00000000`;
$msr = `sudo rdmsr 0x150`;
chomp($msr);
$msr = hex($msr);
$avx2 = ($msr >> 5) & 0x1f;
$avx3 = ($msr >> 10) & 0x1f;
print("AVX2 offset = $avx2, AVX3 offset = $avx3\n");
```

Other MSRs I found useful:

- PERF_STATUS - 0x198
  - 15:0 - current ratio
  - upper bits voltage
- PERF_CTL - 0x199
  - 15:0 - software requested ratio
  - 31 - turbo disable
- ENERGY_PERF_BIAS - 0x1b0
  - 3:0 - power policy: 0=perf, 0xf=energy saving
- IA32_PM_ENABLE - 0x770
  - 0 - HWP enable (W1 once) -- Intel SDM section 14.4.2
- IA32_HWP_CAPABILITIES - 0x771
  - 7:0 - highest ratio
  - 15:8 - guaranteed ratio
  - 23:16 - efficient ratio
  - 31:24 - min ratio
  - The highest ratio field differs per core and shows the favored
cores (Intel SDM section 14.4.3).
- IA32_HWP_REQUEST - 0x774
  - 7:0 - min perf
  - 15:8 - max perf (differs per logical processor to show favored core)
  - 23:16 - request for specific ratio, 0 = HW chooses
  - 31:24 - perf bias, overrides ENERGY_PERF_BIAS. 0=perf, 0xff=energy saving, default 0x80
  - set min=max to force a ratio
- IA32_HWP_REQUEST_PKG - 0x772
  - same as HWP_REQUEST but for whole package, if package_control bit
in HWP_REQUEST is set
  - looks like by default this is not used 
- IA32_HWP_STATUS - 0x777
  - change to various ratio limits
- TURBO_RATIO_LIMITS - 0x1ad
- TURBO_RATIO_CORES - 0x1ae
- IA32_THERM_STATUS - 0x19c
  - anything set in low 16 bits indicates some limit is being hit
- IA32_PACKAGE_THERM_STATUS - 0x1b1
  - 0/1 - thermal status/log
  - 2/3 - PROCHOT/log
  - 4/5 - critical temp/log
  - 6/7 - thermal threshold #1/log
  - 8/9 - thermal threshold #2/log
  - 10/11 - power limit/log

## Complete BIOS settings

Mostly for my own records, here is the full list of BIOS settings I
changed from the defaults:

- Override turbo ratio limits to say turbo to 45 with any number of cores
- Use per-core limits to set 16 cores to 44 and two favored cores to 45
- Set AVX/AVX3 offsets to 6/8 to get ratio 39/37
- Disable hardware P-states to work around 700MHz bug
- Set TjMax to 100C
- Set dynamic load line calibration to 'extreme'. Without this, AVX
workloads would throttle around 17-18 threads. With this, can run 36
threads of mprime AVX3 at 37.
- Override L1/L2 power limits and current limits to max
- Enable package C-states -- saves 10W at idle
- Enable XMP profile for DRAM

