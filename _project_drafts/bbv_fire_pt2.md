---
layout: page
title: Experimenting with the BeagleV-Fire
date: 2024-01-13 00:00:00-0000
description: Customizing the FPGA Logic on a BeagleV-Fire SoC (RiscV + FPGA) SBC.
img: <insert image>
importance: 1
category: Embedded Hardware
giscus_comments: false
giscus_repo: <repo name>
toc:
  sidebar: left
---
## Topics Covered

- My experience getting customizing the FPGA logic on a BeagleV-Fire SoC development board.

## Background



## My Journey

The GPIO and DEFAULT capes share the same constraints file with the GPIO cape having slightly better naming convention.
The device tree for the default cape includes SPI, UART, and PWM components on the FPGA. The GPIO cape does not have these. Mostly impacting the P9 header.
Copying default so I can also test setting pwm, I2C etc with rust.

Steps:
- Created custom-cape folder
- git init for custom-cape
- 


### Boot-up and Go

## Wrapping Up

## Next Steps

## Additional Resources

- [Tio](https://github.com/tio/tio)
- [Top-Level BeagleV-Fire Repository](https://openbeagle.org/beaglev-fire)
- [Gateware Repository](https://openbeagle.org/beaglev-fire/gateware)
