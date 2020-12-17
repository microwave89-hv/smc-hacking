# How manually calculate/correct line checksum in .smc file?

According to Alex Ionescu, in his 2014 talk "*Apple SMC The place to be, definitely (for an implant)*" (https://youtu.be/nSqpinjjgmg?t=2053), do the following:
1. Take a line with the `64:` in front which you want to calculate the *primary* checksum ("checksum for a data line") for and precede each byte with a space.
  e.g. this
  ```
  D:00000000:64:B002002051020000C902000057020000570200005702000057020000000000000000000000000000000000005702000057020000000000005702000057020000:B8
  +         :64:57020000570200005702000057020000570200005702000057020000570200005702000057020000570200005702000057020000570200005702000057020000:90
  ...
  +         :64:FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF:C0
  ```
  then becomes (when considering the second line in this example):
  ```
   57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00
  ```
2. In a text editor of your choice automatically find/replace all spaces by +0x and remove the first "+":
  ```
    0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00
  ```
3. In a terminal prompt enter `$((0x57+0x02+...+0x00))` (where ... denotes all the following hex numbers) and obtain the number in decimal format:
  ```
  -bash: 1424: command not found
  ```
4. Enter `{calculated number} in hex` into any search engine to convert it to base16:
  ```
  590
  From base 10 to base 16: 1424
  ```
5. Mask out everything but the LSB to get the *primary* checksum of the line:
  ```
  590 & 0xff = 90
  ```
  which correlates with the number at the very end of the data line. Note that in this case *no* additional one's or two's complement is calculated from the LSB!
  
6. Calculate (same algorithm as before) the checksum of all (in this case 32) data line checksums to get the checksum vector (hash) of the data block
  ```
  (0xB8+0x90+...+0xC0) & 0xff = 0x63
  ```
  
6. ~~Do an Adler32 over all of the (in this case 32) line checksums for a particular data record, and paste the result into the corresponding checksum vector:~~
  ```
  H:20:6300000000000000000000000000000000000000:63
  ```
~~7. Calculate the checksum of the checksum vector, which is the *secondary* checksum, as before:~~
  ```
  (0x63+...+00) & 0xff = 0x63
  ```
~~8. $$$PROFIT$$$ ;)~~
7. Calculate (same algorithm as before) the checksum of a hash
8. Calculate (same algorithm as before) the checksum of all hash checksums to obtain the security sum
9. Calculate (same algorithm as before) the checksum of the security sum (checksum of security sum ("S") of checksums of all hashes ("H") of all checksums of all data lines in a datablock ("D") (lol..))
a. $$$PROFIT$$$ :)

TL;DR: If possible, craft the payload line in a way (alternative instructions...) such that two changes in the same line are opposing each other `(00 + FF = 01 + FE = FF)`. If not possible, craft another data line in a way that upon updating the two data line checksums accordingly the two checksum changes are opposing each other. This is only possible if there are some unused ffff lines. Otherwise, you might compute all-new checksums for the changes and update the existing ones just as good.
BROTIP: Sum up your changes and add them to -(what was there before). This gives you some delta byte. Subtract this delta from the entire affected checksum tree, for example:
```
- 0f d2
+ 00 bf
-------
  00 22 ==> ("change delta")
```
Substract always the same delta from all the parent checksums, while making sure that you update the proper checksum vector.

# How convert .smc blob to binary

In their 2012 presentation "Practical Exploitation of Embedded Systems" (https://dev.inversepath.com/download/public/embedded_systems_exploitation.pdf), the InversePath researchers have issued the following shell command:
```
$ grep -o -E “[A-Z0-9]{64,}” m96.smc | xxd -r -p > m96.bin
```
This simply becomes
```
$ grep -o -E "[A-Z0-9]{64,}" 2012MBPR15.smc | xxd -r -p > 2012MBPR15.bin
```
for a MacBook Pro Retina 15" from 2012.
The binary file should then be padded with (~~0's,~~ ff's?) to have it somewhat match the ROM layout...

# How disassemble SMC code
~~0. Get (flow-) arm disassembler (any IDA alternatives out there?)~~
0. Open disassembler.io
1. Close tutorial and paste bytes to dissassemble
2. Set arch to arm
3. Set address to actual flash address where these bytes are going to be located when the SMC is running (==> not the address when viewing the 2012MBPR15.bin!!!)
4. Set endianness to TBD
5. Set thumb mode to force-thumb (SMC uses Thumb2 instruction set)
Note: As of now, each time that you make a change to the hex bytes you need to set endianness to BIG and then back to LITTLE, otherwise base address is 0 again
Brotip: Don't get confused by some of the handler addresses in the key table. Sometimes those set the bit 0 of the PC which doesn't seem to have an effect, since the instructions have to be aligned on a 2 byte boundary according to the datasheet / reference manual / architecture manual / whateves.

# How prepare updated .smc file
1. Write down the exact changes that you have made to the rom-alike (= properly padded) binary file
2. In the rom image-alike (padded) binary file copy around 12 to 16 bytes directly preceeding your changes while making sure that the byte string is unique
3. In a temporary text editor window remove all spaces or separators between these bytes
4. In the source .smc file find those bytes and temporarily write down the bytes that are going to be replaced
5. Make the change that you have written down previously to the bytes immediately following the search string
6. Sum up the outgoing bytes and and the result with 0xff
7. Sum up the incoming bytes and and the result with 0xff
8. Calculate the change delta, which may be negative as well
9. Add the change delta to each parent checksum/vector in the affected checksum tree (make sure you choose the correct checksum vector out of the many...)

# How load new .smc file
1. Mount a partition that contains an EFI shell
2. Copy to the root:
  * SmcUtil.efi (see https://github.com/microwave89-hv/hw-fw-notes/blob/master/tools.md#old-smcutil)
  * PristineSmcUpdate.smc
  * YourModifiedSmcUpdate.smc
3. (Write down the following instructions to a sheet of paper in real life ;D )
4. Boot into EFI shell
5. Issue following command:
```
SmcUtil -force -LoadApp PristineSmcUpdate.smc -norestart
```
  (This lets you see if the update mechanism will work in the first place)

5. Follow up with:
```
SmcUtil -force -LoadApp YourModifiedSmcUpdate.smc -norestart
```
6. Wait until flashing finishes (It should only take 1 to 2 s and then tell you that it was flashed successfully). While it's flashing the fans may go crazy and come back to normal. If they don't, do the following:
  * Try to reset the SMC (`SmcUtil -reset`)
  * Try to flash the PristineSmcUpdate.smc
This should make the fans go back to normal.
