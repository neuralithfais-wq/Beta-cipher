# Beta-cipher
Hobby 128-bit ARX cipher — do NOT use in production
"""
PyARX — Educational 128-bit ARX Block Cipher
===========================================

Pure-Python Feistel ARX cipher with SHA-256 key schedule and interference layers.
Built purely for learning and fun.

🚨 NOT CRYPTOGRAPHICALLY SECURE — DO NOT USE FOR REAL DATA 🚨

Author: Neuralithfais-wq
License: MIT

## Quick Start

```bash
git clone https://github.com/neuralithfais-wq/neuralithfais-wq.git
cd neuralithfais-wq
python -c "
from pyarx import encrypt_block, decrypt_block
key = b'my 16-byte secret'
pt  = b'hello world!!!!'
ct  = encrypt_block(pt, key)
print(decrypt_block(ct, key) == pt)  # True ✅
"""
import hashlib
import struct

MASK = 0xFFFFFFFFFFFFFFFF

# --- Utility Functions ---

def rotl(x, n):
    return ((x << n) & MASK) | (x >> (64 - n))

def derive_round_keys(master_key: bytes, rounds=16):
    round_keys = []
    for r in range(rounds):
        h = hashlib.sha256(master_key + r.to_bytes(4, 'big')).digest()
        k = struct.unpack(">Q", h[:8])[0]
        round_keys.append(k)
    return round_keys

# Fixed odd constants for diffusion
C1 = 0x9E3779B185EBCA87  # golden ratio constant
C2 = 0xC2B2AE3D27D4EB4F  # murmurhash constant

# --- Core Round Function ---

def round_function(L, R, k, alpha=23):
    F = (R * C1) & MASK
    L_new = (L + k) & MASK
    L_new ^= F

    G = (L_new * C2) & MASK
    R_new = rotl(R, alpha)
    R_new = (R_new + G) & MASK

    return L_new, R_new

# --- Interference Layer (every 4 rounds) ---

def interference(L, R, beta=17):
    L = (L + R) & MASK
    R ^= L
    R = rotl(R, beta)
    return L, R

# --- Encryption ---

def encrypt_block(block: bytes, master_key: bytes, rounds=16):
    assert len(block) == 16

    L, R = struct.unpack(">QQ", block)
    round_keys = derive_round_keys(master_key, rounds)

    for i in range(rounds):
        L, R = round_function(L, R, round_keys[i])

        if (i + 1) % 4 == 0:
            L, R = interference(L, R)

    return struct.pack(">QQ", L, R)

# --- Decryption ---

def decrypt_block(block: bytes, master_key: bytes, rounds=16):
    assert len(block) == 16

    L, R = struct.unpack(">QQ", block)
    round_keys = derive_round_keys(master_key, rounds)

    for i in reversed(range(rounds)):

        if (i + 1) % 4 == 0:
            # Reverse interference
            R = rotl(R, 64 - 17)
            R ^= L
            L = (L - R) & MASK

        # Reverse round
        G = (L * C2) & MASK
        R = (R - G) & MASK
        R = rotl(R, 64 - 23)

        F = (R * C1) & MASK
        L ^= F
        L = (L - round_keys[i]) & MASK

    return struct.pack(">QQ", L, R)
