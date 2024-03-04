---
title: "Writing a CHIP8 Emulator in C"
date: 2024-03-02T19:28:44+05:30
---

## Storytime
I was always interested in systems programming and one day my friend told me about emulator development and introduced me to CHIP8 and on that day i decided that i am gonna write my own implementation of CHIP8 one day.

## What is CHIP8?
CHIP8 is a interpreted programming language which was first used in COSMAC VIP microcomputer. CHIP8 is not an actual hardware gaming console like Playstation, NES, SNES and Gameboy. CHIP8 programs are basically instructions which can decoded by an interpreter. CHIP8 programs can run on any computer as long as they have that interpreter. In this tutorial we are going to create an interpreter like that.

## Prerequsite
I expect clear understanding of C programming language specially topics like conditionals, loops, pointers, file handling and sturctures.

You will also require basic understanding of binary and hexadecimal number systems and understanding of some library by which you can do things like drawing graphics, keyboard input, etc. we are going to use [SDL2](https://www.libsdl.org/) in this tutorial.

## CHIP8 Specifications

- Memory
    - CHIP8 has access to 4 KB of RAM.

- Keyboard
    - Original CHIP8 has 16 keyboard keys which match to first 16 hexadecimal numbers 0 to F.
    - This Keyboard layout will be used.
    ```markdown
        CHIP8         KEYBOARD

        1 2 3 C       1 2 3 4
        4 5 6 D       Q W E R
        7 8 9 E       A S D F
        A 0 B F       Z X C V
    ```
- Display
    - CHIP8 used 64x32 pixels monochrome(only black and white) display.

- Timers
    - It has an 8 bit delay timer which decrements at a rate of 60Hz(60 times per second) until it reaches zero.
    - It has an 8 bit sound timer which decrements at a rate of 60Hz(60 times per second) unitl it reaches zero.

- 16 8-bit registers
    - CHIP8 has 16 8 bit general purpose Registers V0 to VF.
    - VF is special becuse it is used as a flag register. Many programs set it to either 1 or 0.

- 16 bit program counter
    - program counter usually referred as PC is used to store the address of next instruction to execute.

- 16 bit index register
    - index register or I is a special register used to store memory address.

- 16 level stack
    - CHIP8 has 16 levels of stack which is used to call functions and return from them.

