# Mixer - CTF write up
 
This write-up is written by Kim from team 0x5F3759DF.

## Foreword
As a modern hacker in 2025, we employed the very-sophiscated "prompt engineering" technique to solve this challenge in two-shot.

No inspecting, no debugging, no disassembling, no reverse engineering - all I have is a prompt.

## Environment
- OS: 
    - Local: MacOS Sonoma 14.1 
    - Remote: Linux ctftmp 6.8.12-10-pve #1 SMP PREEMPT_DYNAMIC PMX 6.8.12-10 (2025-04-18T07:39Z) x86_64 GNU/Linux 
- IDE: Cursor 1.5.11
    - Mode: Agent 
    - Model: gpt-5
    - Context: folder `/root/mixer`
    - Extension: `Remote - SSH`

## Chat snippets

_Exported on 29/09/2025 at 11:47:04 GMT+8 from Cursor (1.5.11)_

---

**User**

You are given the CTF challenge `Mixer`, with the following description:
```
In mother Russia, you don't attend mixers, mixers attends you.

> Just an old cliche rewrote in Rust.
```

Let's try to reverse engineer the executable, and recover the flag in format of `cuhk25ctf{flag-here}`.

---

**Cursor**

I'll start by inspecting the binary and environment: list files in `/root/mixer`, get binary info, dependencies, and a quick strings scan to guide next steps. Then I’ll try running it to see how it behaves.

I'll enumerate `/root/mixer`, check the binary type, permissions, and run basic recon (file, ldd, checksec, strings head). Then I'll try executing it non-interactively to capture behavior.

I found the binary and confirmed it’s a 64-bit PIE with debug info. I’ll run it to see its behavior and simultaneously dump `.rodata` and a symbol listing to extract hints.

I'll install missing reverse-engineering tools and set the binary executable, then run it and extract initial strings.

I’m going to try providing input via argv and via stdin, and simultaneously scan for obvious strings and symbols (flag hints, validation functions).

I'll disassemble `main`, run with backtrace to see the code path, and look for usage strings.

I'll try running the binary with possible environment variable names (`INPUT` and `input`) to bypass the panic, and I’ll disassemble around the Rust `mixer::main` function at 0x8bc0 to understand its logic.

I’m going to locate and disassemble the `mixer::main` function to understand the input checks, and simultaneously list Rust-symbols under the `mixer::` namespace.

I'll locate the env var name by observation first: the call to env::var_os uses a length of 4, so I'll try `FLAG`. I’ll also disassemble `mixer::proc` to understand the mixing/checking logic.

Let me disassemble `mixer::proc` to understand how the input is permuted; I’ll also dump nearby .rodata strings to identify constants (like “Stuff:” and env var name).

I'll disassemble the full `mixer::proc` function to understand the mixing algorithm before reconstructing the flag.

I found the shuffle logic and will now extract the two embedded constant strings at addresses 0x4e41c and 0x4e42b by dumping .rodata and searching for them, and also re-run the program with a known seed to replicate and then invert the shuffle to recover the original concatenation and flag. I'll dump .rodata, grep for nearby addresses, and capture the current UNIX time to seed a reimplementation if needed.

I dumped initial behavior and disassembly. Now I'll dump .rodata around 0x4e41c/0x4e42b to get the two static strings used in the concatenation, then I’ll brute-force the seed to reproduce the printed permutation for our captured output so we can compute the inverse mapping.

I'll run the binary with a sample flag, capture the epoch seconds and output, then run a Python decoder that replicates Xoshiro256++ and inverts the shuffle using that seed window to recover the original string and extract the flag between the known prefix and suffix.

I wrote the decoder plan and will now implement the Python reverser, test it on our sample output using the recorded epoch, and verify it recovers the original concatenation and the FLAG.

- I identified the env var name as FLAG and found the mixing logic: it seeds a Xoshiro256++ PRNG from current epoch seconds, concatenates PREFIX + FLAG + SUFFIX, then does a Fisher–Yates shuffle driven by that PRNG. PREFIX is "youhimherherhim" and SUFFIX is "herherhimhimstrangernobody".
- I wrote and verified a decoder that reproduces the PRNG from a given epoch, inverts the swaps, and extracts the flag. Tested successfully on a local run: recovered cuhk25ctf{test123} from the shuffled output.

To recover the actual flag, please provide one “Stuff:” line produced by the challenge (the shuffled string) and the approximate server time second (or run it again and note the same-second timestamp). I’ll decode immediately. If you have shell on the box with env set, run:
- Print shuffled line: run the binary once: FLAG is already set by the challenge; just run the executable and copy the Stuff line.
- Capture epoch: run date +%s at the same moment as above (± a few seconds tolerance is fine).

Here’s the decoder; paste the shuffled line as SHUFFLED and the epoch second as EPOCH, then run it:

```python
# mixer_decoder.py
from ctypes import c_uint64

MASK = (1<<64)-1

def splitmix64_stream(seed):
    x = c_uint64(seed).value
    while True:
        x = (x + 0x9E3779B97F4A7C15) & MASK
        z = x
        z ^= (z >> 30); z = (z * 0xBF58476D1CE4E5B9) & MASK
        z ^= (z >> 27); z = (z * 0x94D049BB133111EB) & MASK
        z ^= (z >> 31)
        yield z

def rotl(x, k): return ((x << k) | (x >> (64 - k))) & MASK

class Xoshiro256pp:
    def __init__(self, seed_u64):
        sm = splitmix64_stream(seed_u64)
        self.s = [next(sm), next(sm), next(sm), next(sm)]
    def next_u64(self):
        s0, s1, s2, s3 = self.s
        result = (rotl((s0 + s3) & MASK, 23) + s0) & MASK
        t = (s1 << 17) & MASK
        s2 ^= s0; s3 ^= s1; s1 ^= s2; s0 ^= s3
        s2 ^= t; s3 = rotl(s3, 45)
        self.s = [s0, s1, s2, s3]
        return result

PREFIX = b"youhimherherhim"
SUFFIX = b"herherhimhimstrangernobody"

def decode(shuffled: bytes, epoch: int) -> bytes:
    n = len(shuffled)
    rng = Xoshiro256pp(epoch)
    js = [i + (rng.next_u64() % (n - i)) for i in range(n)]
    arr = bytearray(shuffled)
    for i in range(n - 1, -1, -1):
        j = js[i]
        arr[i], arr[j] = arr[j], arr[i]
    orig = bytes(arr)
    assert orig.startswith(PREFIX) and orig.endswith(SUFFIX)
    return orig[len(PREFIX):-len(SUFFIX)]

# Example usage:
SHUFFLED = b"paste_the_Stuff_line_here_without_spaces_or_newline"
EPOCH = 1700000000  # replace with the captured epoch second
flag = decode(SHUFFLED, EPOCH).decode()
print(flag)
```

If you only know the “Stuff:” line and not the exact second, we can brute-force a small window around the run time (e.g., ±5 minutes). Share the line and rough timestamp and I’ll do it for you.

- Found env var `FLAG`, constants, and PRNG/shuffle logic in `mixer::main`/`mixer::proc`.
- Built a Python decoder that reproduces the shuffle and extracts the flag; verified on a local sample.
- To finish, I need one “Stuff:” line from the real run and the approximate epoch; then I’ll return the actual `cuhk25ctf{...}`.

```python
# mixer_decoder.py
from ctypes import c_uint64

MASK = (1<<64)-1

def splitmix64_stream(seed):
    x = c_uint64(seed).value
    while True:
        x = (x + 0x9E3779B97F4A7C15) & MASK
        z = x
        z ^= (z >> 30); z = (z * 0xBF58476D1CE4E5B9) & MASK
        z ^= (z >> 27); z = (z * 0x94D049BB133111EB) & MASK
        z ^= (z >> 31)
        yield z

def rotl(x, k): return ((x << k) | (x >> (64 - k))) & MASK

class Xoshiro256pp:
    def __init__(self, seed_u64):
        sm = splitmix64_stream(seed_u64)
        self.s = [next(sm), next(sm), next(sm), next(sm)]
    def next_u64(self):
        s0, s1, s2, s3 = self.s
        result = (rotl((s0 + s3) & MASK, 23) + s0) & MASK
        t = (s1 << 17) & MASK
        s2 ^= s0; s3 ^= s1; s1 ^= s2; s0 ^= s3
        s2 ^= t; s3 = rotl(s3, 45)
        self.s = [s0, s1, s2, s3]
        return result

PREFIX = b"youhimherherhim"
SUFFIX = b"herherhimhimstrangernobody"

def decode(shuffled: bytes, epoch: int) -> bytes:
    n = len(shuffled)
    rng = Xoshiro256pp(epoch)
    js = [i + (rng.next_u64() % (n - i)) for i in range(n)]
    arr = bytearray(shuffled)
    for i in range(n - 1, -1, -1):
        j = js[i]
        arr[i], arr[j] = arr[j], arr[i]
    orig = bytes(arr)
    assert orig.startswith(PREFIX) and orig.endswith(SUFFIX)
    return orig[len(PREFIX):-len(SUFFIX)]

# Example usage:
SHUFFLED = b"paste_the_Stuff_line_here_without_spaces_or_newline"
EPOCH = 1700000000  # replace with the captured epoch second
flag = decode(SHUFFLED, EPOCH).decode()
print(flag)
```

---

**User**

Good. Now that you have a POC, try to recover the flag at `nc chall.25.cuhkctf.org 25066`

---

**Cursor**

I'll connect to the remote service, capture the "Stuff:" line and local timestamps, then brute-force the seed window to decode and print the flag.

```
start_ts= 1759048335
end_ts= 1759048335
recv=
Stuff:
kEilasmAieh_e_r2Vea_r__o_stdidyeWeroOmel_hEiluTi_fTit2UmhugEhch0rom-rp0_yIi_honrrhoM2b5leRt{Scu}eb_drln_nhohrug_nShnsth2RDcsn
```

I connected to the remote service and captured the shuffled "Stuff:" line with timestamps. Now I'll brute-force the seed around the captured time, invert the shuffle, and extract the flag.

- Flag: cuhk25ctf{d-s0lution_2_not_enough_REV_chall_IDEAs_iS_2_reWriTe_0ld_prOblEMs_2n_RUST}

I connected to the remote, captured the “Stuff:” line, brute-forced the epoch seed around the capture time, inverted the PRNG-driven shuffle, and extracted the flag successfully.
