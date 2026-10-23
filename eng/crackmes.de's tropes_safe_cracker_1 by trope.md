# tropes_safe_cracker_1 — write-up

**Crackme:** `tropes_safe_cracker_1`
**Author:** trope (crackmes.de, mirrored on crackmes.one)
**Link:** <https://crackmes.one/crackme/5ab77f6533c5d40ad448cb87>

---

## 1. Preparation

1. Open the file in **x32dbg**.
2. Go to **Handles**, hit **Run** a few times until the program window appears.
3. In the Handles window right-click → **Refresh**.

## 2. Setting the hook on a click

When you left-click a button:

- the button itself receives `WM_LBUTTONDOWN` / `WM_LBUTTONUP` (`0x0201` / `0x0202`);
- the button then sends `WM_COMMAND` (`0x0111`) to its parent (the dialog); in its `wParam`:
  - the **low word** is the **button ID**;
  - the **high word** is the notification code (`BN_CLICKED = 0`).

> ⚠️ These are **two different messages** — they are not added together. The notation "`WM_COMMAND + 0x220`" is incorrect.

In the Handles window pick the **"Open"** button → right-click → **Message breakpoint → WM_LBUTTONUP**. This is a convenient way to catch the exact moment that particular button is clicked.

Then press **Open** in the crackme — we land inside `User32.dll`. Get out of it with **Run to user code (Alt+F9)**.

## 3. The fork — dialog procedure

```asm
00401028 | 55                | push ebp
00401029 | 8BEC              | mov ebp,esp
0040102B | 817D 0C 11010000  | cmp dword ptr ss:[ebp+C],111    ; msg == WM_COMMAND (0x111)
00401032 | 75 67             | jne safe.40109B
00401034 | 817D 10 E9030000  | cmp dword ptr ss:[ebp+10],3E9   ; wParam == 1001 (button "1" ID)
0040103B | 75 07             | jne safe.401044
0040103D | 6A 31             | push 31                         ; ASCII '1' (0x31) — arg for the append routine
0040103F | E8 6D000000       | call safe.4010B1                ; append char to buffer 0x403000
00401044 | 817D 10 EA030000  | cmp dword ptr ss:[ebp+10],3EA   ; ID 1002
0040104B | 75 07             | jne safe.401054
0040104D | 6A 32             | push 32                         ; '2'
0040104F | E8 5D000000       | call safe.4010B1
00401054 | 817D 10 EB030000  | cmp dword ptr ss:[ebp+10],3EB   ; ID 1003
0040105B | 75 07             | jne safe.401064
0040105D | 6A 33             | push 33                         ; '3'
0040105F | E8 4D000000       | call safe.4010B1
00401064 | 817D 10 EC030000  | cmp dword ptr ss:[ebp+10],3EC   ; ID 1004
0040106B | 75 07             | jne safe.401074
0040106D | 6A 34             | push 34                         ; '4'
0040106F | E8 3D000000       | call safe.4010B1
00401074 | 817D 10 ED030000  | cmp dword ptr ss:[ebp+10],3ED   ; ID 1005
0040107B | 75 07             | jne safe.401084
0040107D | 6A 35             | push 35                         ; '5'
0040107F | E8 2D000000       | call safe.4010B1
00401084 | 817D 10 EE030000  | cmp dword ptr ss:[ebp+10],3EE   ; ID 1006 ("Open" button)
0040108B | 75 1E             | jne safe.4010AB
0040108D | E8 41000000       | call safe.4010D3                ; validation routine — set a breakpoint here
00401092 | 6A 00             | push 0
00401094 | E8 0D010000       | call <JMP.&ExitProcess>
00401099 | EB 10             | jmp safe.4010AB
0040109B | 837D 0C 10        | cmp dword ptr ss:[ebp+C],10     ; WM_CLOSE
0040109F | 75 0A             | jne safe.4010AB
004010A1 | 6A 00             | push 0
004010A3 | FF75 08           | push dword ptr ss:[ebp+8]       ; hDlg
004010A6 | E8 E9000000       | call <JMP.&EndDialog>
```

**What happens:** clicking any button lands us in the dialog procedure (`DialogProc`). It looks at `msg` (`[ebp+C]`): if it is `WM_COMMAND` (`0x111`), it then examines `wParam` (`[ebp+10]), whose low word holds the ID of the clicked button.

| Situation | What the handler does |
|---|---|
| ID `1001..1005` (digits) | ASCII code (`0x31='1'` … `0x35='5'`) is pushed as an argument, `safe.4010B1` is called, which appends the char to buffer `0x403000` |
| ID `1006` (**Open**) | calls `safe.4010D3` — the validation routine |
| `msg == WM_CLOSE` (`0x10`) | `EndDialog` |

Set a breakpoint on `safe.4010D3` and continue execution.

## 4. Validation routine

```asm
004010D3 | 56                | push esi
004010D4 | 57                | push edi
004010D5 | 33C9              | xor ecx,ecx
004010D7 | 33C0              | xor eax,eax
004010D9 | 33DB              | xor ebx,ebx
004010DB | 68 00304000       | push safe.403000
004010E0 | E8 CD000000       | call <JMP.&lstrlenA>            ; length of the input buffer string
004010E5 | 83F8 06           | cmp eax,6                       ; exactly 6 characters?
004010E8 | 74 03             | je safe.4010ED
004010EA | 5F                | pop edi
004010EB | 5E                | pop esi
004010EC | C3                | ret                             ; not 6 — silently exit
004010ED | BE 00304000       | mov esi,safe.403000             ; buffer of entered digits
004010F2 | BF FF304000       | mov edi,safe.4030FF             ; reference string "1234567890"
004010F7 | 8A06              | mov al,byte ptr ds:[esi]        ; [esi+0] — 1st entered char
004010F9 | 8A5F 04           | mov bl,byte ptr ds:[edi+4]      ; [edi+4] — 5th char of reference ('5')
004010FC | 80C3 02           | add bl,2                        ; '5' + 2
004010FF | 80EB 03           | sub bl,3                        ; net +2-3 = -1 → '4'
00401102 | 38D8              | cmp al,bl                       ; input[0] == '4'?
00401104 | 74 05             | je safe.40110B
00401106 | E8 7A000000       | call <safe.sub_401185>          ; mismatch — set flag at [0x403109]=1
0040110B | 8A46 02           | mov al,byte ptr ds:[esi+2]      ; input[2]
0040110E | 8A5F 04           | mov bl,byte ptr ds:[edi+4]      ; reference[4]
00401111 | 38D8              | cmp al,bl
00401113 | 74 05             | je safe.40111A
00401115 | E8 6B000000       | call <safe.sub_401185>
0040111A | 8A46 01           | mov al,byte ptr ds:[esi+1]      ; input[1]
0040111D | 8A5F 02           | mov bl,byte ptr ds:[edi+2]      ; reference[2]
00401120 | 38D8              | cmp al,bl
00401122 | 74 05             | je safe.401129
00401124 | E8 5C000000       | call <safe.sub_401185>
00401129 | 8A46 04           | mov al,byte ptr ds:[esi+4]      ; input[4]
0040112C | 8A1F              | mov bl,byte ptr ds:[edi]        ; reference[0]
0040112E | 38D8              | cmp al,bl
00401130 | 74 05             | je safe.401137
00401132 | E8 4E000000       | call <safe.sub_401185>
00401137 | 8A46 05           | mov al,byte ptr ds:[esi+5]      ; input[5]
0040113A | 8A5F 02           | mov bl,byte ptr ds:[edi+2]      ; reference[2]
0040113D | 38D8              | cmp al,bl
0040113F | 74 05             | je safe.401146
00401141 | E8 3F000000       | call <safe.sub_401185>
00401146 | 8A46 03           | mov al,byte ptr ds:[esi+3]      ; input[3]
00401149 | 8A5F 04           | mov bl,byte ptr ds:[edi+4]      ; reference[4]
0040114C | 38D8              | cmp al,bl
0040114E | 74 05             | je safe.401155
00401150 | E8 30000000       | call <safe.sub_401185>
00401155 | 803D 09314000 01  | cmp byte ptr ds:[403109],1      ; "validation failed" flag?
0040115C | 74 18             | je safe.401176                  ; yes — skip the congratulation
0040115E | 6A 00             | push 0
00401160 | 68 14324000       | push safe.403214                ; 403214:"Yup"
00401165 | 68 08324000       | push safe.403208                ; 403208:"Good Job."
0040116A | 6A 00             | push 0
0040116C | E8 29000000       | call <JMP.&MessageBoxA>         ; all checks passed — congratulate
```

**Comparison map:**

| # | Input char | Reference char | Note |
|---|---|---|---|
| 1 | `[esi+0]` | `[edi+4] - 1` | `+2`, then `-3` → net `-1` |
| 2 | `[esi+2]` | `[edi+4]` | |
| 3 | `[esi+1]` | `[edi+2]` | |
| 4 | `[esi+4]` | `[edi+0]` | |
| 5 | `[esi+5]` | `[edi+2]` | |
| 6 | `[esi+3]` | `[edi+4]` | |

On mismatch `sub_401185` is called, which writes `1` to `[0x403109]` — the "validation failed" flag. At the end `cmp byte [0x403109], 1; je 0x401176` — if the flag is set, we **skip the congratulation**.

---

## Impression

As an absolute beginner in reversing, this one seemed very well-crafted to me — clean, easy-to-read code and a non-trivial input scheme (buttons instead of plain console input).