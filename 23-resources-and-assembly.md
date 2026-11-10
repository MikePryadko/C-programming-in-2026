# Ресурсы и ассемблерные файлы

## Встраивание файлов

### Встраивай файл директивой `#embed`, а переносимый запасной путь держи на `.incbin`

Файл рядом с программой можно потерять, подменить или не найти при запуске, а встроенный в бинарник лежит в `.rodata` и доступен без файловой системы, поэтому шаблоны, схемы, шрифты и тестовые данные встраивают при сборке. Директива C23 `#embed` подставляет байты файла как список целочисленных констант. Clang 19 и 20 её понимают; GCC 14 отвечает «invalid preprocessing directive #embed», а макрос `__has_embed` у него не определён, поэтому проверять его надо через `#ifdef` (GCC 15 и новее директиву поддерживает). Переносимый запасной путь: директива ассемблера `.incbin` в файле `.S`, ей нужен только ассемблер, а не хостовая утилита, и она работает при кросс-сборке. Обе реализации прячут за одним заголовком: массив `имя_data` и размер `имя_len` без терминатора; остальной код не знает, как данные попали в бинарник, а система сборки выбирает реализацию.

```text
# file: banner.txt
Привет, мир
```

```c
/* res.h */
#ifndef RES_H
#define RES_H
#include <stddef.h>

extern const unsigned char banner_data[]; /* на байт длиннее banner_len: там 0 */
extern const size_t banner_len;
#endif
```

```c
/* res_embed.c */
/* Clang 19+ */
#include "res.h"

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wsign-conversion"
const unsigned char banner_data[] = {
#embed "banner.txt" suffix(, 0)
};
#pragma clang diagnostic pop
const size_t banner_len = sizeof banner_data - 1;
```

Вариант на ассемблере делает то же: кладёт байты файла в `.rodata`, ставит за ними нулевой байт и сам считает размер. Метка `banner_end` стоит сразу после `.incbin`, поэтому разность меток вычисляет ассемблер, а не C.

```text
/* banner_res.S */
    .section .rodata
    .balign 16
    .globl banner_data
    .type banner_data, @object
banner_data:
    .incbin "banner.txt"
banner_end:
    .byte 0
    .size banner_data, .-banner_data

    .balign 8
    .globl banner_len
    .type banner_len, @object
banner_len:
    .quad banner_end - banner_data
    .size banner_len, .-banner_len
    .section .note.GNU-stack,"",@progbits
```

```c
/* main.c */
#include <stdio.h>
#include "res.h"

int main(void) {
    printf("%zu %d\n", banner_len, banner_data[banner_len]); /* вывод: 21 0 */
    return 0;
}
```

Программа собирается с любой из двух реализаций: `main.c` и `res_embed.c` на Clang, `main.c` и `banner_res.S` на любом компиляторе; результат одинаков. Разность `end - start` двух внешних символов в C не вычисляют: это указатели на разные объекты, вычитание которых запрещено правилом из [главы про владение](04-memory-ownership.md), поэтому размер отдаёт ассемблер отдельным символом.

Параметры `#embed` задают границы данных и обрамление: `limit(n)` берёт не больше `n` байт, `prefix(...)` и `suffix(...)` добавляют элементы до и после, `if_empty(...)` подставляется, когда ресурс пуст (в том числе из-за `limit(0)`). Тип массива определяет, как читаются байты.

```c
/* main.c */
/* Clang 19+ */
#include <stdio.h>

#pragma clang diagnostic ignored "-Wsign-conversion"

static const char as_char[] = {
#embed "banner.txt" limit(8)
};
static const unsigned char as_byte[] = {
#embed "banner.txt" limit(8)
};
static const unsigned char framed[] = {
#embed "banner.txt" limit(2) prefix(0xAA, ) suffix(, 0xBB)
};
static const unsigned char fallback[] = {
#embed "banner.txt" limit(0) if_empty(7, 7)
};

int main(void) {
    printf("%zu %d %d\n", sizeof as_char, as_char[0], as_byte[0]); /* вывод: 8 -48 208 */
    printf("%zu %d %d\n", sizeof framed, framed[0], framed[3]); /* вывод: 4 170 187 */
    printf("%zu %d\n", sizeof fallback, fallback[0]); /* вывод: 2 7 */
    return 0;
}
```

#### Подводные камни

- Тип элементов. Байт 0xD0 (первый байт «П» в UTF-8) в массиве `char` на x86-64 читается как `-48`, в `unsigned char` как `208`; на aarch64 `char` беззнаковый. Двоичные данные встраивай как `unsigned char`, `char` оставляй для текста.
- `-Wsign-conversion` на `unsigned char`. Байты от 0x80 в массиве `unsigned char` под `-Wconversion -Werror` дают ошибку «implicit conversion changes signedness» (Clang 19 даже для простого `#embed`, Clang 20 при `prefix` и `suffix`); массив `char` предупреждений не даёт. Предупреждение выключают прагмой вокруг массива, как в примерах (механика в [возможностях компилятора](08-compiler-features.md)).
- Терминатор. Ресурс без `suffix(, 0)` не является C-строкой: `strlen` и `printf("%s")` по нему выйдут за массив. В примере нулевой байт лежит за концом данных, а `имя_len` его не включает, поэтому и `fwrite(data, 1, len, f)` пишет ровно содержимое файла.
- Путь. `#embed "файл"` ищет файл как `#include`: рядом с исходником, затем в каталогах `-I`; путь в `.incbin` считается иначе (раздел про сборку).
- GCC 14. Файл с `#embed` не собирается, а `#ifdef __has_embed` у него ложно; защита условием внутри одного файла работает, но надёжнее разделить реализации по файлам, как в примере, и выбирать файл в системе сборки.
- Права на запись. Данные в `.rodata` только для чтения: запись в них роняет процесс, изменяемую копию делай явно (следующий раздел).

## Доступ к ресурсам из приложения

### Отдавай ресурс срезом, а изменяемую копию делай в арене

Ресурс это неизменяемая последовательность байт с длиной, то есть срез `str` из [главы про владение](04-memory-ownership.md): он указывает в `.rodata`, ничем не владеет и живёт до конца процесса, поэтому освобождать его не нужно. Реестр `{имя, данные, размер}` с поиском по имени даёт приложению виртуальную файловую систему из встроенных файлов; размер хранится указателем на константу, чтобы таблица оставалась статической (значение `const size_t` из другого файла не является константным выражением, а его адрес является). Для нескольких ресурсов макрос ассемблера `.macro` убирает повтор.

```text
# file: banner.txt
Привет, мир
```

```text
# file: help.txt
usage: app [-v]
quit: q
```

```text
/* res_data.S */
    .section .rodata
    .macro RES name, file
    .balign 16
    .globl \name\()_data
    .type \name\()_data, @object
\name\()_data:
    .incbin "\file"
\name\()_end:
    .byte 0
    .size \name\()_data, .-\name\()_data
    .balign 8
    .globl \name\()_len
    .type \name\()_len, @object
\name\()_len:
    .quad \name\()_end - \name\()_data
    .size \name\()_len, .-\name\()_len
    .endm

    RES banner, "banner.txt"
    RES help, "help.txt"
    .section .note.GNU-stack,"",@progbits
```

```c
/* res.h */
#ifndef RES_H
#define RES_H
#include <stddef.h>

typedef struct { const char *ptr; size_t len; } str;

extern const unsigned char banner_data[], help_data[];
extern const size_t banner_len, help_len;

/* res_find - ищет встроенный ресурс по имени.
 * Возврат: срез в .rodata (неизменяемый, живёт до конца процесса, ptr[len] == 0) или {nullptr, 0}. */
str res_find(const char *name);
#endif
```

```c
/* res.c */
#include <string.h>
#include "res.h"

typedef struct { const char *name; const unsigned char *data; const size_t *len; } entry;

static const entry table[] = {
    {"banner.txt", banner_data, &banner_len},
    {"help.txt", help_data, &help_len},
};

str res_find(const char *name) {
    for (size_t i = 0; i < sizeof table / sizeof table[0]; i++)
        if (strcmp(table[i].name, name) == 0) return (str){(const char *)table[i].data, *table[i].len};
    return (str){nullptr, 0};
}
```

Изменяемую копию берут из арены (определение из [главы про владение](04-memory-ownership.md), здесь в виде заголовка). Для кода, которому нужен `FILE *` (разбор построчно, чужая библиотека, читающая из потока), `fmemopen` оборачивает срез в поток без копирования.

```c
/* arena.h */
#ifndef ARENA_H
#define ARENA_H
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

typedef struct { char *beg, *end; } arena;

static inline arena arena_init(void *buf, size_t cap) { char *p = buf; return (arena){p, p + cap}; }

static inline void *arena_alloc(arena *a, size_t size, size_t align, size_t count) {
    size_t pad = -(uintptr_t)a->beg & (align - 1);
    size_t room = (size_t)(a->end - a->beg);
    if (pad > room || (count && size > (room - pad) / count)) abort();
    char *p = a->beg + pad;
    a->beg = p + size * count;
    return memset(p, 0, size * count);
}
#define NEW(a, T, n) ((T *)arena_alloc((a), sizeof(T), alignof(T), (n)))
#endif
```

```c
/* main.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
#include "arena.h"
#include "res.h"

int main(void) {
    str b = res_find("banner.txt");
    printf("%zu %d\n", b.len, res_find("none.txt").ptr == nullptr); /* вывод: 21 1 */

    char mem[256];
    arena a = arena_init(mem, sizeof mem);
    char *copy = NEW(&a, char, b.len + 1);
    memcpy(copy, b.ptr, b.len);
    copy[0] = '*';
    printf("%d %d\n", copy[0], b.ptr[0]); /* вывод: 42 -48 */

    str h = res_find("help.txt");
    FILE *f = fmemopen((void *)h.ptr, h.len, "r");
    if (!f) return 1;
    char line[64];
    int lines = 0;
    while (fgets(line, sizeof line, f)) lines++;
    fclose(f);
    printf("%d\n", lines); /* вывод: 2 */
    return 0;
}
```

Копия живёт в арене и исчезает вместе с ней; оригинал остался нетронутым (`-48` в последней строке про исходный байт). Чтобы отдать ресурс другой программе или библиотеке, которая принимает только путь, выгрузи его в файл: пиши в цикле до полной записи и заменяй целевой файл атомарно через временный файл, как в `save_atomic` из [главы про системный ввод-вывод](13-system-io.md) (там же причины каждого шага); подходит вызов `save_atomic(dir, name, b.ptr, b.len)`. Для библиотек, которым нужен именно дескриптор, Linux даёт `memfd_create`: анонимный файл в памяти, в который копируют ресурс.

#### Подводные камни

- `fmemopen` принимает `void *`, а не `const void *`: для чтения (`"r"`) приведение убирает `const` безопасно, но режим `"w"` или `"a"` на `.rodata` роняет процесс.
- Тип размера. Длина лежит в `.rodata` восьмибайтовой директивой `.quad`, поэтому в C она объявлена как `size_t`; объявление `int` прочитает только половину слова.
- Выравнивание. `.balign 16` перед данными нужно, если ресурс разбирают приведением к структуре (так делать не стоит, см. [главу про системный ввод-вывод](13-system-io.md)); для побайтного чтения оно не обязательно.
- Реестр с ростом числа ресурсов становится длинным; тогда таблицу генерируют скриптом сборки по списку файлов ([генерация кода](02-build.md)).

## Ресурсы в сборке

### Объявляй файл ресурса зависимостью объекта

Правка встроенного файла должна вызывать пересборку, иначе в программе остаётся старая версия. Зависимости объектов система сборки берёт из depfile, который пишет компилятор с флагом `-MMD`: для `#embed` в нём есть и сам файл ресурса (в проверке Clang 19 и 20 записали в зависимости `/w/src/res/d.bin`), для `.incbin` нет (в проверке depfile на `.S` содержал только сам `.S` и у Clang, и у GCC), поэтому изменение данных пересборку не вызывало. Для `.incbin` зависимость объявляют явно. Путь в `.incbin` считается от рабочего каталога компилятора и каталогов `-I`, а не от каталога `.S`: CMake и Meson запускают компилятор в каталоге сборки, поэтому исходный каталог передают через `-I`. Исходники берутся из предыдущего примера.

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(app C ASM)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)

set_source_files_properties(res_data.S PROPERTIES OBJECT_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/banner.txt;${CMAKE_CURRENT_SOURCE_DIR}/help.txt")

add_executable(app main.c res.c res_data.S)
target_include_directories(app PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
```

В Meson у цели нет списка зависимостей от данных (`link_depends` влияет только на компоновку), поэтому `.S` собирают как `custom_target` с `depend_files`.

```text
# file: meson.build
project('app', 'c', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')
cc = meson.get_compiler('c')

res_o = custom_target('res_data',
    input: 'res_data.S',
    output: 'res_data.o',
    depend_files: files('banner.txt', 'help.txt'),
    command: cc.cmd_array() + ['-I' + meson.current_source_dir(), '-c', '@INPUT@', '-o', '@OUTPUT@'])

executable('app', 'main.c', 'res.c', res_o, include_directories: include_directories('.'))
```

Зависимость проверяют правкой файла и повторной сборкой: после изменения `banner.txt` вторая сборка должна пересобрать `res_data.S` и дать новый размер. Для `#embed` отдельной строки в системе сборки не требуется, если depfile включён (CMake и Meson с Ninja включают его сами).

#### Подводные камни

- Нет `OBJECT_DEPENDS`. После правки `banner.txt` `ninja` отвечает «no work to do» и выдаёт старый размер.
- Путь относительно каталога `.S`. Из каталога сборки компилятор отвечает «Could not find incbin file» (Clang) или «file not found» (gas); решение: `-I` на каталог с ресурсами или абсолютный путь.
- `link_depends` в Meson. Он перекомпонует программу, но не пересоберёт объект с `.incbin`, поэтому остаётся старый ресурс.

## Коротко

### `xxd -i` и собственный генератор массива

`xxd -i файл` печатает массив C, и это самый простой вариант без Clang и ассемблера, но 10 МБ превращаются в 65 МБ исходника, который компилятор разбирает секунды (замер в конце раздела). Собственный генератор `bin2c` на 20 строк даёт те же массивы с нужными именами и терминатором; подключается тем же `custom_target` или `add_custom_command`, что и любой генератор ([генерация кода](02-build.md)).

### `objcopy` и `ld -r -b binary`

`ld -r -b binary -o res.o файл` превращает файл в объект с символами `_binary_<путь>_start`, `_binary_<путь>_end` и `_binary_<путь>_size`, где в имени каждый небуквенный и нецифровой знак пути заменён на `_` (`res/my-file.bin` даёт `_binary_res_my_file_bin_start`). Данные попадают в секцию `.data` (записываемую), а объект не имеет ноты `.note.GNU-stack`, поэтому GNU ld предупреждает «missing .note.GNU-stack section implies executable stack»; секцию переименовывает `objcopy --rename-section .data=.rodata,alloc,load,readonly,data,contents`. Размер берут как `end - start` и сами символы объявляют как массивы; `_size` это абсолютный символ, и GCC 14 с `-pie` не линкует обращение к нему («relocation R_X86_64_PC32 against absolute symbol»), Clang линкует. Явные `.incbin` и `#embed` дают те же данные без этих ловушек.

### Сжатие при встраивании

Крупные ресурсы (шрифты, словари, образы) встраивают сжатыми, а при старте распаковывают в арену: бинарник меньше, но приложение платит временем запуска и памятью. Формат и библиотеку выбирают по этим двум числам; распакованный размер хранят рядом с данными, чтобы выделить арену заранее.

### Архив в хвосте исполняемого файла

Ресурсы без пересборки кода: приложение открывает `/proc/self/exe`, читает с конца оглавление и находит данные, дописанные к ELF-файлу (`cat app data.bin > app2`). Загрузчик ELF хвост игнорирует; цена в том, что ресурсы вне `.rodata` и не отображаются как часть образа, зато их можно менять утилитой после сборки.

### GResource

В экосистеме GLib ресурсы описывает XML-файл, а `glib-compile-resources` собирает из него файл, подключаемый к программе и читаемый через `g_resource_*`; подходит приложению, которое уже зависит от GLib, остальным хватает таблицы выше.

### Стоимость сборки

Проверка на случайном файле 10 МБ (Clang 19 и GCC 14, один запуск, `-c`): `#embed` около 0,5 с, `.incbin` около 0,05 с, массив из `xxd -i` 12 с на Clang 19 и 17 с на GCC 14.

## Когда нужен ассемблер

### Пиши ассемблер, только когда C не выражает нужного, и начинай с вывода компилятора

Ассемблерный файл теряет переносимость, невидим для санитайзеров и требует отдельного теста, поэтому его оправдывает только то, что C выразить не может: переключение стеков (корутины, потоки пользовательского уровня), точная последовательность инструкций (вход в систему, начальный код загрузки), особое соглашение вызова или ручной векторный цикл, которого компилятор не строит. Лестница выбора: обычный C с `-O2`; встроенные функции и интринсики ([возможности компилятора](08-compiler-features.md)); вставка `asm` в одну-две инструкции; и только затем файл `.S`. Перед каждым шагом смотри, что компилятор уже сделал: цикл суммы из этой главы он векторизует сам, поэтому как пример ассемблера он учебный, а не рекомендация (правило про ассемблерные вставки в libc-функции из [главы про производительность](20-performance.md) остаётся в силе).

```sh
printf '#include <stdint.h>\n#include <stddef.h>\nint64_t f(const int32_t *a, size_t n) { int64_t s = 0; for (size_t i = 0; i < n; i++) s += a[i]; return s; }\n' \
  | clang-19 -O2 -x c -S -o - - | grep -c paddq
```

В проверке команда вывела 4 для Clang 19 и 20 (цикл векторизован) и 0 для GCC 14 при `-O2` (цикл оставлен скалярным); результат зависит от компилятора и уровня, поэтому смотри вывод своей сборки. Пример того, чего C не выражает, это переключение стека: функция сохраняет сохраняемые вызванной функцией регистры (разд. про ABI ниже) на текущем стеке, запоминает указатель стека и загружает другой.

```text
/* ctx_swap.S */
/* void ctx_swap(void **save_sp, void *load_sp) */
    .text
    .globl ctx_swap
    .type ctx_swap, @function
ctx_swap:
    pushq %rbp
    pushq %rbx
    pushq %r12
    pushq %r13
    pushq %r14
    pushq %r15
    movq %rsp, (%rdi)
    movq %rsi, %rsp
    popq %r15
    popq %r14
    popq %r13
    popq %r12
    popq %rbx
    popq %rbp
    ret
    .size ctx_swap, .-ctx_swap
    .section .note.GNU-stack,"",@progbits
```

Новый стек готовят так, чтобы шесть `pop` и `ret` вошли в точку входа: на вершине лежат шесть нулей, затем адрес функции, затем нулевой адрес возврата, а вершина выровнена по 16 байт (выравнивание объясняет раздел про ABI).

#### Подводные камни

- Переносимость. Файл на x86-64 не собирается на aarch64: для каждой архитектуры нужен свой файл и запасная реализация на C (следующий раздел).
- Санитайзеры. Обращения из `.S` ни ASan, ни UBSan не проверяют; единственная защита это тест против эталона (раздел про проверку).
- Лишний ассемблер. Функции `memcpy`, `strlen` и подобные в libc уже векторные; своя версия медленнее и добавляет ошибки.

## Ассемблерный файл в сборке и проверка

### Подключай `.S` в исходники цели с выбором по архитектуре и запасным C

Файл `.S` проходит препроцессор C, поэтому в нём работают `#include`, `#if` и макросы вроде `__x86_64__`; `.s` препроцессор не видит. Начинай с `.S`: заголовок `<cet.h>` и условия по флагам доступны только в нём. Комментарии пиши `/* */`: символ однострочного комментария у разных архитектур разный. Файл подключается как обычный исходник: в CMake нужно `project(app C ASM)`, в Meson достаточно языка `c`. Файл выбирают по архитектуре, а там, где его нет, остаётся реализация на C, она же служит эталоном для теста.

```c
/* sum.h */
#ifndef SUM_H
#define SUM_H
#include <stddef.h>
#include <stdint.h>

/* sum_i32 - сумма n элементов a, знаково расширенных до 64 бит.
 * a: n элементов, заимствуется; допустим nullptr при n == 0. Возврат: сумма; ошибок нет. */
int64_t sum_i32(const int32_t *a, size_t n);

/* sum_i32_ref - то же, простой цикл на C; эталон для тестов и запасная реализация. */
int64_t sum_i32_ref(const int32_t *a, size_t n);
#endif
```

```c
/* sum_ref.c */
#include "sum.h"

int64_t sum_i32_ref(const int32_t *a, size_t n) {
    int64_t s = 0;
    for (size_t i = 0; i < n; i++) s += a[i];
    return s;
}

#if !defined(__x86_64__) && !defined(__aarch64__)
int64_t sum_i32(const int32_t *a, size_t n) { return sum_i32_ref(a, n); }
#endif
```

```text
/* sum_x86_64.S */
#include <cet.h>
    .text
    .globl sum_i32
    .type sum_i32, @function
sum_i32:
    _CET_ENDBR
    .cfi_startproc
    xorl %eax, %eax
    testq %rsi, %rsi
    jz 2f
1:  movslq (%rdi), %rdx
    addq %rdx, %rax
    addq $4, %rdi
    decq %rsi
    jnz 1b
2:  ret
    .cfi_endproc
    .size sum_i32, .-sum_i32
    .section .note.GNU-stack,"",@progbits
```

```text
/* sum_aarch64.S */
    .text
    .globl sum_i32
    .type sum_i32, %function
sum_i32:
#if defined(__ARM_FEATURE_BTI_DEFAULT)
    bti c
#endif
    .cfi_startproc
    mov x2, #0
    cbz x1, 2f
1:  ldrsw x3, [x0], #4
    add x2, x2, x3
    subs x1, x1, #1
    b.ne 1b
2:  mov x0, x2
    ret
    .cfi_endproc
    .size sum_i32, .-sum_i32
#if defined(__ARM_FEATURE_BTI_DEFAULT)
    .section .note.gnu.property,"a"
    .balign 8
    .long 4, 16, 5
    .asciz "GNU"
    .long 0xc0000000, 4, 1, 0
#endif
    .section .note.GNU-stack,"",%progbits
```

Синтаксис в примерах AT&T (источник слева, приёмник справа, суффикс размера у мнемоники, `%` у регистров); директива `.intel_syntax noprefix` переключает на Intel-запись (в проверке её приняли GNU as и встроенный ассемблер Clang), а NASM использует Intel-запись изначально. Главный файл вызывает функцию как обычную.

```c
/* main.c */
#include <stdio.h>
#include "sum.h"

int main(void) {
    int32_t a[3] = {1, 2, 3};
    printf("%ld\n", (long)sum_i32(a, 3)); /* вывод: 6 */
    return 0;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(app C ASM)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)

set(SUM_ASM "")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64)$")
  set(SUM_ASM sum_x86_64.S)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64)$")
  set(SUM_ASM sum_aarch64.S)
endif()

add_executable(app main.c sum_ref.c ${SUM_ASM})
```

```text
# file: meson.build
project('app', 'c', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')

asm_by_cpu = {'x86_64': 'sum_x86_64.S', 'aarch64': 'sum_aarch64.S'}
srcs = ['main.c', 'sum_ref.c']
cpu = host_machine.cpu_family()
if cpu in asm_by_cpu
  srcs += asm_by_cpu[cpu]
endif
executable('app', srcs)
```

Флаги профиля из [главы про флаги](01-toolchain.md) на `.S` не мешают: в проверке `-std=c23 -Wall -Wextra -Wpedantic -Wconversion -Wshadow -Wformat=2 -Werror` и санитайзеры не дали ни предупреждений, ни ошибок, но санитайзеры ничего в этом файле не инструментируют.

Второй путь: готовый объект или архив от другого ассемблера или поставщика. NASM (пакет `nasm`) собирает объект одной командой, а система сборки линкует его как обычный файл; чужой объект сначала осматривают.

```text
; sum_nasm.asm
default rel
global sum_i32:function (sum_i32.end - sum_i32)
section .text
sum_i32:
    xor eax, eax
    test rsi, rsi
    jz .done
.loop:
    movsxd rdx, dword [rdi]
    add rax, rdx
    add rdi, 4
    dec rsi
    jnz .loop
.done:
    ret
.end:
section .note.GNU-stack noalloc noexec nowrite progbits
```

```sh
nasm -f elf64 sum_nasm.asm -o sum_nasm.o
readelf -sW sum_nasm.o | grep sum_i32     # тип FUNC, размер 23, GLOBAL
readelf -SW sum_nasm.o | grep GNU-stack   # нота должна быть
readelf -n sum_nasm.o                     # свойства IBT/SHSTK: у этого объекта нет
cc main.c sum_nasm.o -o app -Wl,-z,noexecstack -Wl,-z,cet-report=error
```

Последняя команда в проверке закончилась ошибкой `missing IBT and SHSTK properties` на объекте NASM: объект без свойства CET при `-fcf-protection=full` не проходит жёсткую проверку линкера (подробности в разделе про обязательное содержимое). В CMake готовый объект подключают как файл: `target_link_libraries(app PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/vendor/sum_nasm.o)`, в Meson аргументом `objects: files('vendor/sum_nasm.o')`; исходник NASM собирает `enable_language(ASM_NASM)` в CMake и `add_languages('nasm')` в Meson.

#### Подводные камни

- `.s` вместо `.S`. Без препроцессора не работают `#include <cet.h>` и условия по архитектуре.
- Выбор файла по `CMAKE_SYSTEM_PROCESSOR`. При кросс-сборке значение берётся из файла toolchain и может оказаться другим, чем у хоста ([кросс-компиляция](22-cross-compilation.md)).
- Нет запасной реализации. На третьей архитектуре сборка упадёт неопределённым символом, а не предупреждением.
- Чужой объект в архиве. Нота `.note.GNU-stack` и свойства CET проверяются у каждого объектного файла архива отдельно, и достаточно одного без них, чтобы бинарник получил исполняемый стек или потерял защиту.

### Проверяй ассемблерную функцию дифференциальным тестом против эталона на C

Санитайзеры не инструментируют код из `.S`: чтение за концом массива из ассемблера ASan не заметит, а UBSan не знает о выходе знаковой арифметики за диапазон. Остаётся сравнение ответов с простой реализацией на тех же входах ([глава про тесты](09-testing.md) даёт приём, для фаззинга ту же проверку оборачивают в цель libFuzzer). Блоки, на которых различаются длины 0, 1, нечётные и степени двойки, ловят ошибки хвоста цикла; массив выделяется ровно `n` элементов, чтобы чтения из C за границей ловил ASan.

```c
/* sum_test.c */
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include "sum.h"

static uint32_t rng = 12345;
static uint32_t next(void) { rng = rng * 1664525u + 1013904223u; return rng; }

int main(void) {
    for (size_t n = 0; n <= 100; n++) {
        int32_t *a = malloc((n ? n : 1) * sizeof *a);
        if (!a) abort();
        for (size_t i = 0; i < n; i++) a[i] = (int32_t)next();
        if (sum_i32(a, n) != sum_i32_ref(a, n)) {
            printf("mismatch n=%zu\n", n);
            return 1;
        }
        free(a);
    }
    puts("ok"); /* вывод: ok */
    return 0;
}
```

x86-64 тест запускается нативно на трёх компиляторах под ASan и UBSan, aarch64 версия под `qemu-aarch64` без санитайзеров (санитайзеры на кросс-цели не используются, причина в [кросс-компиляции](22-cross-compilation.md)). Кросс-сборка пакетной установкой: `clang-19 --target=aarch64-linux-gnu -fuse-ld=lld sum_test.c sum_ref.c sum_aarch64.S -o app`, запуск `qemu-aarch64 -L /usr/aarch64-linux-gnu ./app` (без `--sysroot`; подробности в [главе про кросс-компиляцию](22-cross-compilation.md)). Ассемблерный вариант под конкретное расширение процессора (AVX2, NEON) подключают как ещё одного кандидата в таблицу диспетчеризации из [главы про возможности компилятора](08-compiler-features.md), а тест вызывает каждого кандидата напрямую.

#### Подводные камни

- Чтение за концом массива. ASan его в `.S` не видит: в проверке версия `sum_i32`, читающая на элемент больше, завершилась под ASan без отчёта; тест против эталона ловит такую ошибку, только если лишнее значение меняет сумму, поэтому в тест включай входы, где за массивом лежит известный ненулевой байт. Логическую ошибку тест ловит сразу: версия, пропускающая нечётный хвост, провалила его на `n = 1`.
- Один и тот же компилятор. Проверяй ту же сборку, что идёт в продукт: `-O2` и флаги профиля без санитайзеров, санитайзеры только на C-части.
- Эталон из той же реализации. Эталон должен быть заведомо простым циклом, а не копией оптимизированной логики.

## Вызовы и сохранение регистров

### Соблюдай соглашение вызова и выравнивание стека

Ассемблерная функция живёт по правилам вызывающего: где лежат аргументы, какие регистры она обязана вернуть нетронутыми и как выровнен стек, и нарушение даёт ошибки в произвольном месте программы. Правила двух архитектур, которые нужны в этой главе:

| | System V x86-64 | AAPCS64 |
|---|---|---|
| Целочисленные аргументы | rdi, rsi, rdx, rcx, r8, r9 | x0-x7 |
| Результат | rax (rdx для второго слова) | x0 |
| Сохраняет вызванная функция | rbx, rbp, r12-r15 | x19-x28 и x29 |
| Стек перед вызовом | выровнен по 16 байт (при входе в функцию на 8 меньше) | sp выровнен по 16 байт всегда |
| Адрес возврата | на стеке | в регистре x30, функция с вызовами его сохраняет |

Остальные регистры функция портит свободно. Пример обходит массив и вызывает C-функцию на каждый элемент: указатель, счётчик и адрес обратного вызова должны пережить вызов, поэтому лежат в сохраняемых регистрах, которые функция сначала сохраняет, а потом восстанавливает.

```text
/* each_x86_64.S */
#include <cet.h>
    .text
    .globl each_i32
    .type each_i32, @function
each_i32:
    _CET_ENDBR
    .cfi_startproc
    pushq %rbx
    .cfi_adjust_cfa_offset 8
    .cfi_rel_offset %rbx, 0
    pushq %r12
    .cfi_adjust_cfa_offset 8
    .cfi_rel_offset %r12, 0
    pushq %r13
    .cfi_adjust_cfa_offset 8
    .cfi_rel_offset %r13, 0
    movq %rdi, %rbx
    movq %rsi, %r12
    movq %rdx, %r13
1:  testq %r12, %r12
    jz 2f
    movl (%rbx), %edi
    callq *%r13
    addq $4, %rbx
    decq %r12
    jmp 1b
2:  popq %r13
    popq %r12
    popq %rbx
    .cfi_adjust_cfa_offset -24
    ret
    .cfi_endproc
    .size each_i32, .-each_i32
    .section .note.GNU-stack,"",@progbits
```

```text
/* each_aarch64.S */
    .text
    .globl each_i32
    .type each_i32, %function
each_i32:
    .cfi_startproc
    stp x29, x30, [sp, #-48]!
    .cfi_def_cfa_offset 48
    .cfi_offset x29, -48
    .cfi_offset x30, -40
    mov x29, sp
    stp x19, x20, [sp, #16]
    str x21, [sp, #32]
    .cfi_offset x19, -32
    .cfi_offset x20, -24
    .cfi_offset x21, -16
    mov x19, x0
    mov x20, x1
    mov x21, x2
1:  cbz x20, 2f
    ldr w0, [x19], #4
    blr x21
    sub x20, x20, #1
    b 1b
2:  ldr x21, [sp, #32]
    ldp x19, x20, [sp, #16]
    ldp x29, x30, [sp], #48
    .cfi_restore x29
    .cfi_restore x30
    .cfi_def_cfa_offset 0
    ret
    .cfi_endproc
    .size each_i32, .-each_i32
    .section .note.GNU-stack,"",%progbits
```

```c
/* main.c */
#include <stddef.h>
#include <stdio.h>

void each_i32(const int *a, size_t n, void (*cb)(int));

static int total;
static void add(int v) { total += v; }

int main(void) {
    int a[4] = {1, 2, 3, 4};
    each_i32(a, 4, add);
    printf("%d\n", total); /* вывод: 10 */
    return 0;
}
```

На x86-64 при входе в функцию стек смещён на 8 байт (адрес возврата), три `push` по 8 байт возвращают его к выравниванию по 16, поэтому `callq` в цикле происходит на выровненном стеке; с двумя `push` вызов `printf` в обратном вызове мог бы упасть на SSE-инструкции в libc. Чётность числа сохранённых регистров поэтому не случайна: при чётном числе добавляют `subq $8, %rsp`.

#### Подводные камни

- Порча сохраняемого регистра. Функция, изменившая `rbx` и не восстановившая его, ломает вызывающего далеко от места ошибки; если нужен ещё регистр, сохрани и восстанови его.
- Невыровненный стек перед `call`. Падение внутри libc на инструкции `movaps` без видимой связи с ассемблером.
- Возврат из функции с несбалансированным стеком. Число `push` и `pop` на каждом пути выхода одинаково, иначе `ret` берёт чужой адрес.
- Регистры вызываемой C-функции. Всё, что не входит в сохраняемые регистры (на x86-64 rax, rcx, rdx, rsi, rdi, r8-r11), после `call` содержит мусор: нужное держи в сохраняемых регистрах или на стеке.

## Обязательное в файле

### Включай в каждый `.S` ноту стека, метки перехода, описание символа и директивы разворота стека

Каждая строка шапки и подвала файла защищает от отдельной беды, поэтому набор не сокращают.

Нота `.note.GNU-stack` сообщает компоновщику, что объект не требует исполняемого стека. Без неё GNU ld (2.42) помечает стек программы исполняемым, `readelf -lW` показывает у `GNU_STACK` флаги `RWE`, а линковка печатает предупреждение «missing .note.GNU-stack section implies executable stack»; с нотой, либо с флагом `-Wl,-z,noexecstack` на линковке, стек `RW`; lld без ноты тоже даёт `RW`. Исполняемый стек открывает выполнение данных со стека, поэтому нота нужна независимо от линкера: файл не должен зависеть от флагов чужой линковки.

```sh
cc -c sum_x86_64.S -o sum.o && cc main.o sum.o -o app
readelf -lW app | grep GNU_STACK        # RW: нота в файле есть
readelf -lW app_bare | grep GNU_STACK   # RWE: объект собран без ноты
```

`_CET_ENDBR` из `<cet.h>` раскрывается в `endbr64` при `-fcf-protection` и в пустоту без него; инструкция нужна первой в каждой функции, на адрес которой могут перейти косвенно (при IBT переход на другую инструкцию вызывает аварию). Тот же заголовок добавляет в объект ноту `.note.gnu.property` со свойствами IBT и SHSTK. Если хотя бы один объект без этой ноты попал в линковку, итоговый бинарник теряет свойство целиком (в проверке `readelf -n` собранной программы перестал показывать `x86 feature: IBT, SHSTK`), а флаг `-Wl,-z,cet-report=error` превращает это в ошибку `bare.o: error: missing IBT and SHSTK properties`. Подключай заголовок в каждом файле, а проверку включай в профиль продукта. У GCC 14 в Ubuntu `-fcf-protection=full` включён по умолчанию, у Clang 19 нужен явный флаг; в `.S` макрос `__CET__` отражает флаг сборки этого файла. На aarch64 аналог: инструкция `bti c` первой в функции и нота свойства `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` (в примерах выше записана вручную под условием `__ARM_FEATURE_BTI_DEFAULT`, которое задаёт `-mbranch-protection=standard`); Clang с флагом ноту сам не добавил, а lld с `-z bti-report=error` указал объект без неё.

Директивы `.globl`, `.type` и `.size` делают символ глобальным, функцией и задают его длину: без `.type` и `.size` дизассемблер и `perf` показывают безымянный участок, а в отладчике нет границ функции. `.hidden` скрывает символ вне разделяемой библиотеки: он виден при компоновке, но не экспортируется.

Директивы CFI (`.cfi_startproc`, `.cfi_adjust_cfa_offset`, `.cfi_rel_offset` и остальные) описывают, как восстановить кадр вызывающего на каждой инструкции. Без них раскрутка стека обрывается в ассемблерной функции: `backtrace()` из обратного вызова в `each_i32` без CFI вернул 2 кадра, с CFI 6 (на Clang 19 и GCC 14); LLDB 20 в той же проверке восстановил стек и без директив (команды `bt` и `frame select` описаны в [главе про отладку](10-debugging.md)), но `backtrace()` без директив стек потерял, поэтому CFI пиши всегда.

Позиционная независимость: код и данные адресуют относительно счётчика команд, а не абсолютным адресом. Сборка `.S` с `movq $msg, %rdi` в разделяемую библиотеку падает на линковке («relocation R_X86_64_32S against `.rodata' can not be used when making a shared object»), с `leaq msg(%rip), %rdi` собирается; вызов внешней функции принято писать с `@PLT` (в проверке `jmp puts` без него тоже слинковался: компоновщик создал запись сам).

```text
/* greet_x86_64.S */
#include <cet.h>
    .section .rodata
msg:
    .asciz "hi"
    .text
    .globl greet
    .hidden greet
    .type greet, @function
greet:
    _CET_ENDBR
    .cfi_startproc
    leaq msg(%rip), %rdi
    jmp puts@PLT
    .cfi_endproc
    .size greet, .-greet
    .section .note.GNU-stack,"",@progbits
```

```c
/* main.c */
#include <stdio.h>
void greet(void);

int main(void) {
    greet(); /* вывод: hi */
    return 0;
}
```

Хвостовой переход `jmp puts@PLT` оставляет вызов `puts` на выровненном стеке вызывающего. Синтаксис метки типа различается: `@function` у x86-64 и `%function` у aarch64 (на 32-битном ARM `@` начинает комментарий, поэтому для ARM принята форма с `%`); в проверке обе формы приняли и встроенный ассемблер Clang 19, и GNU as на обеих архитектурах, но пиши принятую для архитектуры.

#### Подводные камни

- Нота есть, но линковка чужого объекта без неё. Достаточно одного такого файла, чтобы стек стал исполняемым; осмотр объектов (`readelf -SW | grep GNU-stack`) и `-Wl,-z,noexecstack` на линковке гарантируют результат.
- Забытая CFI в функции, которая вызывает другие. Отладчик может справиться, а `backtrace()` и инструменты, читающие таблицы разворота, обрываются на этой функции.
- Абсолютный адрес данных. Работает в исполняемом файле без PIE и ломает сборку разделяемой библиотеки, поэтому адресуй через `%rip` с самого начала.
- `.hidden` на символе, который должны видеть снаружи. Он пропадает из динамической таблицы разделяемой библиотеки (в проверке `greet` исчез из `readelf --dyn-syms`), и `dlsym` его не найдёт; скрывай только внутренние функции.

## Что ещё стоит знать

Формат объектов и соглашение вызова у Windows и macOS другие (COFF и Mach-O, свой набор регистров аргументов), и ассемблерные файлы там пишутся отдельно; цели и их ограничения описаны в [главе про кросс-компиляцию](22-cross-compilation.md). Для одной функции под расширение процессора достаточно атрибута `target` и интринсиков ([возможности компилятора](08-compiler-features.md)), файл `.S` нужен позже. Корутины и переключение контекста без собственного ассемблера дают `makecontext` и `swapcontext` из libc.
