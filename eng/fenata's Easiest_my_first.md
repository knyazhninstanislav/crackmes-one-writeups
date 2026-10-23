# Writeup: Easiest_my_first by fenata (crackmes.one)

**Target:** [Easiest_my_first](https://crackmes.one/crackme/6ac7d71aace07d1c3fbc49e8)  
**Tool:** IDA  
**Difficulty:** Very Easy

---

## 1. Initial Recon

Open the binary in IDA and hit **Ctrl+F12** to bring up the Strings window. Immediately, three suspicious strings stand out:

```asm
.rdata:000000014001F040    db 'Enter ur password:',0
.rdata:000000014001F058    db '123456',0
.rdata:000000014001F060    db 'Congratulations!',0
```

Classic setup. But the real question is: **under what condition does the "Congratulations!" branch get hit?**

Double-click the `"Congratulations!"` string to jump to its cross-reference in the disassembly. IDA lands us inside `sub_140014E30`.

---

## 2. Following the XREF

The data reference for `"Congratulations!"` points to `sub_140014E30+AB`. Scrolling up a bit, the comparison logic becomes obvious:

```asm
lea     rdx, Source                  ; "123456" <--- hardcoded string loaded
mov     rcx, [rbp+190h+Destination]
call    j_strcpy                     ; copy "123456" into Destination buffer
nop
mov     rdx, [rbp+190h+Destination]  ; rdx = "123456"
lea     rcx, [rbp+190h+Str1]         ; rcx = our keyboard input
call    j_strcmp                     ; strcmp("123456", our_input)
test    eax, eax
jnz     short loc_140014F0A          ; if != 0 (mismatch), jump AWAY
```

If the comparison is **not equal** (`eax != 0`), the `jnz` jumps straight past the congratulations block. If it **is equal** (`eax == 0`), execution falls through to:

```asm
lea     rdx, aCongratulation         ; "Congratulations!"
mov     rcx, cs:?cout@std@@3V?$basic_ostream@DU@...
call    sub_1400111FE
```

…and the success message gets printed.

---

## 3. The Password

The logic is a straight `strcmp` against a hardcoded literal. The "password" is:

```
123456
```

That's it. No encryption, no hashing, no obfuscation. Just a plaintext `strcpy` followed by `strcmp`.

---

## 4. Verdict

Enter `123456` at the prompt, get the congratulations. The author literally named it "Easiest_my_first" and hardcoded the most common password on Earth into the `.rdata` section.

**Nice protection :)**