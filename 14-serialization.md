# Сериализация

## Выбор подхода

### Выбирай формат по требованиям к эволюции схемы и к недоверенному вводу

Формат переживает версии программ и данные от чужих рук, поэтому выбирают его по четырём вопросам: как меняется схема, нужен ли разбор без копирования, должен ли человек читать данные глазами, откуда приходят байты.

| Подход | Примеры | Эволюция схемы | Без копирования | Читается глазами | Недоверенный ввод |
| --- | --- | --- | --- | --- | --- |
| Ручной бинарный | кадр, TLV (практики ниже) | версия кадра, номера полей | да, срезы буфера | нет | свой код: проверки и фаззинг обязательны |
| Текстовый | JSON (cJSON, jsmn), TOML | поля по имени, лишние пропускаются | нет (cJSON), да (jsmn) | да | зрелые разборщики, но нужны лимиты размера и глубины |
| Схема и кодогенерация | protobuf-c, nanopb, flatbuffers | номера полей и правила совместимости схемы | только flatbuffers | нет | генерируемый код проверяет длины, схема фиксирует типы |
| Самоописываемый компактный | MessagePack, CBOR | поля по имени, как в JSON | частично | нет | лимиты размера и глубины ставит вызывающий |

По ситуации выбор выглядит так:

- Обмен между своими процессами и сервисами, данные живут годами: схема с кодогенерацией (protobuf-c, а для микроконтроллеров без `malloc` nanopb), совместимость задают номера полей.
- Внешний программный интерфейс и всё, что читает человек: JSON; файлы настроек: TOML или JSON ([слои конфигурации](15-cli-and-config.md)).
- Внутренний канал двух частей одной программы без лишней зависимости: ручной кадр с TLV, на сотню строк кода.
- Кэш, который пересоздаётся при смене сборки: плоский образ в памяти без разбора.

Что бы ни выбрали, правила на границе одни: каждая длина проверена до выделения, каждое число из данных проверено на диапазон, версию читают первой.

## Бинарные форматы

Практики этого раздела строятся на одном курсоре, а примеры собираются вместе: разбор одного поля из буфера разобран в [главе про системный уровень](13-system-io.md), здесь он обобщён на последовательность полей. Порядок байт везде little-endian; он записывается в формате побайтово, а не через `memcpy` целого.

### Пиши и читай через курсор с липкой ошибкой

Проверка длины перед каждым полем засоряет код, а одна пропущенная проверка становится уязвимостью, поэтому позицию и первую ошибку хранит курсор: после ошибки операции ничего не делают, и результат проверяют один раз в конце. Читатель работает над срезом `str` из [главы про владение](04-memory-ownership.md) и не владеет памятью, писатель пишет в буфер вызывающего.

```c
/* cur.h */
#ifndef CUR_H
#define CUR_H
#include <stddef.h>
#include <stdint.h>

typedef struct { const char *ptr; size_t len; } str;
typedef struct { unsigned char *buf; size_t cap, len; int err; } wr;
typedef struct { const unsigned char *buf; size_t len, pos; int err; } rd;

/* Операции липкие: err хранит первый код (-ENOSPC у писателя, -EBADMSG у читателя), после него
 * писатель не пишет, а читающие функции возвращают 0 и не двигают позицию. Порядок байт: little-endian.
 * r_bytes заимствует память читателя: срез действителен, пока жив исходный буфер. */
wr wr_init(void *buf, size_t cap);
void w_u8(wr *w, uint8_t v);
void w_u16(wr *w, uint16_t v);
void w_u32(wr *w, uint32_t v);
void w_u64(wr *w, uint64_t v);
void w_bytes(wr *w, str s);
rd rd_init(str s);
uint8_t r_u8(rd *r);
uint16_t r_u16(rd *r);
uint32_t r_u32(rd *r);
uint64_t r_u64(rd *r);
str r_bytes(rd *r, size_t n);
#endif
```

```c
/* wr.c */
#include "cur.h"
#include <errno.h>
#include <string.h>

wr wr_init(void *buf, size_t cap) { return (wr){.buf = buf, .cap = cap}; }

static unsigned char *w_room(wr *w, size_t n) {
    if (w->err) return nullptr;
    if (n > w->cap - w->len) { w->err = -ENOSPC; return nullptr; }
    unsigned char *p = w->buf + w->len;
    w->len += n;
    return p;
}

static void w_le(wr *w, uint64_t v, size_t n) {
    unsigned char *p = w_room(w, n);
    for (size_t i = 0; p && i < n; i++) p[i] = (unsigned char)(v >> (8 * i));
}

void w_u8(wr *w, uint8_t v) { w_le(w, v, 1); }
void w_u16(wr *w, uint16_t v) { w_le(w, v, 2); }
void w_u32(wr *w, uint32_t v) { w_le(w, v, 4); }
void w_u64(wr *w, uint64_t v) { w_le(w, v, 8); }
void w_bytes(wr *w, str s) {
    unsigned char *p = w_room(w, s.len);
    if (p && s.len) memcpy(p, s.ptr, s.len);
}
```

```c
/* rd.c */
#include "cur.h"
#include <errno.h>

rd rd_init(str s) { return (rd){.buf = (const unsigned char *)s.ptr, .len = s.len}; }

static const unsigned char *r_take(rd *r, size_t n) {
    if (r->err) return nullptr;
    if (n > r->len - r->pos) { r->err = -EBADMSG; return nullptr; } /* остаток, а не pos + n */
    const unsigned char *p = r->buf + r->pos;
    r->pos += n;
    return p;
}

static uint64_t r_le(rd *r, size_t n) {
    const unsigned char *p = r_take(r, n);
    uint64_t v = 0;
    for (size_t i = 0; p && i < n; i++) v |= (uint64_t)p[i] << (8 * i);
    return v;
}

uint8_t r_u8(rd *r) { return (uint8_t)r_le(r, 1); }
uint16_t r_u16(rd *r) { return (uint16_t)r_le(r, 2); }
uint32_t r_u32(rd *r) { return (uint32_t)r_le(r, 4); }
uint64_t r_u64(rd *r) { return r_le(r, 8); }
str r_bytes(rd *r, size_t n) {
    const unsigned char *p = r_take(r, n);
    return p ? (str){(const char *)p, n} : (str){};
}
```

Запись и чтение идут подряд без проверок, а результат проверяют один раз в конце; усечённый вход оставляет читателя в состоянии ошибки, а не за границей массива.

```c
/* main.c */
#include "cur.h"
#include <errno.h>
#include <stdio.h>

int main(void) {
    unsigned char buf[16];
    wr w = wr_init(buf, sizeof buf);
    w_u8(&w, 7);
    w_u16(&w, 0x0201);
    w_u32(&w, 0x04030201u);
    w_bytes(&w, (str){"hi", 2});
    printf("%d %zu\n", w.err, w.len); /* вывод: 0 9 */
    for (size_t i = 0; i < w.len; i++) printf("%02x ", buf[i]);
    puts(""); /* вывод: 07 01 02 01 02 03 04 68 69 */

    rd r = rd_init((str){(const char *)buf, w.len});
    unsigned a = r_u8(&r);
    unsigned b = r_u16(&r);
    unsigned c = r_u32(&r);
    str s = r_bytes(&r, 2);
    printf("%u %u %u %.*s %d\n", a, b, c, (int)s.len, s.ptr, r.err); /* вывод: 7 513 67305985 hi 0 */

    rd t = rd_init((str){(const char *)buf, 4});
    r_u8(&t);
    r_u32(&t); /* в буфере осталось 3 байта: ошибка, позиция не сдвинулась */
    unsigned x = r_u8(&t); /* после ошибки читающие функции возвращают 0 */
    printf("%d %zu %u\n", t.err == -EBADMSG, t.pos, x); /* вывод: 1 1 0 */
}
```

Порядок байт проверен запуском на big-endian: тот же код собирается кросс-компилятором для s390x и запускается под qemu-user, вывод совпадает побайтно (платформенные детали кросс-сборки: [кросс-компиляция](22-cross-compilation.md)):

```sh
s390x-linux-gnu-gcc -std=c2x -O1 -I. wr.c rd.c main.c -o t
qemu-s390x -L /usr/s390x-linux-gnu ./t
```

#### Подводные камни

- Цикл по счётчику из данных обязан проверять `!r.err` в условии: без этого после первой ошибки цикл крутит нулевые чтения до конца счётчика, а счётчик может быть 2^32.
- Значения после ошибки нулевые, но не «безопасные»: результат используют только после проверки `err`, а частично записанный буфер при `w.err != 0` не отправляют.
- `r_bytes` возвращает срез внутрь буфера читателя. Чтобы данные пережили буфер, их копируют в арену ([память и арены](04-memory-ownership.md)).
- `memcpy` целого в буфер и обратно записывает порядок хоста: на big-endian тот же код даёт другие байты (проверено на s390x), поэтому форматы обмена только побайтово.
- Курсор хранит только первый код: `-ENOSPC` у писателя и `-EBADMSG` у читателя; причины внутри `-EBADMSG` он не различает. «Неполно, ждём данных» от «испорчено» отличает кадр (следующая практика), а не курсор.

### Описывай поля структуры таблицей из имени, типа и смещения

Одна таблица заменяет отдельные функции записи, разбора и печати для каждой структуры: новое поле добавляется строкой, а компилятор следит, чтобы тип в таблице совпадал с типом поля. Дескриптор определён здесь; главы про [аргументы и конфигурацию](15-cli-and-config.md) и [базы данных](17-databases.md) повторяют его копией, чтобы примеры собирались сами; источник один - эта глава. Порождение таблицы из списка полей X-макросом описано в [главе про контейнеры и макросы](11-containers-and-generics.md).

Для сборки примера нужна арена из главы про владение: строки читаются из потока в память арены.

```c
/* arena.h */
#ifndef ARENA_H
#define ARENA_H
#include <stddef.h>

typedef struct { char *beg, *end; } arena;

arena arena_init(void *buf, size_t cap);
void *arena_alloc(arena *a, size_t size, size_t align, size_t count);

#define NEW(a, T, n) ((T *)arena_alloc((a), sizeof(T), alignof(T), (n)))
#endif
```

```c
/* arena.c */
#include "arena.h"
#include <assert.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

arena arena_init(void *buf, size_t cap) {
    char *p = buf;
    return (arena){p, p + cap};
}

void *arena_alloc(arena *a, size_t size, size_t align, size_t count) {
    assert(align && (align & (align - 1)) == 0);
    size_t pad = -(uintptr_t)a->beg & (align - 1);
    size_t room = (size_t)(a->end - a->beg);
    if (pad > room || (count && size > (room - pad) / count)) abort();
    char *p = a->beg + pad;
    a->beg = p + size * count;
    return memset(p, 0, size * count);
}
```

Дескриптор и две функции, которые обходят таблицу. Тип поля попадает в таблицу через `_Generic` по выражению `((T *)0)->f`: оно не вычисляется, нулевой указатель разыменован не будет.

```c
/* desc.h */
#ifndef DESC_H
#define DESC_H
#include "arena.h"
#include "cur.h"
#include <stddef.h>
#include <stdint.h>

typedef enum { F_I32, F_U32, F_U8, F_STR } ftype;
typedef struct { const char *name; ftype type; size_t off; } fdesc;
#define TYPE_OF(T, f) _Generic(((T *)0)->f, int32_t: F_I32, uint32_t: F_U32, uint8_t: F_U8, const char *: F_STR)
#define FIELD(T, f) {#f, TYPE_OF(T, f), offsetof(T, f)}

/* desc_write - пишет поля obj в порядке таблицы t из n строк; строки как u32 длина и байты. Ошибка: w->err. */
void desc_write(wr *w, const fdesc *t, size_t n, const void *obj);
/* desc_read - читает поля в obj; строки копирует в арену a с завершающим NUL. Ошибка: r->err. */
void desc_read(rd *r, const fdesc *t, size_t n, void *obj, arena *a);
#endif
```

```c
/* desc.c */
#include "desc.h"
#include <errno.h>
#include <string.h>
void desc_write(wr *w, const fdesc *t, size_t n, const void *obj) {
    for (size_t i = 0; i < n; i++) {
        const char *p = (const char *)obj + t[i].off;
        switch (t[i].type) {
        case F_I32: case F_U32: { uint32_t v; memcpy(&v, p, sizeof v); w_u32(w, v); break; }
        case F_U8: w_u8(w, (uint8_t)*p); break;
        case F_STR: {
            const char *s;
            memcpy(&s, p, sizeof s);
            size_t len = s ? strlen(s) : 0;
            if (len > UINT32_MAX) { w->err = -EMSGSIZE; return; }
            w_u32(w, (uint32_t)len);
            w_bytes(w, (str){s, len});
            break;
        }
        }
    }
}
void desc_read(rd *r, const fdesc *t, size_t n, void *obj, arena *a) {
    for (size_t i = 0; i < n && !r->err; i++) {
        char *p = (char *)obj + t[i].off;
        switch (t[i].type) {
        case F_I32: case F_U32: { uint32_t v = r_u32(r); memcpy(p, &v, sizeof v); break; }
        case F_U8: *(uint8_t *)p = r_u8(r); break;
        case F_STR: {
            str s = r_bytes(r, r_u32(r)); /* длина проверена до выделения */
            if (r->err) break;
            char *d = NEW(a, char, s.len + 1);
            if (s.len) memcpy(d, s.ptr, s.len);
            const char *cd = d;
            memcpy(p, &cd, sizeof cd);
            break;
        }
        }
    }
}
```

Структура и её таблица лежат в одном заголовке, чтобы запись, разбор и фаззер видели одно определение.

```c
/* item.h */
#ifndef ITEM_H
#define ITEM_H
#include "desc.h"

typedef struct { int32_t id; uint32_t flags; uint8_t level; const char *name; } item;
static const fdesc item_fields[] = {FIELD(item, id), FIELD(item, flags), FIELD(item, level), FIELD(item, name)};
#define ITEM_N (sizeof item_fields / sizeof item_fields[0])
#endif
```

```c
/* main.c */
#include "item.h"
#include <stdio.h>

int main(void) {
    item in = {.id = -5, .flags = 0x10, .level = 3, .name = "alpha"};
    unsigned char buf[64];
    wr w = wr_init(buf, sizeof buf);
    desc_write(&w, item_fields, ITEM_N, &in);
    printf("%d %zu\n", w.err, w.len); /* вывод: 0 18 */

    char mem[256];
    arena a = arena_init(mem, sizeof mem);
    item out = {0};
    rd r = rd_init((str){(const char *)buf, w.len});
    desc_read(&r, item_fields, ITEM_N, &out, &a);
    printf("%d %u %u %s\n", out.id, out.flags, out.level, out.name); /* вывод: -5 16 3 alpha */
    printf("%s %d\n", item_fields[3].name, item_fields[3].type == F_STR); /* вывод: name 1 */
}
```

Поле, для которого в `_Generic` нет ветки, не собирается: `int64_t`, `bool`, `char *` без `const`, массив `char name[16]` (массив в выражении превращается в `char *`) дают ошибку компиляции, и это нужная реакция. Новый тип добавляют в `ftype`, в `TYPE_OF` и в оба `switch`; раскладку `fdesc` не меняют. Проверено на Clang 19, Clang 20 и GCC 14 под флагами книги.

#### Подводные камни

- Порядок строк таблицы равен порядку байт формата: перестановка ломает совместимость. Если формат должен развиваться, таблицу используют вместе с тегами из практики про TLV: номер поля равен индексу строки плюс один, строки только добавляют в конец, удалённое поле остаётся занятой строкой.
- Таблица описывает память, а не формат: смещения и padding зависят от компилятора и платформы, на проводе их нет. Поэтому запись идёт поле за полем через курсор, а чтение через `memcpy` в поле, а не через приведение указателя.
- Строка `const char *` в чтении принадлежит арене: объект с такими полями живёт не дольше арены, а длина каждой строки ограничена остатком входа, поэтому арену берут размером входа плюс по байту на NUL и выравниванию на каждую строку (на нехватку арена отвечает `abort`).
- Таблица не проверяет значения: диапазоны (`level <= 9`) и взаимосвязь полей проверяются отдельно после разбора, в одном месте.

### Оформляй сообщение кадром: magic, версия, длина с потолком, CRC

Поток байт сам не говорит, где сообщение начинается, обрезано ли оно и не испорчено ли, поэтому кадр делает это проверяемым за один вызов. Он расширяет сообщение из главы про [системный уровень](13-system-io.md): там длина ограничена `MSG_MAX`, здесь добавлены признак начала, версия и контрольная сумма.

Раскладка: `magic` (2 байта), `version` (1), `len` (4), `payload` (len байт), `crc32` (4) от всего, что идёт перед ней. Потолок `FRAME_MAX` проверяется до ожидания тела, чтобы собеседник не заставил копить гигабайты.

```c
/* frame.h */
#ifndef FRAME_H
#define FRAME_H
#include "cur.h"

enum { FRAME_MAGIC = 0x4B42, FRAME_VER = 1, FRAME_HDR = 7, FRAME_TAIL = 4, FRAME_MAX = 1 << 20 };
typedef struct { uint8_t ver; str payload; } frame;

uint32_t crc32_ieee(str s);

/* frame_put - записывает кадр с полезной нагрузкой payload в out.
 * Возврат: число байт, -EMSGSIZE (payload > FRAME_MAX) или -ENOSPC. */
int frame_put(unsigned char *out, size_t cap, str payload);

/* frame_get - разбирает один кадр из начала in; f->payload заимствует in.
 * Возврат: число использованных байт; 0 - кадр неполный, нужно больше данных;
 * -EBADMSG - magic, длина или CRC неверны; -EPROTONOSUPPORT - версия новее FRAME_VER. */
int frame_get(str in, frame *f);
#endif
```

```c
/* frame.c */
#include "frame.h"
#include <errno.h>

uint32_t crc32_ieee(str s) {
    uint32_t c = 0xFFFFFFFFu;
    for (size_t i = 0; i < s.len; i++) {
        c ^= (unsigned char)s.ptr[i];
        for (int k = 0; k < 8; k++) c = (c >> 1) ^ (0xEDB88320u & (0u - (c & 1u)));
    }
    return ~c;
}

int frame_put(unsigned char *out, size_t cap, str payload) {
    if (payload.len > FRAME_MAX) return -EMSGSIZE;
    wr w = wr_init(out, cap);
    w_u16(&w, FRAME_MAGIC);
    w_u8(&w, FRAME_VER);
    w_u32(&w, (uint32_t)payload.len);
    w_bytes(&w, payload);
    if (w.err) return w.err;
    w_u32(&w, crc32_ieee((str){(const char *)out, w.len}));
    return w.err ? w.err : (int)w.len;
}

int frame_get(str in, frame *f) {
    if (in.len < FRAME_HDR) return 0;
    rd r = rd_init(in);
    uint16_t magic = r_u16(&r);
    uint8_t ver = r_u8(&r);
    uint32_t len = r_u32(&r);
    if (magic != FRAME_MAGIC || len > FRAME_MAX) return -EBADMSG;
    if (in.len - FRAME_HDR < (size_t)len + FRAME_TAIL) return 0;
    str payload = r_bytes(&r, len);
    uint32_t crc = r_u32(&r);
    if (crc != crc32_ieee((str){in.ptr, FRAME_HDR + len})) return -EBADMSG;
    if (ver > FRAME_VER) return -EPROTONOSUPPORT;
    *f = (frame){ver, payload};
    return (int)(FRAME_HDR + len + FRAME_TAIL);
}
```

Функция разбора отвечает «неполно» нулём и ничего не копит: вызывающий дописывает пришедшие байты в свой буфер, зовёт `frame_get`, пока она возвращает положительное число, и сдвигает остаток к началу (цикл ожидания на `poll` и автомат соединения: [системный уровень](13-system-io.md)). Пример кормит её по одному байту, как худший случай сети.

```c
/* main.c */
#include "frame.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    unsigned char wire[64];
    int n = frame_put(wire, sizeof wire, (str){"ping", 4});
    n += frame_put(wire + n, sizeof wire - (size_t)n, (str){"pong!", 5});
    printf("%d %08x\n", n, crc32_ieee((str){"123456789", 9})); /* вывод: 31 cbf43926 */

    unsigned char acc[64];
    size_t have = 0;
    int frames = 0, waits = 0;
    for (int i = 0; i < n; i++) {
        acc[have++] = wire[i];
        frame f;
        int used = frame_get((str){(const char *)acc, have}, &f);
        if (used == 0) waits++;
        if (used > 0) {
            frames++;
            memmove(acc, acc + used, have - (size_t)used);
            have -= (size_t)used;
        }
    }
    printf("%d %d %zu\n", frames, waits, have); /* вывод: 2 29 0 */

    frame f;
    wire[9] ^= 1; /* испорчен байт нагрузки */
    printf("%d\n", frame_get((str){(const char *)wire, (size_t)n}, &f) == -EBADMSG); /* вывод: 1 */
    wire[9] ^= 1;
    wire[6] = 0x7F; /* заявленная длина выше потолка: отказ сразу, без ожидания тела */
    printf("%d\n", frame_get((str){(const char *)wire, 7}, &f) == -EBADMSG); /* вывод: 1 */
}
```

#### Подводные камни

- CRC находит случайные повреждения, а не подделку: для подлинности нужен код аутентификации или подпись, а проверять его надо сравнением за постоянное время ([безопасность](19-security.md)).
- Ответ `-EBADMSG` означает потерю границ в потоке: соединение закрывают или ищут следующий `magic` с проверкой CRC; выбранную политику записывают в формат, а не оставляют на усмотрение разбирающего.
- CRC считается по заголовку вместе с длиной: повреждённая длина не пройдёт проверку, но потолок всё равно проверяют раньше, иначе читатель ждёт и копит «тело» огромной длины.
- Версию кадра различают отдельным кодом: «версия новее» означает «обнови программу», а не «данные испорчены». Совместимость добавлений внутри нагрузки обеспечивают поля с номерами (следующая практика), а не повышение версии кадра на каждое поле.
- Размер кадра приводят к `int` только после проверки потолка: `FRAME_MAX` меньше `INT_MAX`, поэтому возврат безопасен.

### Кодируй поля тегом и длиной и пропускай неизвестные

Формат, в котором каждое поле несёт номер и способ кодирования, переживает добавление полей: старый читатель пропускает чужое, а новый получает значения по умолчанию для отсутствующих. Целые кодируют переменной длины (LEB128): по 7 бит данных в байте, младшая группа первой, старший бит байта говорит «есть продолжение»; отрицательные числа сначала переводят через zigzag, иначе 64-битное `-1` занимает 10 байт. Две типичные ошибки реализации (знаковый сдвиг в zigzag и сдвиг varint на 64 и больше) показаны в конце следующей практики; UBSan сообщает о них как о левом сдвиге отрицательного значения и о слишком большом показателе сдвига.

```c
/* varint.h */
#ifndef VARINT_H
#define VARINT_H
#include "cur.h"

/* w_varint - пишет v в 1-10 байт. */
void w_varint(wr *w, uint64_t v);
/* r_varint - читает значение; -EBADMSG при обрыве, длине больше 10 байт, значении за пределами 64 бит
 * и избыточной форме (лишний нулевой старший байт: такое значение допускает две записи). */
uint64_t r_varint(rd *r);

/* zigzag: 0, -1, 1, -2 переходят в 0, 1, 2, 3. Сдвиг выполняется над беззнаковым типом, знак берётся сравнением. */
static inline uint64_t zz_enc(int64_t v) { return ((uint64_t)v << 1) ^ (0 - (uint64_t)(v < 0)); }
static inline int64_t zz_dec(uint64_t u) { return (int64_t)(u >> 1) ^ -(int64_t)(u & 1); }
#endif
```

```c
/* varint.c */
#include "varint.h"
#include <errno.h>

void w_varint(wr *w, uint64_t v) {
    while (v >= 0x80) {
        w_u8(w, (uint8_t)(v | 0x80));
        v >>= 7;
    }
    w_u8(w, (uint8_t)v);
}

uint64_t r_varint(rd *r) {
    uint64_t v = 0;
    for (int i = 0;; i++) {
        uint8_t b = r_u8(r);
        if (r->err) return 0;
        if (i == 9 && b > 1) { r->err = -EBADMSG; return 0; } /* десятый байт: остался один бит */
        v |= (uint64_t)(b & 0x7F) << (7 * i); /* i не больше 9: сдвиг меньше 64 */
        if (!(b & 0x80)) {
            if (i > 0 && b == 0) { r->err = -EBADMSG; return 0; } /* избыточная форма */
            return v;
        }
    }
}
```

Крайние случаи в одном запуске: значения, потолок в десять байт, одиннадцатый байт, переполнение, избыточная форма и обрыв.

```c
/* main.c */
#include "varint.h"
#include <stdio.h>

static int try(const unsigned char *p, size_t n, uint64_t *v) {
    rd r = rd_init((str){(const char *)p, n});
    *v = r_varint(&r);
    return r.err == 0;
}

int main(void) {
    const unsigned char a[] = {0xAC, 0x02};
    const unsigned char top[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x01};
    const unsigned char over[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x02};
    const unsigned char eleven[] = {0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x01};
    const unsigned char wide[] = {0x80, 0x00};
    const unsigned char cut[] = {0x80};
    uint64_t v;
    int ok = try(a, sizeof a, &v);
    printf("%d %llu\n", ok, (unsigned long long)v); /* вывод: 1 300 */
    ok = try(top, sizeof top, &v);
    printf("%d %llu\n", ok, (unsigned long long)v); /* вывод: 1 18446744073709551615 */
    printf("%d %d %d %d\n", try(over, sizeof over, &v), try(eleven, sizeof eleven, &v),
           try(wide, sizeof wide, &v), try(cut, sizeof cut, &v)); /* вывод: 0 0 0 0 */
    printf("%llu %llu %llu\n", (unsigned long long)zz_enc(-1), (unsigned long long)zz_enc(1),
           (unsigned long long)zz_enc(INT64_MIN)); /* вывод: 1 2 18446744073709551615 */
    printf("%d\n", zz_dec(zz_enc(INT64_MIN)) == INT64_MIN); /* вывод: 1 */
}
```

Поле сообщения: тег `номер << 3 | тип`, затем значение. Тип 0 - число переменной длины, тип 2 - длина и байты; других типов формат не знает, а значит, пропустить поле с неизвестным типом нельзя и разбор отвечает ошибкой. Неизвестный номер с известным типом читатель пропускает, не зная смысла.

```c
/* tlv.h */
#ifndef TLV_H
#define TLV_H
#include "varint.h"
#include <stdbool.h>

enum { WT_VAR = 0, WT_BYTES = 2 };
typedef struct { unsigned num; int wt; uint64_t v; str s; } field;

void w_field_u(wr *w, unsigned num, uint64_t v);
void w_field_s(wr *w, unsigned num, int64_t v); /* со знаком: через zigzag */
void w_field_b(wr *w, unsigned num, str s);

/* r_field - читает следующее поле в *f. true: поле прочитано; false: данные кончились (r->err == 0)
 * или ошибка (r->err != 0). Номер 0 и неизвестный тип кодирования - ошибка; неизвестный номер - нет. */
bool r_field(rd *r, field *f);
#endif
```

```c
/* tlv.c */
#include "tlv.h"
#include <errno.h>

static void w_tag(wr *w, unsigned num, int wt) { w_varint(w, ((uint64_t)num << 3) | (unsigned)wt); }

void w_field_u(wr *w, unsigned num, uint64_t v) { w_tag(w, num, WT_VAR); w_varint(w, v); }
void w_field_s(wr *w, unsigned num, int64_t v) { w_field_u(w, num, zz_enc(v)); }
void w_field_b(wr *w, unsigned num, str s) { w_tag(w, num, WT_BYTES); w_varint(w, s.len); w_bytes(w, s); }

bool r_field(rd *r, field *f) {
    if (r->err || r->pos == r->len) return false;
    uint64_t tag = r_varint(r);
    if (!r->err && (tag >> 3 == 0 || tag >> 3 > UINT32_MAX)) r->err = -EBADMSG;
    if (r->err) return false;
    f->num = (unsigned)(tag >> 3);
    f->wt = (int)(tag & 7);
    if (f->wt == WT_VAR) {
        f->v = r_varint(r);
    } else if (f->wt == WT_BYTES) {
        uint64_t n = r_varint(r);
        if (!r->err && n > r->len - r->pos) r->err = -EBADMSG;
        if (!r->err) f->s = r_bytes(r, (size_t)n);
    } else {
        r->err = -EBADMSG;
    }
    return !r->err;
}
```

Версионирование следует из этих правил. Версия 2 добавила поле 3 в конец: старый читатель (знает номера до 2) пропускает его и получает то, что понимает, а новый читатель на старом сообщении оставляет имя пустым.

```c
/* user.h */
#ifndef USER_H
#define USER_H
#include "tlv.h"
#include <errno.h>

typedef struct { uint32_t id; int32_t score; str name; } user; /* имя появилось в версии 2 */

static inline int user_put(wr *w, const user *u, bool v2) {
    w_field_u(w, 1, u->id);
    w_field_s(w, 2, u->score);
    if (v2) w_field_b(w, 3, u->name); /* новое поле: новый номер */
    return w->err;
}

/* known - наибольший номер, известный читателю этой версии */
static inline int user_get(str in, user *u, unsigned known) {
    rd r = rd_init(in);
    for (field f; r_field(&r, &f);) {
        if (f.num > known) continue; /* неизвестное поле пропущено по типу кодирования */
        int64_t s = f.wt == WT_VAR ? zz_dec(f.v) : 0;
        if (f.num == 1 && f.wt == WT_VAR && f.v <= UINT32_MAX) u->id = (uint32_t)f.v;
        else if (f.num == 2 && f.wt == WT_VAR && s >= INT32_MIN && s <= INT32_MAX) u->score = (int32_t)s;
        else if (f.num == 3 && f.wt == WT_BYTES) u->name = f.s;
        else return -EBADMSG; /* известный номер с чужим типом или значением вне диапазона */
    }
    return r.err;
}
#endif
```

```c
/* main.c */
#include "user.h"
#include <stdio.h>

int main(void) {
    user u = {7, -3, {"ann", 3}};
    unsigned char b1[32], b2[32];
    wr w1 = wr_init(b1, sizeof b1), w2 = wr_init(b2, sizeof b2);
    user_put(&w1, &u, false);
    user_put(&w2, &u, true);
    printf("%zu %zu\n", w1.len, w2.len); /* вывод: 4 9 */
    user x = {0};
    int e = user_get((str){(const char *)b2, w2.len}, &x, 2); /* читатель версии 1, сообщение версии 2 */
    printf("%d %u %d %zu\n", e, x.id, x.score, x.name.len); /* вывод: 0 7 -3 0 */
    user y = {0};
    e = user_get((str){(const char *)b2, w2.len}, &y, 3); /* читатель версии 2, сообщение версии 2 */
    printf("%d %u %d %zu\n", e, y.id, y.score, y.name.len); /* вывод: 0 7 -3 3 */
    user z = {0};
    e = user_get((str){(const char *)b1, w1.len}, &z, 3); /* читатель версии 2, сообщение версии 1 */
    printf("%d %u %d %zu\n", e, z.id, z.score, z.name.len); /* вывод: 0 7 -3 0 */
}
```

#### Подводные камни

- Номер поля нельзя использовать повторно с другим смыслом или типом: удалённый номер резервируют навсегда, иначе старые данные читаются как новые.
- Отсутствие поля и нулевое значение неотличимы: если различие важно, добавляют поле-признак или отдельный номер, а не «особое значение».
- Порядок полей в потоке не гарантирован и номера могут повториться: читатель не полагается на порядок, а для повтора выбирает правило (побеждает последнее значение или ошибка) и фиксирует его в формате.
- Знаковое 64-битное число без zigzag занимает десять байт. Zigzag диапазон не проверяет: 64-битное значение в поле, которое по смыслу 32-битное, проверяют после разбора, как `user_get`, а не обрезают молча.
- Реализация LEB128 строго ограничена: десять байт, десятый не больше 1, без избыточных форм. Нестрогий разбор принимает несколько записей одного значения, и разные узлы расходятся в подписанных или хэшируемых данных.

### Считай каждое число из данных недоверенным

Любую длину, счётчик, смещение, сдвиг и тип, пришедшие извне, выбрал отправитель, поэтому все эти значения проверяют до того, как они начнут управлять памятью, циклом или сдвигом. Правила проверки короткие и одинаковые для всех форматов этой главы:

- Длина: сравнивают с остатком (`len > end - pos`, где остаток уже неотрицателен), а не суммой `pos + len > end`: сумма без знака оборачивается и пропускает огромную длину. Память выделяют только после проверки; `r_bytes` и `desc_read` из примеров выше так и делают.
- Произведение счётчика на размер элемента считают через `ckd_mul` из `<stdckdint.h>` ([целые типы](03-types-and-ub.md)) и сравнивают с остатком входа до `malloc` или `NEW`.
- Сдвиг на число, пришедшее из данных, не меньше ширины типа даёт UB: `r_varint` ограничивает число байт, а не проверяет сдвиг после факта.
- Глубину рекурсивных форматов ограничивают счётчиком в параметрах рекурсии и отказом выше предела; суммарный размер сообщения и число элементов ограничивают отдельно, иначе небольшой вход раздувается в память и время.
- Перечисление и `bool` читают как целое, проверяют диапазон и только потом присваивают.

Свойство «разбор(запись(x)) = x» ловит ошибки, которые не падают: обвязка фаззера строит значение из байтов, пишет его, читает обратно и сравнивает, а произвольные байты подаёт разборщику напрямую. Это применение дифференциального приёма из [главы про тесты](09-testing.md), где объяснено, как собирать цели и корпус.

```c
/* fuzz.c */
#include "item.h"
#include <stdlib.h>
#include <string.h>

int LLVMFuzzerTestOneInput(const unsigned char *data, size_t size) {
    size_t cap = 2 * size + 128;
    char *mem = malloc(cap);
    unsigned char *out = malloc(size + 16);
    if (!mem || !out) abort();
    arena a = arena_init(mem, cap);
    item x = {0}, y = {0};
    rd r = rd_init((str){(const char *)data, size});
    desc_read(&r, item_fields, ITEM_N, &x, &a); /* произвольные байты: разбор не должен падать */
    if (!r.err) {
        wr w = wr_init(out, size + 16);
        desc_write(&w, item_fields, ITEM_N, &x);
        rd r2 = rd_init((str){(const char *)out, w.len});
        desc_read(&r2, item_fields, ITEM_N, &y, &a);
        if (w.err || r2.err || x.id != y.id || x.flags != y.flags || x.level != y.level || strcmp(x.name, y.name))
            abort(); /* нарушено свойство: чтение записанного отличается от записанного */
    }
    free(out);
    free(mem);
    return 0;
}
```

Запуск ограничивают числом прогонов, чтобы цель годилась для CI; корпус начинают с одного корректного сообщения. Проверено запуском: если в копии `desc.c` сломать запись поля `level`, фаззер находит нарушение свойства в пределах тех же 200000 прогонов.

```sh
clang-19 -std=c23 -O1 -g -fsanitize=fuzzer,address,undefined desc.c wr.c rd.c arena.c fuzz.c -o fuzz
./fuzz corpus -runs=200000 -max_len=256
```

Типичные ошибки, которые ловят санитайзеры. Наивный zigzag сдвигает влево отрицательное число (UBSan: left shift of negative value); правильная форма сдвигает беззнаковое значение.

```c
/* Ошибка: UB - наивный zigzag сдвигает влево отрицательное число */
#include <stdint.h>
#include <stdio.h>

int main(void) {
    volatile int64_t v = -1;
    uint64_t z = (uint64_t)((v << 1) ^ (v >> 63));
    printf("%llu\n", (unsigned long long)z);
}
```

Декодер varint без границы числа байт сдвигает на 70 и больше; UB возникает уже на одиннадцатом байте.

```c
/* Ошибка: UB - сдвиг varint на 64 и больше при длинной цепочке продолжений */
#include <stdint.h>
#include <stdio.h>

int main(void) {
    unsigned char in[11] = {0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x01};
    uint64_t v = 0;
    int shift = 0;
    size_t i = 0;
    unsigned char b;
    do {
        b = in[i++];
        v |= (uint64_t)(b & 0x7F) << shift;
        shift += 7;
    } while (b & 0x80);
    printf("%llu\n", (unsigned long long)v);
}
```

Сумма `pos + len` без знака оборачивается (это определено и ничего не сообщает), проверка пропускает длину из данных, а UB возникает при чтении за границей блока: его ловит ASan.

```c
/* Ошибка: UB - pos + len оборачивается, проверка длины из данных пропускает чтение за буфером */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    unsigned char *buf = calloc(8, 1);
    if (!buf) abort();
    volatile size_t len = (size_t)-2; /* длина из данных */
    size_t pos = 4, sum = 0;
    if (pos + len <= 8)
        for (size_t i = 0; i < len; i++) sum += buf[pos + i];
    printf("%zu\n", sum);
    free(buf);
}
```

Байт из данных, скопированный `memcpy` в `bool`, при значении больше 1 даёт UB. В C значение вне `enum` не вызывает UB (проверено: UBSan молчит), но `switch` без `default` молча пропускает его, поэтому перечисление читают как целое и проверяют диапазон явно.

```c
/* Ошибка: UB - байт из данных скопирован в bool, значение 2 не является булевым */
#include <stdbool.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    unsigned char wire[1] = {2};
    bool flag;
    memcpy(&flag, wire, 1);
    if (flag) puts("set");
}
```

Правильная форма: счётчик сравнивают с остатком входа и умножают с проверкой переполнения до выделения.

```c
#include <stdckdint.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

/* read_counted - выделяет массив count элементов по sz байт, если он может уместиться в rest байт входа. */
void *read_counted(uint64_t count, size_t sz, size_t rest) {
    size_t bytes;
    if (count > SIZE_MAX || ckd_mul(&bytes, (size_t)count, sz) || bytes > rest) return nullptr;
    return calloc(count ? (size_t)count : 1, sz);
}

int main(void) {
    void *ok = read_counted(4, 4, 16);
    void *wrap = read_counted(UINT64_MAX / 2, 4, 16); /* произведение переполняется */
    void *big = read_counted(1u << 30, 4, 16); /* больше остатка входа */
    printf("%d %d %d\n", ok != nullptr, wrap == nullptr, big == nullptr); /* вывод: 1 1 1 */
    free(ok);
}
```

## Образ в памяти

### Превращай граф указателей в плоский образ из индексов и смещений

Указатели не переживают запись в файл и смену адресов, поэтому граф хранят индексами и смещениями от базы: образ можно записать целиком, загрузить и обходить без разбора. Это кэш одной сборки программы, а не формат обмена: порядок байт и раскладка хоста; в заголовок настоящего кэша добавляют версию и признак сборки, а при несовпадении кэш пересоздают. Правило из главы про [системный уровень](13-system-io.md) остаётся в силе: структуру на буфер не накладывают, поля читают аксессорами по смещению, а загрузка проверяет каждое смещение и индекс один раз, после чего аксессоры не проверяют ничего. Хэндлы вместо указателей объяснены в [главе про паттерны](21-patterns.md).

Пример: дерево путей, где узел хранит имя и родителя. Образ: заголовок из числа узлов и размера таблицы строк, записи `{смещение имени, индекс родителя}`, таблица строк.

```c
/* flat.h */
#ifndef FLAT_H
#define FLAT_H
#include <stddef.h>
#include <stdint.h>
#include <string.h>

typedef struct { const char *ptr; size_t len; } str;
typedef struct node { const char *name; const struct node *parent; } node;
typedef struct { str raw; uint32_t count; } flat;
constexpr uint32_t FLAT_NONE = UINT32_MAX;

/* u32_at - читает u32 по смещению off через memcpy: образ может лежать с любым выравниванием. */
static inline uint32_t u32_at(str s, size_t off) {
    uint32_t v;
    memcpy(&v, s.ptr + off, sizeof v);
    return v;
}

/* flat_build - превращает массив узлов (parent указывает внутрь того же массива) в образ.
 * Возврат: размер образа или -ENOSPC. Образ - кэш одной сборки: порядок байт и раскладка хоста.
 * Родитель должен стоять в массиве раньше потомка (меньший индекс): иначе flat_open отвергнет образ. */
int flat_build(unsigned char *out, size_t cap, const node *nodes, uint32_t n);

/* flat_open - проверяет образ целиком; после успеха аксессоры ничего не проверяют.
 * raw заимствуется. Возврат: 0 или -EBADMSG. */
int flat_open(str raw, flat *f);

/* Аксессоры: i меньше f->count. flat_name возвращает срез внутри образа. */
str flat_name(const flat *f, uint32_t i);
uint32_t flat_parent(const flat *f, uint32_t i); /* FLAT_NONE у корня */
#endif
```

```c
/* flat_build.c */
#include "flat.h"
#include <errno.h>
#include <string.h>

int flat_build(unsigned char *out, size_t cap, const node *nodes, uint32_t n) {
    size_t base = 8 + 8 * (size_t)n, pos = base;
    if (cap > UINT32_MAX) cap = UINT32_MAX; /* смещения 32-битные */
    if (cap < base) return -ENOSPC;
    for (uint32_t i = 0; i < n; i++) {
        size_t len = strlen(nodes[i].name) + 1;
        if (len > cap - pos) return -ENOSPC;
        memcpy(out + pos, nodes[i].name, len);
        uint32_t rec[2] = {(uint32_t)(pos - base), nodes[i].parent ? (uint32_t)(nodes[i].parent - nodes) : FLAT_NONE};
        memcpy(out + 8 + 8 * (size_t)i, rec, sizeof rec);
        pos += len;
    }
    uint32_t head[2] = {n, (uint32_t)(pos - base)};
    memcpy(out, head, sizeof head);
    return (int)pos;
}
```

Загрузка проверяет образ целиком и только после этого отдаёт аксессоры.

```c
/* flat.c */
#include "flat.h"
#include <errno.h>
#include <stdckdint.h>
#include <string.h>

int flat_open(str raw, flat *f) {
    size_t recs;
    if (raw.len < 8) return -EBADMSG;
    uint32_t n = u32_at(raw, 0), slen = u32_at(raw, 4);
    if (ckd_mul(&recs, (size_t)n, (size_t)8) || recs > raw.len - 8 || slen != raw.len - 8 - recs) return -EBADMSG;
    for (uint32_t i = 0; i < n; i++) {
        uint32_t off = u32_at(raw, 8 + 8 * (size_t)i), par = u32_at(raw, 12 + 8 * (size_t)i);
        if (off >= slen || !memchr(raw.ptr + 8 + recs + off, 0, slen - off)) return -EBADMSG; /* имя кончается в таблице */
        if (par != FLAT_NONE && par >= i) return -EBADMSG; /* родитель раньше потомка: циклов нет */
    }
    *f = (flat){raw, n};
    return 0;
}

str flat_name(const flat *f, uint32_t i) {
    const char *p = f->raw.ptr + 8 + 8 * (size_t)f->count + u32_at(f->raw, 8 + 8 * (size_t)i);
    return (str){p, strlen(p)};
}

uint32_t flat_parent(const flat *f, uint32_t i) { return u32_at(f->raw, 12 + 8 * (size_t)i); }
```

Три испорченных образа: родитель-потомок по кругу, смещение имени за таблицей, обрезанный образ. Все три отвергаются при открытии, обход не видит ни одного.

```c
/* main.c */
#include "flat.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    node nodes[3];
    nodes[0] = (node){"usr", nullptr};
    nodes[1] = (node){"lib", &nodes[0]};
    nodes[2] = (node){"libc.so", &nodes[1]};
    unsigned char img[128];
    int n = flat_build(img, sizeof img, nodes, 3);
    flat f;
    int rc = flat_open((str){(const char *)img, (size_t)n}, &f);
    str leaf = flat_name(&f, 2);
    printf("%d %d %.*s %u\n", n, rc, (int)leaf.len, leaf.ptr, flat_parent(&f, 2)); /* вывод: 48 0 libc.so 1 */

    uint32_t two = 2, far = 99;
    memcpy(img + 12, &two, sizeof two); /* родитель корня: узел 2 */
    int e1 = flat_open((str){(const char *)img, (size_t)n}, &f);
    memcpy(img + 12, &(uint32_t){FLAT_NONE}, sizeof two);
    memcpy(img + 16, &far, sizeof far); /* смещение имени узла 1 за таблицей строк */
    int e2 = flat_open((str){(const char *)img, (size_t)n}, &f);
    memcpy(img + 16, &(uint32_t){4}, sizeof far);
    int e3 = flat_open((str){(const char *)img, (size_t)n - 1}, &f);
    printf("%d %d %d\n", e1 == -EBADMSG, e2 == -EBADMSG, e3 == -EBADMSG); /* вывод: 1 1 1 */
}
```

#### Подводные камни

- Цикл в графе: индексы из недоверенного образа могут замкнуться. Правило «родитель раньше потомка» проверяется при загрузке и исключает циклы; для произвольных ссылок при обходе считают шаги и останавливаются на `count`.
- Образ из файла проще всего читать `fread` в буфер и проверять: буфер принадлежит процессу и никто его не меняет. Отображение `mmap` годится (`flat_open` работает по тому же срезу), но файл, усечённый другим процессом, даёт `SIGBUS` при обращении, а изменённый после проверки содержимое обходит проверку, поэтому отображают файл, который другие не пишут ([системный уровень](13-system-io.md)).
- Смещения 32-битные: размер образа ограничивают `UINT32_MAX` до записи (`flat_build` урезает `cap`), иначе обрезанное смещение указывает не туда.
- Образ арены целиком (память с указателями внутри) писать на диск не рекомендуется: указатели недействительны в другом запуске, а загрузка требует доверия к каждому из них.

## JSON

### Выбирай инструмент JSON по задаче: дерево, потоковый писатель или токены

Дерево в памяти (cJSON) удобно, когда документ небольшой и к нему обращаются произвольно, а потоковые инструменты (свой писатель, jsmn) нужны, когда документ большой, формат известен заранее или надо обойтись без выделений. Недоверенный JSON в любом случае получает потолок размера и глубины до разбора.

`cJSON` (`#include <cjson/cJSON.h>`, линковка `-lcjson`) строит и разбирает деревья. `cJSON_Parse` возвращает `nullptr` при ошибке; элементы, добавленные в массив или объект, переходят во владение контейнеру, а при неудаче добавления остаются у вызывающего; строку из `cJSON_PrintUnformatted` освобождают через `free`, дерево одним `cJSON_Delete` корня; общее дерево не потокобезопасно. Разбирать из буфера без завершающего NUL нужно через `cJSON_ParseWithLengthOpts`: `cJSON_Parse` читает до NUL и выйдет за буфер. Хвост после документа внутри заданной длины молча принимается, о нём говорит возвращаемый конец разбора.

```c
#include <cjson/cJSON.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    static const char text[] = "{\"port\":8080,\"id\":9007199254740993}  x";
    size_t n = sizeof text - 1;
    char *wire = malloc(n); /* как после read: без завершающего NUL */
    if (!wire) abort();
    memcpy(wire, text, n);
    const char *end;
    cJSON *doc = cJSON_ParseWithLengthOpts(wire, n, &end, false);
    if (!doc) return 1;
    printf("%zu\n", (size_t)(wire + n - end)); /* вывод: 3 */
    const cJSON *port = cJSON_GetObjectItemCaseSensitive(doc, "port");
    const cJSON *id = cJSON_GetObjectItemCaseSensitive(doc, "id");
    if (cJSON_IsNumber(port) && cJSON_IsNumber(id)) {
        printf("%d\n", port->valueint); /* вывод: 8080 */
        printf("%.0f\n", id->valuedouble); /* вывод: 9007199254740992 */
    }
    cJSON_Delete(doc);
    free(wire);
}
```

Число 9007199254740993 (2^53 + 1) cJSON хранит как `double` и теряет единицу. Сборка дерева: каждый вызов может вернуть `nullptr`, добавление может не удаться, а владение переходит только при успехе.

```c
#include <cjson/cJSON.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    cJSON *root = cJSON_CreateObject();
    if (!root) return 1;
    cJSON *tags = cJSON_AddArrayToObject(root, "tags"); /* создан и присоединён: принадлежит root */
    if (!tags || !cJSON_AddNumberToObject(root, "port", 8080)) { cJSON_Delete(root); return 1; }
    cJSON *t = cJSON_CreateString("a\"b");
    if (!t || !cJSON_AddItemToArray(tags, t)) { /* неудача: t остаётся у нас */
        cJSON_Delete(t);
        cJSON_Delete(root);
        return 1;
    }
    char *text = cJSON_PrintUnformatted(root);
    if (text) puts(text); /* вывод: {"tags":["a\"b"],"port":8080} */
    free(text);
    cJSON_Delete(root);
}
```

Писателю без дерева достаточно трёх вещей: запятая между элементами, экранирование строк и корректная печать чисел. Экранировать обязаны кавычку, обратную косую черту и все управляющие символы меньше 0x20; байты UTF-8 проходят как есть, но недопустимую последовательность писатель не заметит, поэтому её проверяют до записи.

```c
#include <stdio.h>

typedef struct { FILE *f; bool need_comma; } jw;

static void json_str(FILE *f, const char *s) {
    fputc('"', f);
    for (; *s; s++) {
        unsigned char c = (unsigned char)*s;
        if (c == '"' || c == '\\') fprintf(f, "\\%c", c);
        else if (c == '\n') fputs("\\n", f);
        else if (c < 0x20) fprintf(f, "\\u%04x", c);
        else fputc(c, f);
    }
    fputc('"', f);
}

static void jw_key(jw *j, const char *k) {
    if (j->need_comma) fputc(',', j->f);
    json_str(j->f, k);
    fputc(':', j->f);
    j->need_comma = true;
}

int main(void) {
    jw j = {stdout, false};
    fputc('{', j.f);
    jw_key(&j, "name");
    json_str(j.f, "a\"b\n\x01");
    jw_key(&j, "ratio");
    fprintf(j.f, "%.17g", 0.1);
    fputs("}\n", j.f); /* вывод: {"name":"a\"b\n\u0001","ratio":0.10000000000000001} */
}
```

Числа с плавающей точкой пишут с 17 значащими цифрами (`%.17g`): этого достаточно, чтобы `double` вернулся из текста тем же значением, а 15 цифр уже не хватает. Если процесс сменил `LC_NUMERIC` на локаль с запятой, и `printf`, и `strtod` начинают работать с запятой: свой писатель выдаёт невалидный JSON, а свой разборщик останавливается на точке. Поэтому числа для обмена форматируют и разбирают при `LC_NUMERIC`, равном `C`, а для вывода пользователю локаль включают только на время вывода (правило локали процесса: [локализация](18-localization.md)).

```c
#include <locale.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    double x = 0.1 + 0.2;
    char a[32], b[32];
    snprintf(a, sizeof a, "%.15g", x);
    snprintf(b, sizeof b, "%.17g", x);
    printf("%s %d\n", a, strtod(a, nullptr) == x); /* вывод: 0.3 0 */
    printf("%s %d\n", b, strtod(b, nullptr) == x); /* вывод: 0.30000000000000004 1 */
    if (!setlocale(LC_NUMERIC, "ru_RU.UTF-8")) return 1;
    snprintf(b, sizeof b, "%.17g", x);
    printf("%s %g\n", b, strtod("0.5", nullptr)); /* вывод: 0,30000000000000004 0 */
}
```

Для большого документа без дерева подходит `jsmn` (пакет `libjsmn-dev`, один заголовок `jsmn.h`): он не выделяет память, не копирует и размечает входной буфер токенами `{тип, начало, конец, размер}` в массиве вызывающего. Токен без декодирования: экранирование в строках и числа остаются текстом, поэтому большой идентификатор читают прямо из границ токена и не теряют точность. Хватило ли массива токенов, говорит `JSMN_ERROR_NOMEM`; размер массива служит заодно потолком на сложность документа.

Ключи объекта идут как токены-строки с `size` 1, за каждым стоит значение, поэтому поиск пропускает поддерево значения, а не идёт подряд: плоский перебор спутал бы ключ со строкой внутри массива.

```c
#include <jsmn.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int skip(const jsmntok_t *t, int i) { /* индекс токена после поддерева i */
    int j = i + 1;
    for (int k = 0; k < t[i].size; k++) j = skip(t, j);
    return j;
}

static int find(const char *js, const jsmntok_t *t, const char *key) { /* индекс значения ключа верхнего уровня */
    int i = 1;
    for (int k = 0; k < t[0].size; k++) {
        size_t len = (size_t)(t[i].end - t[i].start);
        if (len == strlen(key) && !strncmp(js + t[i].start, key, len)) return i + 1;
        i = skip(t, i + 1);
    }
    return -1;
}

int main(void) {
    const char *js = "{\"tags\":[\"id\",\"b\"],\"id\":9007199254740993}";
    jsmn_parser p;
    jsmntok_t tok[16];
    jsmn_init(&p);
    int n = jsmn_parse(&p, js, strlen(js), tok, 16);
    if (n < 1 || tok[0].type != JSMN_OBJECT) return 1;
    int v = find(js, tok, "id");
    if (v < 0 || tok[v].type != JSMN_PRIMITIVE) return 1;
    errno = 0;
    unsigned long long id = strtoull(js + tok[v].start, nullptr, 10);
    printf("%llu %d\n", id, errno == 0); /* вывод: 9007199254740993 1 */
    jsmn_init(&p);
    printf("%d\n", jsmn_parse(&p, js, strlen(js), tok, 4) == JSMN_ERROR_NOMEM); /* вывод: 1 */
}
```

#### Подводные камни

- Целые больше 2^53 теряют точность при любом разборе через `double`: идентификаторы такого размера передают строкой в формате либо читают из текста токена, как в примере с jsmn.
- `NaN` и бесконечность в JSON записать нельзя: значение проверяют через `isfinite` до печати.
- Хвост после документа внутри заданной длины `cJSON_ParseWithLengthOpts` принимает молча: целостность проверяют сравнением возвращённого конца разбора с концом буфера. Флаг завершающего NUL к буферу без NUL неприменим.
- Глубина вложенности у cJSON ограничена: 1000 уровней разбираются, 1050 дают `nullptr` (проверено). Размер входа ограничивают сами до вызова разбора.
- Печать и разбор чисел в cJSON от `LC_NUMERIC` не зависят (проверено под `ru_RU.UTF-8`: `1.5` разбирается и печатается точкой), а собственный код с `printf` и `strtod` зависит.

## Коротко

### protobuf-c: схема в файле .proto и сгенерированный код

`protoc-c` (пакет `protobuf-c-compiler`) по `.proto` строит `.pb-c.h` и `.pb-c.c`, библиотека `libprotobuf-c-dev` даёт `-lprotobuf-c`. Совместимость обеспечивают номера полей, как в практике про TLV: добавляют поля с новыми номерами, не переиспользуют удалённые. Версия 1.4.1 из пакетов не поддерживает `optional` в proto3: генератор отказывает с сообщением (проверено), присутствие поля выражают обёрткой или признаком.

```text
// file: sample.proto
syntax = "proto3";

message Sample {
  uint32 id = 1;
  string name = 2;
  repeated sint32 deltas = 3;
}
```

Владение: `sample__unpack` возвращает объект, выделенный аллокатором (`NULL` вместо аллокатора означает системный), и `NULL` при ошибке, в том числе на усечённом входе; строки и массивы внутри принадлежат объекту, освобождает всё один вызов `sample__free_unpacked`. Чтобы выделять из арены, вместо `NULL` передают `ProtobufCAllocator` с функциями `alloc`, `free` и указателем `allocator_data`. Буфер для `sample__pack` вызывающий готовит сам: размер даёт `sample__get_packed_size`.

```c
/* pb_demo.c */
#include "sample.pb-c.h"
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int32_t deltas[] = {-1, 2};
    Sample m = SAMPLE__INIT;
    m.id = 7;
    m.name = "alpha";
    m.n_deltas = 2;
    m.deltas = deltas;
    size_t n = sample__get_packed_size(&m);
    uint8_t *buf = malloc(n);
    if (!buf) abort();
    sample__pack(&m, buf);
    Sample *u = sample__unpack(NULL, n, buf);
    if (!u) abort();
    printf("%zu %u %s %zu %d\n", n, u->id, u->name, u->n_deltas, u->deltas[0]); /* вывод: 13 7 alpha 2 -1 */
    sample__free_unpacked(u, NULL);
    printf("%d\n", sample__unpack(NULL, n - 1, buf) == NULL); /* вывод: 1 */
    free(buf);
}
```

Сборка: `protoc-c --c_out=. sample.proto`, затем компиляция `pb_demo.c` и `sample.pb-c.c` с `-lprotobuf-c`. Сгенерированный код версии 1.4.1 на всех трёх компиляторах проходит флаги книги (`-Wpedantic -Wconversion -Werror`, проверено на схемах с перечислением, вложенными сообщениями, `oneof` и `map`); если новая версия генератора начнёт предупреждать, флаги ослабляют только для сгенерированного файла, а не для проекта.

### nanopb: protobuf без выделения памяти

Пакет `nanopb` ставит генератор `nanopb_generator`, пакет `libnanopb-dev` даёт заголовки и статическую библиотеку `libprotobuf-nanopb.a`. Тот же `.proto` даёт те же байты на проводе (13 байт на сообщении выше, как у protobuf-c). Размеры строк и массивов задаёт файл `.options` (`Sample.name max_size:16`, `Sample.deltas max_count:4`): поля становятся массивами фиксированного размера внутри структуры, и память под них выделять не нужно. Кодирование и разбор идут через потоки: `pb_ostream_from_buffer`, `pb_encode`, `pb_istream_from_buffer`, `pb_decode`; усечённый вход даёт `false`. Подходит для микроконтроллеров и там, где недопустима динамическая память. Генератор и библиотека собираются и работают под флагами книги (проверено сборкой и запуском).

### flatbuffers: чтение без разбора

Буфер читают по смещениям прямо на месте, без копирования и распаковки, как плоский образ из практики выше; для данных извне буфер сначала проверяют верификатором, иначе смещениям доверяют так же, как указателям. В пакетах noble есть `flatbuffers-compiler` (`flatc`) и `libflatbuffers-dev`, но генератора C в `flatc` нет (проверено по справке); для C нужен flatcc, которого в apt нет.

### TOML: файлы настроек с типами

TOML подходит для файлов настроек, которые правит человек: комментарии, строгие типы (целые, даты), таблицы. Разборщик для C, tomlc99, в пакетах noble отсутствует (вне apt); слои конфигурации и выбор формата описаны в [главе про конфигурацию](15-cli-and-config.md).

### MessagePack: двоичный JSON

Те же значения, что в JSON (числа, строки, массивы, ассоциативные массивы), в компактной двоичной записи; схема не нужна, поля идут по именам, а значит, размер больше, чем у схемного формата. В noble C-библиотека лежит в пакете `libmsgpack-dev` (msgpack-c 4.0.0; заголовок `msgpack.h`, линковка `-lmsgpackc`, файл `msgpack.pc`). Запись идёт в `msgpack_sbuffer` через `msgpack_packer_init`, чтение через `msgpack_unpack_next` с освобождением `msgpack_unpacked_destroy`; значение `{"id":7}` занимает 5 байт (упаковка и распаковка собираются и работают под флагами книги, проверено).

### CBOR: стандартизованный двоичный формат с теми же значениями

Пакет `libcbor-dev` (0.10.2, `-lcbor`). Контракт проверен запуском: элементы считаются по ссылкам, `cbor_decref(&item)` освобождает, а `cbor_move` передаёт владение при вставке в контейнер; `cbor_serialize_alloc` отдаёт буфер, который освобождает вызывающий через `free`; `cbor_load` при ошибке возвращает `nullptr`, а код лежит в `result.error.code` (на усечённом входе `CBOR_ERR_NOTENOUGHDATA`). Значение `{"id":7}` занимает те же 5 байт, что и в MessagePack. Размер входа ограничивают до вызова разбора.

## Что ещё стоит знать

YAML для недоверенного ввода не используют: сложная грамматика с алиасами и неявными типами делает разбор хрупким, а для файлов настроек есть TOML и JSON.

Из других JSON-библиотек в пакетах есть jansson (дерево со счётчиком ссылок); yyjson в noble отсутствует (вне apt).

Сжатие накладывают поверх сериализации (zstd, zlib, lz4: [справочник библиотек](28-appendix-linux-libraries.md)); размер после распаковки ограничивают так же, как длину поля, иначе небольшой вход раздувается в память.

ASN.1 DER и BER нужны для сертификатов и ключей; их разбирают библиотеки TLS и криптографии ([справочник библиотек](28-appendix-linux-libraries.md)), а не код приложения.
