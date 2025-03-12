string missing
rich header, DanS? https://0xrick.github.io/win-internals/pe3/
encoding overlong https://kevinboone.me/overlong.html

## Simple analysis
In the folder we find two files:
- `Overlong.exe`
- `Message.txt`

I looked at the text-file first. It says:

```html
The secret of this next challenge is cleverly hidden. However, with the right approach, finding the solution will not take an <b>overlong</b> amount of time.
```

There's definitely a hint here that points towards the term *overlong*. I was not aware of its meaning and a quick google search without any further context did not help. So I decided to move on for now.

When I executed the `.exe` file, the following dialog box popped up:

![[Pasted image 20250311082246.png]]

Selecting OK just closed out the program. 

I noticed a hint about *decoding* something.

## Static PE Analysis
I opened up the executable in `PE-Bear`. I learn the following general things:
- It's a very small executable, only 3584 bytes when loaded. 
- The presence of the *Rich Header* suggests it was compiled using Visual Studio or at least using Microsoft's toolchain (check out this [source](https://0xrick.github.io/win-internals/pe3/) for an explanation on this). 
- The *magic number* `NT32` suggested this was a 32-bit program. 
- Only `MessageBoxA()` from `USER32.dll` was (statically) imported
- The raw and virtual size do not suggest *packing*

  ![[Pasted image 20250311083236.png]]

Looking at the strings, I noticed something interesting. First of all, there were very few of them, 8 to be exact. That can make sense since the program is so simple and small. However, though the string in the title of the dialog box (`Output`) was present, the string `I never broke the encoding: ` was not. That means the string is most probably saved in an encrypted or *encoded* form and decoded at runtime. 

This could further confirm my earlier suspicion that some part of the dialog box string has not been fully decoded.

## Disassembly in Ghidra
I decided it was not time to crack open the executable in Ghidra. 
### General function layout
After the automatic analysis in Ghidra, the `main` function was detected. Before I dive into the instruction by instruction details of an executable, I try to get a broad view of the program's workings as much as possible. This can help point me in the right direction and save lots of time. To do this, I first looked at the function call tree, with the `main` function at the root:

![[Pasted image 20250311085120.png]]

Luckily, the function call tree is extremely simple and there's really only two functions. I'll try and get a general sense of what these functions are doing.

```C
// main
undefined4 main(void)

{
  byte local_88 [128];
  uint local_8;
  
  local_8 = FUN_00401160(local_88,&DAT_00402008,0x1c);
  local_88[local_8] = 0;
  MessageBoxA((HWND)0x0,(LPCSTR)local_88,s_Output_0040 3000,0);
  return 0;
}
```

Looking at the main function, we see that the `FUN_00401160()` is called only once, almost immediately after which `MessageBox()` is called. Thus I know that somewhere inside this `FUN_00401160()`, the message string must be decoded. 

```C
//FUN_00401160
uint __cdecl FUN_00401160(byte *param_1,byte *param_2,uint  param_3)

{
  byte bVar1;
  int iVar2;
  uint local_8;
  
  local_8 = 0;
  while( true ) {
    if (param_3 <= local_8) {
      return local_8;
    }
    iVar2 = FUN_00401000(param_1,param_2);
    param_2 = param_2 + iVar2;
    bVar1 = *param_1;
    param_1 = param_1 + 1;
    if (bVar1 == 0) break;
    local_8 = local_8 + 1;
  }
  return local_8;
}
```

I can see that the `FUN_00401160` function contains a loop and that `FUN_00401000()` is called in each iteration. Also, `local_8` seems to be the iterator and is increased by 1 each iteration.

```C
// FUN_00401000
undefined4 __cdecl FUN_00401000(byte *param_1,byte *para m_2)

{
  undefined4 local_c;
  byte local_8;
  
  if ((int)(uint)*param_2 >> 3 == 0x1e) {
    local_8 = (byte)((param_2[2] & 0x3f) << 6) | param_2[3] & 0x 3f;
    local_c = 4;
  }
  else if ((int)(uint)*param_2 >> 4 == 0xe) {
    local_8 = (byte)((param_2[1] & 0x3f) << 6) | param_2[2] & 0x 3f;
    local_c = 3;
  }
  else if ((int)(uint)*param_2 >> 5 == 6) {
    local_8 = (byte)((*param_2 & 0x1f) << 6) | param_2[1] & 0x3f;
    local_c = 2;
  }
  else {
    local_8 = *param_2;
    local_c = 1;
  }
  *param_1 = local_8;
  return local_c;
}
```

Finally, the `FUN_00401000` function seems to contain quite some conditional statements and seems to perform bit-shifts and bit-masks. `param_2` seems to be what the operations are performed on and `param_1` seems to be what the result is stored in. They are both of type `byte *`. All in all this seems to be the actual decoding function, where one character is decoded at a time. 

Looking back, in `FUN_00401160` that same `param_1` `byte` pointer is increased by one each time, which makes sense if it points into the region in memory where the eventually decoded string must live. So this function seems to iterate through that region and call `FUN_00401000` to decode one character at a time.

Now that I have a general sense of what the functions do, I will rename them to reflect this.

![[Pasted image 20250311091017.png]]

I now had a two leads I could further investigate:
- The place where the decoded string and encoded data were stored
- The workings of the decoding function

Again, I do not want or need to know every working detail of this program, since that would take way too much time and would not help me reach my goal. But I think these two things are important enough.
### The decoded string and the encoded data
To find out where the decoded string and the encoded data were being held in memory, I started at `decode_char`.

```C
// decode_char
[..]
if ((int)(uint)*param_2 >> 3 == 0x1e) {
    local_8 = (byte)((param_2[2] & 0x3f) << 6) | param_2[3] & 0x 3f;
    local_c = 4;
}
[..]
*param_1 = local_8;
[..]
```
In each of the conditions, a `local_8` was assigned a value that some combinations of operations evaluated to. Then, `*param_1` was set to that `local_8`. Furthermore, `param_2` seemed to be the subject of each condition and also of each decoding operation. It seemed that `param_1` pointed to the decoded string and `param_2` to the encoded data.

I followed these parameters to their calling function `iterate_tru_chars` to see where they lead.

```C
// iterate_tru_chars
[..]
iVar2 = decode_char(param_1,param_2);
[..]
```

It seems that they are also the first and second parameter of the calling function. I follow them up a level again.

```C
// main
byte local_88 [128];
[..]
local_8 = iterate_tru_chars(local_88,&DAT_00402008,0x1c)
[..]
```

So, I can conclude that the `byte` array called `local_88` declared in `main` is where the decoded string is stored. Makes sense, since `MessageBox` called later in `main` will also need it. Furthermore I can conclude that the encoded data seems to be present over at `DAT_00402008`. I will rename both to reflect this new knowledge:

```C
// main
undefined4 entry(void)

{
  byte decoded_message [128];
  uint local_8;
  
  local_8 = iterate_tru_chars(decoded_message,&encoded_data,0x1c);
  decoded_message[local_8] = 0;
  MessageBoxA((HWND)0x0,(LPCSTR)decoded_message,s_O utput_00403000,0);
  return 0;
}
```

## Overlong encoding
At this point I had some more context for the program and decided to continue deciphering the *overlong* clue. When googling "overlong decoding", I found what I was looking for. 

>Credits for this explanation go to [this source](https://kevinboone.me/overlong.html).

Unicode (UTF-8) characters can be encoded in a variable number of bytes. Simple (from a western viewpoint) characters like the ones in the ASCII set can be represented with one byte. For example the character "A" can be represented as:
- `65` (decimal)
- `0x41` (hex)
- `01 00 00 01` (binary)

However, more non-standard characters that fall outside of the 127 that can fit in one byte, need more than one byte. In this case, the most significant bits in the most significant byte signal how many bytes this character is encoded in. The two most significant bits of the next few bytes signal for continuation of the same character. For example:

`(110)[01001] (10)[10111011]`

The `(110)` in the first byte tells us there are two bytes to this character, the `(10)` in the second byte confirms that this is a continuation byte. `[0100110111011]` is then the actual value being encoded. (Just an example with random values)

The best practice is that a character is represented using the least possible bytes. So "A" is represented with one byte because `U+65` can fit in one byte. However, it's completely possible to encode characters in more bytes than needed. For example "A" can be encoded in the following ways:
- `0xC1, 0x81` = `(110)[00001]  (10)[000001]` = `000 0100 0001` = `x41` = `65`
- `0xE0,0x81,0x81` = `(1110)[0000] (10)[000001] (10)[000001]` = `0000 0000 0100 0001` = `x0041` = `65`

When characters are encoded with more bytes than needed, they're said to be *overlong*. It's best practice to prevent this because it introduces security issues; relatively simple sanitizing of strings could be circumvented.

## The decoding function

Looking at the `decode_char` function I could now see that decoding potentially overlong encoded characters is exactly what it does.

```C
// decode_char
undefined4 __cdecl decode_char(byte *decoded_message,byt e *encoded_data)

{
  undefined4 local_c;
  byte local_8;
  
  if ((int)(uint)*encoded_data >> 3 == 0b00011110) {
    local_8 = (byte)((encoded_data[2] & 0x3f) << 6) | encoded_d ata[3] & 0x3f;
    local_c = 4;
  }
  else if ((int)(uint)*encoded_data >> 4 == 0b00001110) {
    local_8 = (byte)((encoded_data[1] & 0b00111111) << 6) | en coded_data[2] & 0b00111111;
    local_c = 3;
  }
  else if ((int)(uint)*encoded_data >> 5 == 0b00000110) {
    local_8 = (byte)((*encoded_data & 0b00011111) << 6) | enc oded_data[1] & 0b00111111;
    local_c = 2;
  }
  else {
    local_8 = *encoded_data;
    local_c = 1;
  }
  *decoded_message = local_8;
  return local_c;
}

```

The bitshifts and comparisons in the `if`-statements check for a byte with 4, 3, 2 or 1 leading `1` bits respectively. `local_8` and thus `decoded_message` is then set to the 6 least significant bits of both the last and second to last byte and then combined and cast into a byte again (removing the 4 most significant bits). In case of a one-byte encoded character the byte is just copied. Furthermore, the returned `local_c` is set to 4, 3, 2, or 1 based on exactly this. In `iterate_tru_chars` we can see that this `local_c` value is used to increment the pointer into `encoded_data`.

```C
// iterate_tru_chars
[..]
iVar2 = decode_char(decoded_message,encoded_data);
encoded_data = encoded_data + iVar2;
[..]
```

Since only the least significant bits of the two least significant bytes are used and the result is cast into a byte, it seems that the developers know that only ASCII characters will be encountered.

## The solution
We now know that the encoded data is just overlong encoded Unicode, that normal software does not recognize and thus was not shown in our strings list. We also know that the decoding function is meant to specifically decode overlong encoded ASCII characters.

At this point, I thought I might need to find hidden encoded characters in the encoded Unicode by parsing them differently. To get a feel for what that that different method might be, I decided to use `x32dbg` to debug the program.

Here I noticed something interesting though: 

![[Pasted image 20250311201447.png]]
![[Pasted image 20250311201519.png]]

When the string was almost completely decoded, the pointer into the encoded data was nowhere near the end. Notice that there are at least 112 bytes left in this data pocket. This is where my intuition told me that maybe the decoding process just had not finished yet. This would also explain the `:` at the end, like some more should be there.

Taking another look at `iterate_tru_chars` I noted that the loop was defined like:
```C
local_8 = 0;
while( true ) {
	if (param_3 <= local_8) {
	  return local_8;
	}
	[..]
	local_8 = local_8 + 1;
}
return local_8;
```

It seems that this loop terminates when `local_8` is equal to `param_3`. I found that `param_3`was just a hardcoded `0x1c` = `28`. So the decoding loop would only run for 28 iterations. Makes sense since the string `I never broke the encoding: ` is also 28 characters. I patched the value to be `0xFF` in `x32dbg` and let the program run. 

![[Pasted image 20250311202346.png]]

Et voila