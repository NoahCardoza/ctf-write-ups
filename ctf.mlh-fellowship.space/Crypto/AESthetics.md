# AESthetics

> Cryptography is beautiful, isn't it? You just have to use it right.
> 
> ---
> [`encrypt.c`](resources/AESthetics/encrypt.c)
> [`secret-flag.png.crypt`](resources/AESthetics/secret-flag.png.crypt)

This one was a doozy! It was only solved by 4 teams. This was the last challenge to solve
for nearly 24 hours before I solved it ~5 minutes before a rival team securing first
place for me and my team.

While I have some basic experience with C, I'd never used the openssl lib with it before.
The first thing I did was figure out how to compile it. After a little legwork I boiled it
down to:

```
gcc -I/usr/local/opt/openssl/include -L/usr/local/opt/openssl/lib -lcrypto -o crypt encrypt.c
```

I ran it on a couple files and it seems to legit... so far. However, there is no way they
could expect us to break AES in such a short amount of time... Plus brute forcing isn't
within the scope of this challenge. There must be something wrong with this code:

Re: prompt:

> You just have to use it right.

Let's find out how you are supposed to use [openssl AES with C](https://www.google.com/search?q=openssl+AES+with+C).
The first result happened to be this [stack overflow](https://stackoverflow.com/questions/9889492/how-to-do-encryption-using-aes-in-openssl)
thread. You can't go wrong with stack overflow? Right?

According to on answer, the C code should look something along the lines of:

```c
static const unsigned char key[] = {
    0x00, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77,
    0x88, 0x99, 0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff,
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
    0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f
};

int main()
{
    unsigned char text[]="hello world!";
    unsigned char enc_out[80];
    unsigned char dec_out[80];

    AES_KEY enc_key, dec_key;

    AES_set_encrypt_key(key, 128, &enc_key);
    AES_encrypt(text, enc_out, &enc_key);      

    AES_set_decrypt_key(key,128,&dec_key);
    AES_decrypt(enc_out, dec_out, &dec_key);

    int i;

    printf("original:\t");
    for(i=0;*(text+i)!=0x00;i++)
        printf("%X ",*(text+i));
    printf("\nencrypted:\t");
    for(i=0;*(enc_out+i)!=0x00;i++)
        printf("%X ",*(enc_out+i));
    printf("\ndecrypted:\t");
    for(i=0;*(dec_out+i)!=0x00;i++)
        printf("%X ",*(dec_out+i));
    printf("\n");

    return 0;
} 
```

Let's compare it to the file we've got...

One thing that stands out: there is no while loop using XOR.

It seems as if in the original file, they are encrypting the counter, and then
looping through the encrypted counter XOR'ing the input in 16 byte chunks.

If we look inside the `main` function, we can see that the first 16 bytes of the
encrypted output happen to be the first iteration of the counter. If we could just figure
out how to increment the counter... Wait! We'd still need the secret key to properly
encrypt the counter before XOR'ing though...

Clearly this is getting messy. Let's try to at least figure out how the counter is being
incremented.

```c
/* helper to increment the 128-bit counter */
void
inc_counter(uint8_t *counter)
{
	uint8_t *pos = counter + 15;
	while (pos > counter) {
		if (++(*pos) != 0x00) {
			return;
		}
		pos--;
	}
	(*pos)++;
}
```

WAIT A SECOND! If `*pos` is of type `uint8_t` (unsigned int), it can never be negative, meaning `0` is
the "smallest" possible number it could ever be. If we step down and take a look at the first if statement
in the while loop, we see that they take the value of `pos` (`*pos`) and increment it by `1` (`++(*pos)`)
before comparing it in the if statement (`(++(*pos) != 0x00)`). Since we know that `0 <= pos`, we know that
`1 <= ++pos` which will never equal `0` (`0x00`).

You might say this function tried to throw us for a LOOP! When in fact, it exist immediately.

This means that the same 16 bytes are used over and over and over, using XOR to encrypt the
whole file! This makes things a lot easier than trying to break AES.

It turns out all the AES code was just to throw us off the trail, the actually encryption
isn't very complicated at all!

Ok, lets review what we know:

1. The file isn't actually encrypted with AES.
2. The file is XOR'ed in 16 byte chunks, repeatedly.
3. The file starts with the unencrypted 16 bytes that are used to XOR the rest of the file.
4. It would seem that the original file was a PNG based off the file name.

First off, if you are familiar with the XOR operator (`^`), you should know it's its own inverse.

E.G.:

+ `8  ^ 16 = 24` 
+ `16 ^ 24 = 8 ` 
+ `8  ^ 24 = 12`

Using this, we can build some what of an equation:

`encrypted_bytes[i] ^ unknowns_bytes[i] = original_bytes[i]`

We have the `encrypted_bytes`,  now all we have to do is figure out 16 of the
`unknowns_bytes`.

Going back to our 4th observation, it seems as if the original file was a PNG. What
do we know about a PNG? Most file types have what's called a [magic number](https://en.wikipedia.org/wiki/List_of_file_signatures).

A PNG's magic number (in hex) is:

```
89 50 4E 47 0D 0A 1A 0A
```

While this is helpful, it's only 8 bytes, which will only allow us to decode every 8 bytes...

PNG's also contain an 8 byte trailer at the end of the file:

```
49 45 4E 44 AE 42 60 82
```

At first glance, you might think we've solved it! Sadly not. Running `wc -c secret-flag.png.crypt`
tells us that the file is `5548` bytes long. Which means there are `12` odd bytes out when writing
in `16` byte chunks.

If we overlap the header and footer bytes, we'll only be able to decode the first 12 bytes:

```
POS : 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16
HEAD: 89 50 4E 47 0D 0A 1A 0A ?? ?? ?? ?? ?? ?? ?? ??
TAIL: ?? ?? ?? ?? 49 45 4E 44 AE 42 60 82 ?? ?? ?? ??
```

<sub>**NOTE**: `POS` refers to the position in the 16 byte counter, not the whole file.</sub>

SO CLOSE! But this isn't enough to open the file successfully.

After opening a couple of PNG's with `xxd` I notice that in addition
to the header, every PNG seems to start with the same `16` bytes:

```
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
```

<sub>**NOTE**: I pulled this straight from `xxd`.</sub>

We finally have enough to write a program to decode the key and decrypt
the file:

```py
from itertools import cycle

buffer = []
header_bytes = ['89', '50', '4e', '47', '0d', '0a', '1a', '0a', '00', '00', '00', '0d', '49', '48', '44', '52']
key_bytes = []


with open('secret-flag.png.crypt', 'rb') as f:
    # skip header
    f.read(16)

    for header_byte, enc_byte in zip(header_bytes, f.read(16)):
        # convert from hex to dec
        header_byte = int(header_byte, 16)
        
        # write the PNG header
        buffer.append(header_byte)

        # decode the key
        key_bytes.append(enc_byte ^ header_byte)
    
    for key_byte, enc_byte in zip(cycle(key_bytes), f.read()):
        buffer.append(key_byte ^ enc_byte)

with open('secret-flag.png', 'wb') as f:
    f.write(bytes(buffer))
```

And sure enough! There's the flag!

## P.S.

When I first looked over the C file, I missed that `inc_counter` didn't actually do anything! To
figure this out, I first compiled the program it with the `-g` flag to generates debug information
like so:

```
gcc -g -I/usr/local/opt/openssl/include -L/usr/local/opt/openssl/lib -lcrypto -o crypt encrypt.c
```

Then I loaded it into `lldb` on my Mac with `lldb crypt` and ran the executable setting stdin/out
since that's where our program get's it's input and writes its output:

```
process launch -i ./secret-flag.png.crypt -o test.crypt --stop-at-entry
```

Now lets set a breakpoint at the `inc_counter` function:

```
breakpoint set -n inc_counter
```

This is what my session in `lldb` looks like:

```
(lldb) c
Process 6552 resuming
your super secure random key: 71a1eb0d9534f777f227f682deab6c90
Process 6552 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100003bc8 crypt`inc_counter(counter="3�_\x7f�]��a\bc4�\x89N") at encrypt.c:27:17
   24  	void
   25  	inc_counter(uint8_t *counter)
   26  	{
-> 27  		uint8_t *pos = counter + 15;
   28  		while (pos > counter) {
   29  			if (++(*pos) != 0x00) {
   30  				return;
Target 0: (crypt) stopped.
(lldb) s
Process 6552 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step in
    frame #0: 0x0000000100003bd6 crypt`inc_counter(counter="3�_\x7f�]��a\bc4�\x89N") at encrypt.c:28:9
   25  	inc_counter(uint8_t *counter)
   26  	{
   27  		uint8_t *pos = counter + 15;
-> 28  		while (pos > counter) {
   29  			if (++(*pos) != 0x00) {
   30  				return;
   31  			}
Target 0: (crypt) stopped.
(lldb) s
Process 6552 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step in
    frame #0: 0x0000000100003be4 crypt`inc_counter(counter="3�_\x7f�]��a\bc4�\x89N") at encrypt.c:29:11
   26  	{
   27  		uint8_t *pos = counter + 15;
   28  		while (pos > counter) {
-> 29  			if (++(*pos) != 0x00) {
   30  				return;
   31  			}
   32  			pos--;
Target 0: (crypt) stopped.
(lldb) s
Process 6552 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step in
    frame #0: 0x0000000100003bfb crypt`inc_counter(counter="3�_\x7f�]��a\bc4�\x89O") at encrypt.c:30:4
   27  		uint8_t *pos = counter + 15;
   28  		while (pos > counter) {
   29  			if (++(*pos) != 0x00) {
-> 30  				return;
   31  			}
   32  			pos--;
   33  		}
Target 0: (crypt) stopped.
```

First, type `c` to **continue** to my newly set breakpoint. Then I use `s` 3 times
to **step** till it hit the return statement.

We can see with the trace above that the loop immediately terminates, which we
could have also concluded with the logic I used initially.
