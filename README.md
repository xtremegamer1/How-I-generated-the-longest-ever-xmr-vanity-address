# How I generated the longest ever xmr vanity address
## Preface
I became interested in vanity addresses when I was browsing around on the dark web. I noticed that most of the onion services I visited contained the name of the service inside of the onion address. For example, duckduckgo provides an onion service at: https://duckduckgogg42xjoc72x3sjasowoarfbgcmvfimaftt6twagswzczad.onion. I figured that these addresses probably had a small field at the beginning for a name followed by relevant cryptographic data. To my suprise, there was no such field, and these addresses are nearly indistingishable from being totally random, being generated as a SHA-1 hash of the private key for the serice. At the same time, I was learning about proof-of-work blockchains, and I realized the process to create one of these onion addresses was no different than the process of generating cryptographic proof-of-work as described in the Bitcoin whitepaper. I have no plans to run an onion service, but I thought it might be cool to have a stylized BTC, LTC, or XMR address. Such addresses, I learned, are called "vanity addresses"

## My Idea
Monero addresses are somewhat unique compared to other crypto addresses, because they consist of 2 keys instead of just 1: these keys are called the **public view key** and the **public spend key** and are derived from the private spend and view key using a cryptographic elliptic curve algorithm. After finding out monero addresses are actually 2 keys, I realized that I could generate significantly longer vanity addresses at the junction between these 2 keys. This would allow me to easily beat any existing length records for vanity addresses, while unfortunately precluding vanity strings at the beginning of the address where they are most visible. Generating 2 halves of the vanity address seperately effectively square roots the chance of finding a valid portion of the address, making searches that would normally take hundreds of years on average possible to do in just minutes.

## Monero Address Weirdness
Monero addresses are implemented in a somewhat obtuse way. The first byte is a "network byte" which specifies whether an address is a part of the Mainnet or testnet and whether or not it is a "subaddress," which is a seperate address corresponding to a different set of private keys derived deterministically from the main private keys and an index. Following this is the 32-byte public spend key, then the 32-byte public view key, and finally a 4-byte checksum. Simple enough, but the conversion to its typical base58 representation is kind of a pain. The address is divided front to back into 8-byte chunks, each of which is treated as a big-endian integer, with the 5 remaining bytes also given the same treatment; Finally, each of these integers is converted to base 58 representation, with the most significant digit first (same as our typical base 10 representation). Each 8-byte section becomes an 11-digit base 58 number and the 5-byte section becomes a 7-digit base 58 number, leading to an overall 95-character address. The most notable quirk this produces is that each 11-digit base58 number is limited to the max 64 bit integer, which can be somewhat limiting; For example, the first character of every 11-character set can never be one of the last 16 characters in the encoding, which are all lowercase letters, as such an 11-digit base58 number would be greater than 2^64. In the creation of vanity addresses, we are concerned with 2 of these 11-digit sections. One of them is made from byte 23 to byte 30 of the spend key, and one of byte 31 of the spend key and bytes 0-6 of the view key. Which, of course, means that when generating the spend key we will need to make sure it is compatible with the view key, otherwise no matter what view key we generate it will not match what we want it to. We can do this by just making a decent approximation of the needed 64-bit integer, and filtering for the most significant byte in that integer along with the first half of the vanity string when generating the spend key.

## Implementation
I tried 2 projects for this, [vanity-xmr-cuda](https://github.com/SChernykh/vanity_xmr_cuda) and [vanity-monero](https://github.com/hinto-janai/monero-vanity). The latter ended up being both faster and easier to modify, despite being CPU only and written in Rust, which I had had no prior experience in. (I wonder if the monero-vanity could be implemented on a GPU to speed it up sugnificantly, or if it relies on unique features of CPUs that GPUs lack?) Anyway, after figuring out how to do it, it was very easy. The string I wanted was 'xtremegamer1' which I decided to divide into 'xtrem' + the right byte in the spend key, and search for 'egamer1' in the view key after substituting the byte into the first 7-byte section.

To find the right byte to use at the end of the spend key:

```bash
$ python
>>> b58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"
>>> b58.index('e') * 58 ** 10 + b58.index('g') * 58 ** 9 + b58.index('a') * 58 ** 8 + b58.index('m') * 58 ** 7 + b58.index('e') * 58 ** 6 
16233758987665858624
>>> hex(16233758987665858624)
'0xe149e5ca0520e040'
```

Now, I modify monero-vanity to find the right spend and view key. Here is the original code from monero-vanity (address.rs, lines 93-102):

```rust
			// Calculate 1st `11` characters of Monero address.
			let mut bytes = [0_u8; 11];
			bytes[0] = NETWORK_BYTE;
			bytes[1..].copy_from_slice(&y.as_bytes()[0..10]);

			let addr = &crate::encode::encode_11(&bytes);

			// Check for regex match.
			// SAFETY:
			// The input is known UTF-8 compatible bytes.
			if regex.is_match(unsafe { &std::str::from_utf8_unchecked(&addr[..]) }) { /*do some stuff*/ }
```

Modified to look for the spend key (changing which 8-byte section is searched and checking for the right final byte):

```rust
			// Calculate 1st `11` characters of Monero address.
			let mut bytes = [0_u8; 8];
			bytes[0] = NETWORK_BYTE;
			bytes[0..].copy_from_slice(&y.as_bytes()[23..31]);

			let addr = &crate::encode::encode_11(&bytes);

			// Check for regex match.
			// SAFETY:
			// The input is known UTF-8 compatible bytes.
			if regex.is_match(unsafe { &std::str::from_utf8_unchecked(&addr[..]) }) && y.as_bytes()[31] == 0xe1 { /*do some stuff*/ }
```

Modified to look for the view key (carrying over the last byte from the spend key and combining it with the first 7 from the view key, and then checking the resulting base58):

```rust
			let mut bytes = [0_u8; 8];
			bytes[0] = 0xe1; // NETWORK_BYTE;
			bytes[1..].copy_from_slice(&y.as_bytes()[0..7]);

			let addr = &crate::encode::encode_11(&bytes);

			// Check for regex match.
			// SAFETY:
			// The input is known UTF-8 compatible bytes.
			if regex.is_match(unsafe { &std::str::from_utf8_unchecked(&addr[..]) }) {/*blah*/}
```

Finally, we take both public keys and the network byte, checksum it, and convert to base58 to get the address, and then we can generate a wallet from the address and private keys! This can be done using monero-wallet-cli:

```bash
$ ./monero-wallet-cli --generate-from-keys your-new-wallet-name-here

```

And you're done! 14- and even 16- digit vanity strings are definitely feasible now with enough computation time, albiet not at the start of the wallet address. Using this method I generated 2 12-character vanity addresses:

47Uo6EFMW5FBEosp951V9SdiiyZLkATdafCu2of**xtremegamer1**wYkchuK1R6sW4mp6bTFQANzf2gbtidM4yoCD6EoJXphP - this one only took 6 minutes to generate
465sL8j2WqWZu7i7B7t9iKYyMbUpAvi9tgNPMDj**MommyMi1kers**96hEYQTimtvNa56cN7tjVqDxUKXMQDbzikXcJ199V8cu - this one took over 6 hours, idk I may have messed something up

Assuming a speed of 1 billion checked addresses per second (close to 20 times what I got), and 50 possible positions for this text, finding a specific 12-digit string would take an average of 637 years to find without using my method. Here is how I figured that:

```
Change of finding something with a 1/n chance after m attempts (c) = (1-1/n) ^ m
Let m := a * n
Therefore c = (1-1/n) ^ an
for large n we can approximate with lim n->inf (1-1/n)^an = e^-a
for c = 0.5 (average case) e^-a = 0.5 therefore a = ln(2)
recall m = a * n therefore m = ln(2) * n
ln(2) * 58 ^ 12 / 50 = 2.01 * 10 ^ 19
2.01 * 10 ^ 19 attempts / 10^9 attempts per sec = 2.01 * 10^10 secs
2.01 * 10^10 secs / 31536000 secs/year = 637 years
```


Keep in mind that using this method will create a non-deterministic wallet, meaning that the view key cannot be derived from the spend key. This means you won't be able to generate a seed phrase for it using ordinary wallet software, and if you are intent on using seedphrases you will have to generate them manually using the seed phrase libraries and you will need 2, one for your private spend and one for your private view key.

Finally, enjoy these long vanity addresses while you can, because Jamtis, a proposed change to Monero that is a part of the Seraphis proposal that will likely be added in the future, will replace these nice addresses with even longer and uglier base32 ones. This method will likely still be possible after the change but there will be fewer characters to work with and all previously generated addresses will no longer work.

If you have any questions or comments feel free to raise an issue about it.
