# My First Crackme (Very Easy) — райтап

**Крякми:** `My First Crackme (Very Easy)`
**Автор:** TrynaToBe6for6Def (Discord: `imrate`)
**Ссылка:** <https://crackmes.one/crackme/6ac65258cdee6e086a17364d>
**Платформа:** Windows, x86-64, C/C++
**Условие:** найти корректный пароль. Патчить не нужно.
**Ключ (пароль):** `H3llo_Crackme`
**Флаг:** `CMO{H3llo_Crackme}`

---

## 1. Разведка

Открываем файл в **IDA** и идём в **Strings** (Shift+F12). Среди строк находим три интересных:

```text
.rdata:...30A9  aYeaaaaaDiscord  db 'Yeaaaaa!!! Discord : imrate',0Ah,0
.rdata:...30C6  aFlagCmoS        db 'Flag : CMO{%s}',0Ah,0
.rdata:...30D6  aNotToday        db 'Not today...',0Ah,0
```

Это три исхода программы: успех с флагом, успех без флага (промежуточное поздравление) и провал. Переходим по **XREF** на `aFlagCmoS` — попадаем в `sub_140001440`, ту самую «развилку».

## 2. Развилка

В `sub_140001440` находим блок, который решает, куда идти:

```asm
loc_1400014AB:                     ; Str
    lea     rcx, [rbp+10h+Str]     ; указатель на буфер ввода (наш пароль)
    lea     rdx, Control           ; "\r\n"
    call    strcspn                ; обрезаем ввод по первому \r или \n (хвост от fgets)
    mov     [rbp+rax+10h+Str], 0   ; ставим '\0' на место перевода строки
    lea     rcx, [rbp+10h+Str]     ; аргумент — уже очищенная строка
    call    sub_140001520          ; <-- процедура проверки пароля
    cmp     eax, 0
    jz      short loc_1400014F5    ; eax==0 -> "Not today...", иначе -> поздравление с флагом
```

> `strcspn` здесь — не «кладёт ввод», а **отрезает** `\r\n`, оставшиеся после `fgets`. Результат — «чистая» строка без перевода строки, её и проверяет `sub_140001520`.

## 3. Процедура проверки `sub_140001520`

```asm
sub     rsp, 38h
mov     [rsp+38h+Str1], rcx      ; сохраняем указатель на наш ввод
lea     rcx, [rsp+38h+Str2]      ; rcx = адрес буфера для расшифрованной строки
call    sub_7FF7E4771560         ; <-- расшифровка эталона в Str2
mov     rcx, [rsp+38h+Str1]      ; rcx = наш ввод
lea     rdx, [rsp+38h+Str2]      ; rdx = расшифрованный эталон
call    strcmp                   ; сравнение строк
cmp     eax, 0
setz    al                       ; al = 1, если строки равны, иначе 0
and     al, 1
movzx   eax, al                  ; вернуть 0/1 как int
add     rsp, 38h
retn
```

Смысл: функция возвращает `1`, если ввод совпал с расшифрованным эталоном, и `0` иначе. `setz/and/movzx` — просто аккуратное преобразование флага `ZF` в `int` 0/1, ничего хитрого.

## 4. Расшифровка эталона `sub_7FF7E4771560`

```asm
loc_7FF7E4771571:
    cmp     [rsp+10h+var_10], 0Dh       ; счётчик i, длина 13 (0xD)
    jnb     short loc_7FF7E47715A6
    mov     rcx, [rsp+10h+var_10]       ; i
    lea     rax, unk_7FF7E47730E4       ; база зашифрованного куска в .rdata
    movzx   eax, byte ptr [rax+rcx]     ; encrypted[i]
    xor     eax, 2Ah                    ; XOR с 0x2A
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

Алгоритм простой:

- берём **13 байт** по адресу `unk_...30E4` (это RVA от базы модуля, а не фиксированный VA — в IDA он отображается с учётом image base);
- каждый байт **XOR-им с `0x2A`**;
- результат складываем в `Str2`;
- в `Str2[13]` пишем `'\0'` — получаем C-строку.

## 5. Восстановление ключа

Смотрим дамп по `unk_...30E4` (13 байт). В моём случае это были байты:

```
62 19 46 46 45 75 69 58 4B 49 41 47 4F
```

(в IDA они могут отображаться как `b`, `0x19`, `F`, `F`, `E`, `u`, `i`, `X`, `K`, `I`, `A`, `G`, `O` — это одно и то же, просто разные представления).

Пишем расшифровщик. Важно работать **байтами**, а не «сырой строкой с `chr(0x19)`» — так меньше шансов запутаться:

```python
enc = bytes([
    0x62, 0x19, 0x46, 0x46, 0x45, 0x75, 0x69,
    0x58, 0x4B, 0x49, 0x41, 0x47, 0x4F,
])
key = bytes(b ^ 0x2A for b in enc)
print(key.decode())   # H3llo_Crackme
```

Получаем:

```
H3llo_Crackme
```

## 6. Проверка

Вводим `H3llo_Crackme` в окно крякмиса и жмём Enter. Программа печатает:

```text
Yeaaaaa!!! Discord : imrate
Flag : CMO{H3llo_Crackme}
```

- `Yeaaaaa!!! Discord : imrate` — это подпись автора, к решению отношения не имеет.
- `Flag : CMO{%s}` — шаблон, куда подставляется `%s = H3llo_Crackme`.

## 7. Итог

| Что | Значение |
|---|---|
| Пароль | `H3llo_Crackme` |
| Флаг | `CMO{H3llo_Crackme}` |
| Метод | статический анализ в IDA: strings → XREF → расшифровка XOR 0x2A → strcmp |
| Патчинг | не требуется (условие крякмиса — «No patching») |

Отличный «первый» крякмис: минимум кода, прозрачная схема «зашифрованный эталон + `strcmp`», и ровно одна ловушка — `strcspn` с `"\r\n"`, из-за которой ввод с `fgets` надо обрезать перед сравнением. Райтап завершён.