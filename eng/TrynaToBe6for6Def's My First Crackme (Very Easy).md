# My First Crackme (Very Easy) — write-up

**Crackme:** `My First Crackme (Very Easy)`
**Author:** TrynaToBe6for6Def (Discord: `imrate`)
**Link:** <https://crackmes.one/crackme/6ac65258cdee6e086a17364d>
**Platform:** Windows, x86-64, C/C++
**Goal:** recover the correct password. No patching required.
**Key (password):** `H3llo_Crackme`
**Flag:** `CMO{H3llo_Crackme}`

---

## 1. Recon

Open the binary in **IDA** and go to **Strings** (Shift+F12). Three interesting strings stand out:

```text
.rdata:...30A9  aYeaaaaaDiscord  db 'Yeaaaaa!!! Discord : imrate',0Ah,0
.rdata:...30C6  aFlagCmoS        db 'Flag : CMO{%s}',0Ah,0
.rdata:...30D6  aNotToday        db 'Not today...',0Ah,0
```

These are the three program outcomes: success with a flag, an intermediate success message, and failure. Follow the **XREF** on `aFlagCmoS` — it lands in `sub_140001440`, the branch point.

## 2. The branch

Inside `sub_140001440` we find the code that decides where to go:

```asm
loc_1400014AB:                     ; Str
    lea     rcx, [rbp+10h+Str]     ; pointer to the input buffer (our password)
    lea     rdx, Control           ; "\r\n"
    call    strcspn                ; trim the input at the first \r or \n (leftover from fgets)
    mov     [rbp+rax+10h+Str], 0   ; write '\0' over the newline
    lea     rcx, [rbp+10h+Str]     ; argument — the cleaned string
    call    sub_140001520          ; <-- password check
    cmp     eax, 0
    jz      short loc_1400014F5    ; eax==0 -> "Not today...", otherwise -> congratulate with the flag
```

> `strcspn` here does **not** "store the input" — it **trims** the trailing `\r\n` left by `fgets`. The result is a clean string, which is then handed to `sub_140001520`.

## 3. The check routine `sub_140001520`

```asm
sub     rsp, 38h
mov     [rsp+38h+Str1], rcx      ; save a pointer to our input
lea     rcx, [rsp+38h+Str2]      ; rcx = address of the buffer for the decrypted string
call    sub_7FF7E4771560         ; <-- decrypt the reference into Str2
mov     rcx, [rsp+38h+Str1]      ; rcx = our input
lea     rdx, [rsp+38h+Str2]      ; rdx = decrypted reference
call    strcmp                   ; compare the two strings
cmp     eax, 0
setz    al                       ; al = 1 if equal, else 0
and     al, 1
movzx   eax, al                  ; return 0/1 as int
add     rsp, 38h
retn
```

So the function returns `1` if the input matches the decrypted reference and `0` otherwise. The `setz / and / movzx` sequence is just a tidy way to turn the `ZF` flag into an `int` 0/1 — nothing clever.

## 4. The decryption routine `sub_7FF7E4771560`

```asm
loc_7FF7E4771571:
    cmp     [rsp+10h+var_10], 0Dh       ; loop counter i, length 13 (0xD)
    jnb     short loc_7FF7E47715A6
    mov     rcx, [rsp+10h+var_10]       ; i
    lea     rax, unk_7FF7E47730E4       ; base of the encrypted blob in .rdata
    movzx   eax, byte ptr [rax+rcx]     ; encrypted[i]
    xor     eax, 2Ah                    ; XOR with 0x2A
    mov     dl, al
    mov     rax, [rsp+10h+var_8]        ; Str2
    mov     rcx, [rsp+10h+var_10]       ; i
    mov     [rax+rcx], dl               ; Str2[i] = encrypted[i] ^ 0x2A
    mov     rax, [rsp+10h+var_10]
    add     rax, 1
    mov     [rsp+10h+var_10], rax       ; ++i
    jmp     short loc_7FF7E4771571
loc_7FF7E47715A6:
    mov     rax, [rsp+10h+var_8]
    mov     byte ptr [rax+0Dh], 0       ; Str2[13] = '\0'
    add     rsp, 10h
    retn
```

The algorithm is simple:

- take **13 bytes** at `unk_...30E4` (this is an RVA, not a fixed VA — IDA displays it relative to the image base);
- **XOR each byte with `0x2A`**;
- store the result into `Str2`;
- write `'\0'` at `Str2[13]` — the result is a C string.

## 5. Recovering the key

Dump 13 bytes at `unk_...30E4`. In my case they were:

```
62 19 46 46 45 75 69 58 4B 49 41 47 4F
```

(IDA may show them as `b`, `0x19`, `F`, `F`, `E`, `u`, `i`, `X`, `K`, `I`, `A`, `G`, `O` — same bytes, different display).

Write the decryptor. Work with **bytes**, not "a raw string with `chr(0x19)`" — it's less error-prone:

```python
enc = bytes([
    0x62, 0x19, 0x46, 0x46, 0x45, 0x75, 0x69,
    0x58, 0x4B, 0x49, 0x41, 0x47, 0x4F,
])
key = bytes(b ^ 0x2A for b in enc)
print(key.decode())   # H3llo_Crackme
```

We get:

```
H3llo_Crackme
```

## 6. Verifying

Type `H3llo_Crackme` into the crackme's window and press Enter. The program prints:

```text
Yeaaaaa!!! Discord : imrate
Flag : CMO{H3llo_Crackme}
```

- `Yeaaaaa!!! Discord : imrate` is the author's signature — irrelevant to the solution.
- `Flag : CMO{%s}` is the template, with `%s = H3llo_Crackme` substituted in.

## 7. Result

| Item | Value |
|---|---|
| Password | `H3llo_Crackme` |
| Flag | `CMO{H3llo_Crackme}` |
| Method | static analysis in IDA: strings → XREF → XOR 0x2A decryption → strcmp |
| Patching | not needed (the crackme states "No patching") |

A great "first" crackme: minimal code, a transparent "encrypted reference + `strcmp`" scheme, and exactly one trap — `strcspn` with `"\r\n"`, which forces you to trim the `fgets` newline before comparing. Write-up complete.