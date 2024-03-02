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

 
