# MOS PowerTop

mos-powertop provides a **MOS plugin** that integrates the `powertop`
utility into the MOS ecosystem.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, configuration files (such as `settings.json`).

The plugin allows MOS to make use of powertop for power consumption monitoring and auto-tune management.

### Binary Source

- powertop: [https://github.com/fenrus75/powertop](https://github.com/fenrus75/powertop)

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and package
the plugin and its associated components for MOS.

The build process is fully automated and produces artifacts that can be
installed through the MOS Hub.

---

## Licensing

The contents of this repository (plugin code, build scripts, configuration,
and automation) are licensed under **GPL-3.0**.

`powertop` itself is licensed under its respective upstream license.

---

## Third-Party Software

This repository builds and packages third-party open-source software.
Packaged components remain licensed under their original upstream licenses.

Refer to `THIRD_PARTY.md` for details.
