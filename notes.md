# How manually calculate/correct line checksum in .smc file?

According to Alex Ionescu, in his talk "https://youtu.be/nSqpinjjgmg?t=2053", do the following:
1. Take the line with the 64: in front which you want to calculate the checksum for and add a space in front of each byte:
```
D:00000000:64:B002002051020000C902000057020000570200005702000057020000000000000000000000000000000000005702000057020000000000005702000057020000:B8
+         :64:57020000570200005702000057020000570200005702000057020000570200005702000057020000570200005702000057020000570200005702000057020000:90
```
  > ...
  Considering the second line it becomes:
  >  57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00 57 02 00 00
2. Now find/replace all spaces by +0x and remove the first "+":
  > 0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00+0x57+0x02+0x00+0x00
3. In a terminal prompt enter $((0x57+0x02+...+0x00)) (where ... denotes all the following hex numbers):
  > -bash: 1424: command not found
4. Enter {calculated number} in hex into any searching machine:
  > 590
  > From base 10 to base 16: 1424
5. Mask everything but the LSB:
  > 90, which correlates with the number at the very end of the line. Note that from the LSB *no* additional one's or two's complement is calculated here!
6. Do an Adler32 over all the (in this case 32) line checksums for a particular data record, and paste the result into the corresponding checksum vector:
  > H:20:6300000000000000000000000000000000000000:63
7. Calculate the checksum of the checksum vector:
  > (0x63+...+00) & 0xff = 0x63
8. $$$PROFIT$$$ ;)
