# How manually calculate/correct line checksum in .smc file?

According to Alex Ionescu, in his 2014 talk "Apple SMC The place to be, definitely (for an implant)" (https://youtu.be/nSqpinjjgmg?t=2053), do the following:
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
0. Get (flow-) arm disassembler (any IDA alternatives out there?)
1. Set instruction set to TBD (Thumb2?)
2. Set endianness to TBD
