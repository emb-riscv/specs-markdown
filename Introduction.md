# Chapter 1: Introduction

This is a draft of the microcontroller architecture description document for RISC-V. Feedback welcome.

## Mission Statement

Define a modern C/C++ friendly microcontroller architecture, that makes writing embedded software easier and more productive. And enjoy the process!

## Goal

Define a set of specifications for RISC-V microcontrolers intended for real-time, low power, bare metal embedded systems.

## Limitations

These specifications intentionaly **do not** include application class devices which use virtual memory and/or have supervisor/hypervisor modes which are intended to run operating systems (like GNU/Linux). For this class of devices, see the "RISC-V Privileged Architecture" specifications.

## Sub-profiles

Since there are many microcontroller configurations, 3 classes were identified

- S (small): single core, 32-bits, low end (intended to support PIC & AVR applications; comparable with Cortex-M0)
- M (medium): single core, 32/64-bits, regular (intended to support common multi-threaded applications; comparable with Cortex-M[34])
- L (large): multi core, 32/64-bits, high end (intended to support hard real-time, high performance applications)


