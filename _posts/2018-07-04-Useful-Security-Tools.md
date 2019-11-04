---
title: Useful Security Tools
author: Andy Smith
date: 2018-07-04 23:24:00
---

This is an incomplete list of some useful security tools.

### Forensics
[FTK Imager](https://accessdata.com/product-download)
NTFS imager that lets you browse all data in an NTFS partition. Useful for discovering NTFS-specific data such as security IDs of computers a partition has been connected to.

[Volatility](https://www.volatilityfoundation.org/)
Memory forensics tool

[Binwalk](https://github.com/ReFirmLabs/binwalk)
Looks for headers inside of files to find data structures or files inside of files.

[Foremost](https://tools.kali.org/forensics/foremost)
Searches for files inside of files based on their headers, but you must know what you are looking for.

[ExifTool](https://www.sno.phy.queensu.ca/~phil/exiftool/)
View, edit, or delete exif data in files. Better alternative to the normal `exif` tool.

[zsteg](https://github.com/zed-0xff/zsteg)
Steganography tool for discovering files inside PNG and BMP images.

[FLOSS](https://github.com/fireeye/flare-floss)
FireEye Labs Obfuscated String Solver. Alternative to the normal `strings` command and automatically tries to deobfuscate strings in a binary or file.

### Web Exploitation
[SQLMap](http://sqlmap.org/)
Automatic SQL injection

### Reverse Engineering
[OllyDBG](http://www.ollydbg.de/)
Binary code analyzer and disassembler

[x64dbg](https://x64dbg.com/)
x64/x32 debugger for Windows

[Radare2](http://www.radare.org/r/)
Disassembler and debugger