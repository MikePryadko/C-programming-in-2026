# Возможности компилятора

## Карта атрибутов и расширений

Книга пользуется обеими формами атрибутов. Стандартные `[[nodiscard]]`, `[[deprecated]]`, `[[maybe_unused]]`, `[[fallthrough]]`, `[[noreturn]]` пишутся только в стандартной форме. Атрибут GNU на объявлении функции или объекта пишется как `[[gnu::...]]` (под `-std=c23 -Wpedantic` проходит на Clang 19, Clang 20 и GCC 14), а `__attribute__((...))` остаётся там, где позиция `[[...]]` неудобна или невозможна: на локальной переменной (`cleanup`), после списка параметров перед телом функции (`diagnose_if`, `enable_if`), у атрибутов, которые Clang принимает только в форме `__attribute__` (`no_sanitize`, `optnone`: `__has_c_attribute(gnu::no_sanitize)` в Clang равно 0); `assume` в такой форме принимают все три, а `[[gnu::assume]]` понимает только GCC 14, `[[clang::assume]]` не принимает ни Clang 19 и 20, ни GCC (предупреждение «unknown attribute»). Сами атрибуты GNU это расширение GCC и Clang, а не стандарт C.

Запасной вариант на случай, когда компилятор атрибута не знает, зависит от того, что атрибут делает. Диагностический (`nodiscard`, `format`, `deprecated`, `warn_unused_result`) при отсутствии заменяется пустым макросом: программа без него работает так же. Атрибут, меняющий смысл программы (`cleanup`, `packed`, `section`, `constructor`, `weak`, `alias`), при отсутствии даёт `#error`: молчаливый пропуск превратил бы его в ошибку времени выполнения. `visibility("default")` к ним не относится: без него символ остаётся видимым по умолчанию, а при сборке с `-fvisibility=hidden` пропадает только экспорт, который ловит проверка при сборке или тест (поэтому макрос пустой, как в [сборке](02-build.md)). Неизвестный атрибут под `-Werror` и так ошибка на всех трёх компиляторах, поэтому проверку оформляют макросами в одном заголовке `compat.h` (практика про `__has_*` ниже); проверки функций libc туда не входят, их делает сборка ([сборка](02-build.md)).

Карта построена по тексту книги и показывает форму записи, которой в книге пользуются; что в ней отмечено «здесь», разобрано ниже. Главы про сериализацию, логирование, базы данных, кросс-компиляцию и ресурсы пользуются теми же формами записи и в карту отдельными строками не вынесены.

| Возможность и форма | Что даёт | Clang и GCC | Если нет | Глава |
| --- | --- | --- | --- | --- |
| `[[nodiscard]]`, `[[maybe_unused]]`, `[[deprecated]]` | предупреждение об игнорируемом результате, неиспользуемом имени, устаревшем API | оба, стандарт C23 | пустой макрос | [API и модули](07-api-and-modules.md), [ошибки](05-errors-and-cleanup.md) |
| `[[fallthrough]]`, `[[noreturn]]` | явный проход в следующую метку, функция не возвращает управление | оба, стандарт C23 | пустой макрос | здесь, раздел «Коротко» |
| `[[unsequenced]]`, `[[reproducible]]` | чистота функции, разрешение объединять вызовы | GCC 15; остальные пропускают с предупреждением | пустой макрос по `__has_c_attribute` | [производительность](20-performance.md) |
| `[[gnu::format(printf, 3, 4)]]` на объявлении, `__attribute__((format(printf, 1, 2)))` | проверка форматной строки у обёртки | оба | пустой макрос | [строки](06-strings-and-buffers.md), [ошибки](05-errors-and-cleanup.md), [API и модули](07-api-and-modules.md) |
| `__attribute__((cleanup(f)))` на локальной переменной | вызов функции при выходе переменной из области видимости | оба | `#error` | [ошибки](05-errors-and-cleanup.md) |
| `__attribute__((counted_by(n)))` | длина гибкого массива для проверки границ | Clang 19 и 20; GCC 14 игнорирует | пустой макрос по `__has_attribute` | [память](04-memory-ownership.md) |
| `__attribute__((nonstring))` | символьный массив без завершающего нуля | GCC; в Clang 19 и 20 `__has_attribute(nonstring)` равно 0 | пустой макрос | [строки](06-strings-and-buffers.md) |
| `[[gnu::visibility("default")]]`, `__attribute__((visibility("default")))` | экспорт символа из библиотеки | оба | пустой макрос (символ остаётся видимым по умолчанию) | [API и модули](07-api-and-modules.md), [паттерны](21-patterns.md), [сборка](02-build.md) |
| `__attribute__((used))` | не удалять неиспользуемый объект | оба | `#error` | здесь (символы и секции), [отладка](10-debugging.md) |
| `__attribute__((packed))` | убирает дополнение между полями | оба | `#error` | [производительность](20-performance.md), здесь (прагмы) |
| `__attribute__((always_inline))`, `[[gnu::noinline]]`, `[[gnu::cold]]`, `[[gnu::flatten]]` | управление подстановкой и размещением кода | оба | пустой макрос | здесь (оптимизация), [производительность](20-performance.md) |
| `__attribute__((capability("mutex")))`, `guarded_by` и `-Wthread-safety` | статическая проверка блокировок | только Clang | пустой макрос | [потоки](12-concurrency.md), [набор флагов](01-toolchain.md) |
| `__attribute__((overloadable))` | перегрузка функций по типам | только Clang | `_Generic` | [контейнеры](11-containers-and-generics.md) |
| `_Nonnull`, `_Nullable` | допустимость нулевого указателя в типе | только Clang | пустой макрос | [API и модули](07-api-and-modules.md) |
| `unreachable()`, `<stdbit.h>` | недостижимая ветка, операции над битами | C23, все три | `__builtin_unreachable`, `__builtin_clz` и родственные | [типы и UB](03-types-and-ub.md), здесь |
| `ckd_add`, `ckd_sub`, `ckd_mul` | проверка переполнения | C23, все три | `__builtin_add_overflow` | [типы и UB](03-types-and-ub.md) |
| `__has_include`, `__has_attribute`, `__has_c_attribute`, `__has_builtin` | наличие заголовка, атрибута, встроенной функции | оба | вариант без возможности | [сборка](02-build.md), [память](04-memory-ownership.md), [отладка](10-debugging.md), здесь |
| `__builtin_debugtrap()` | останов в отладчике | Clang; запасной вариант `raise(SIGTRAP)` | как указано | [отладка](10-debugging.md) |
| `#pragma`, встроенные функции, `weak`, `alias`, `section`, `constructor`, `ifunc`, `target` | прагмы, символы, диспетчеризация | оба, различия указаны в практиках | по правилу выше | здесь |

## Диагностика

### Глуши предупреждение на одном месте, а не флагом на весь проект

Предупреждение, выключенное флагом (`-Wno-conversion`), пропадает во всём коде, а прагма `push`, `ignored`, `pop` действует вокруг одного вызова, поэтому остальной код продолжает проверяться. Прагму `#pragma GCC diagnostic` понимают и GCC, и Clang; `#pragma clang diagnostic` GCC не знает, и под `-Wall -Werror` это ошибка `unknown-pragmas`. Внутри макроса прагму пишут через `_Pragma`.

```c
/* diag.h */
#ifndef DIAG_H
#define DIAG_H
#define DIAG_PRAGMA(x) _Pragma(#x)
#define DIAG_PUSH DIAG_PRAGMA(GCC diagnostic push)
#define DIAG_POP DIAG_PRAGMA(GCC diagnostic pop)
#define DIAG_IGNORE(w) DIAG_PRAGMA(GCC diagnostic ignored w)
#endif
```

```c
/* legacy.c */
#include "diag.h"

[[deprecated("use new_checksum")]] unsigned old_checksum(const char *s);
unsigned new_checksum(const char *s);

unsigned checksum_compat(const char *s, int legacy) {
    if (!legacy) return new_checksum(s);
    DIAG_PUSH
    DIAG_IGNORE("-Wdeprecated-declarations")
    unsigned r = old_checksum(s);
    DIAG_POP
    return r;
}
```

Чужие заголовки, которые нельзя править, не оборачивают в прагмы по месту: каталог подключают флагом `-isystem` (компилятор не показывает предупреждения из его заголовков), а в самом заголовке то же делает `#pragma GCC system_header`. Своя часть проекта остаётся под полным набором предупреждений.

#### Подводные камни

- Прагма `#pragma clang diagnostic` без защиты: см. выше, пиши `#pragma GCC diagnostic` или оборачивай в `#ifdef __clang__`.
- Неизвестное имя в `ignored`. Под `-Werror` ошибкой оно становится и на GCC (`-Wpragmas`), и на Clang (`-Wunknown-warning-option`); предупреждение, которое знает только один компилятор, прячут за проверкой возможностей (см. практику про `__has_*`).
- Незакрытый `push`. Каждому `push` нужен свой `pop`, иначе настройки утекают на остальную часть файла; в макросах держи пару рядом, как `DIAG_PUSH` и `DIAG_POP`.
- Глушение вместо исправления. Если предупреждение указывает на настоящую ошибку (знак, усечение), правят код; прагмой закрывают вызов устаревшей функции или заведомо безопасное приведение с комментарием, почему.

## Определение возможностей

### Проверяй возможности через `__has_*`, а не по номеру версии

Версия компилятора не говорит, что он умеет: GCC 14 игнорирует `counted_by`, который Clang знает давно, а `__GNUC__` определяет и Clang (равно 4). Проверка возможности напрямую даёт правильный ответ на любом компиляторе, поэтому все такие проверки собирают в один заголовок `compat.h`, а остальной код пользуется его макросами.

Доступные проверки: `__has_include(<h>)` (заголовок), `__has_attribute(имя)` (GNU-атрибут), `__has_c_attribute(имя)` или `__has_c_attribute(пространство::имя)` (стандартный и `gnu::`), `__has_builtin(имя)`, `__has_feature(имя)` и `__has_extension(имя)` (Clang; на GCC 14 тоже определены), `__has_warning("-Wимя")` (только Clang). Предопределённые макросы: `__STDC_VERSION__`, `__clang__`, `__SANITIZE_ADDRESS__`. Для санитайзера Clang 19 и 20 макрос `__SANITIZE_ADDRESS__` не определяют: надёжный способ `__has_feature(address_sanitizer)` (GCC 14 принимает и его).

```c
/* compat.h */
#ifndef COMPAT_H
#define COMPAT_H

#if __has_attribute(format)
#define COMPAT_FORMAT(fmt, first) __attribute__((format(printf, fmt, first)))
#else
#define COMPAT_FORMAT(fmt, first)
#endif

#if __has_attribute(cleanup)
#define COMPAT_CLEANUP(fn) __attribute__((cleanup(fn)))
#else
#error "нужен __attribute__((cleanup)): компилятор не поддерживается"
#endif

#if __has_builtin(__builtin_expect)
#define COMPAT_UNLIKELY(c) __builtin_expect(!!(c), 0)
#else
#define COMPAT_UNLIKELY(c) (c)
#endif

#if __has_c_attribute(unsequenced)
#define COMPAT_UNSEQUENCED [[unsequenced]]
#else
#define COMPAT_UNSEQUENCED
#endif

#ifdef __has_warning
#if __has_warning("-Wcast-function-type-strict")
#define COMPAT_IGNORE_STRICT_CAST _Pragma("clang diagnostic ignored \"-Wcast-function-type-strict\"")
#endif
#endif
#ifndef COMPAT_IGNORE_STRICT_CAST
#define COMPAT_IGNORE_STRICT_CAST
#endif
#endif
```

```c
/* main.c */
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include "compat.h"

static void say(const char *fmt, ...) COMPAT_FORMAT(1, 2);
static void say(const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    vprintf(fmt, ap);
    va_end(ap);
}

static void release(char **p) { free(*p); }

int main(void) {
    char *buf COMPAT_CLEANUP(release) = malloc(8);
    if (COMPAT_UNLIKELY(!buf)) abort();
    say("%d items\n", 3); /* вывод: 3 items */
    return 0;
}
```

Диагностический атрибут `format` при отсутствии заменён пустым макросом, меняющий смысл `cleanup` даёт `#error`: это правило из карты. Атрибут языка `[[unsequenced]]` проверяется через `__has_c_attribute`, а `COMPAT_IGNORE_STRICT_CAST` подставляет прагму Clang только там, где предупреждение с таким именем существует (на GCC макрос пустой). Вложенный `#ifdef __has_warning` обязателен: следующий пример, записанный в одно условие, на Clang собирается, а на GCC 14 падает в препроцессоре.

```c
/* has_trap.c */
/* Ошибка: GCC 14 отвечает "missing binary operator before token", Clang собирает */
#if defined(__has_warning) && __has_warning("-Wconversion")
int has_conversion_warning = 1;
#endif
int unit;
```

#### Подводные камни

- `defined(X) && X(...)` в одном `#if`. Препроцессор разбирает всё выражение целиком, и незнакомое имя с круглой скобкой для GCC синтаксическая ошибка; вложи `#if` внутрь `#ifdef`. Это касается `__has_warning`: `__has_feature` и `__has_builtin` GCC 14 определяет.
- `__GNUC__` как признак GCC. Clang определяет его (значение 4): GCC отличают по `defined(__GNUC__) && !defined(__clang__)`, а чаще не отличают вовсе и проверяют возможность.
- `__has_builtin` равно 1, а использовать с тем же успехом нельзя. На aarch64 `__has_builtin(__builtin_cpu_supports)` в Clang 19 равно 1 (имена расширений ARM он принимает, а `"asimd"` отвергает предупреждением «invalid cpu feature string»), в GCC 14 равно 0, а `__builtin_cpu_init` в обоих отсутствует; проверяй именно ту встроенную, которую вызываешь, и ветку для x86 защищай ещё и `#if defined(__x86_64__)`.
- Значение `__has_c_attribute` для стандартных атрибутов это дата версии (например 202311), а не 1: сравнивай с нулём, а не на равенство.

## Прагмы и встроенные функции

### Ограничивайся коротким списком прагм и защищай те, что знает один компилятор

Прагмы привязаны к компилятору, а неизвестную под `-Wall -Werror` GCC и Clang считают ошибкой, поэтому каждую из них используют осознанно. Прагмы диагностики разобраны выше; здесь остальное, что встречается на практике.

`#pragma once` работает на всех трёх компиляторах и заменяет защиту от повторного включения; в книге принята классическая защита `#ifndef` ([API и модули](07-api-and-modules.md)), потому что она стандартна. В основном файле (`.c`) `#pragma once` даёт предупреждение.

`#pragma pack(push, 1)` и `pop` убирают дополнение между полями. Взятие адреса поля упакованной структуры даёт указатель, выровненный хуже, чем требует его тип, а его разыменование под UBSan ошибка. Для структуры с атрибутом `packed` оба компилятора предупреждают об этом (`-Waddress-of-packed-member`), а для `#pragma pack` в проверке предупреждения не было, так что внимательность нужна самому. Поле читают присваиванием или копируют через `memcpy`. Разбор бинарных сообщений описан в [системном вводе-выводе](13-system-io.md).

```c
#include <stdint.h>
#include <stdio.h>

#pragma pack(push, 1)
typedef struct { uint8_t tag; uint32_t len; } hdr;
#pragma pack(pop)
static_assert(sizeof(hdr) == 5, "hdr без дополнения");

int main(void) {
    hdr h = {1, 7};
    uint32_t len = h.len;
    printf("%zu %u\n", sizeof(hdr), len); /* вывод: 5 7 */
    return 0;
}
```

```c
/* Ошибка: UB, указатель на поле упакованной структуры не выровнен для uint32_t */
#include <stdint.h>
#include <stdio.h>

#pragma pack(push, 1)
typedef struct { uint8_t tag; uint32_t len; } hdr;
#pragma pack(pop)

int main(void) {
    hdr h = {1, 7};
    uint32_t *p = &h.len;
    printf("%u\n", *p);
    return 0;
}
```

`#pragma GCC poison имя` превращает любое последующее упоминание имени в ошибку компиляции: так в проекте запрещают функции, которые книга запрещает (`gets`, `strcpy`, `sprintf`). Прагму ставят после всех `#include`, иначе она сработает на системных заголовках. Оба компилятора принимают её одинаково.

```c
/* poison.c */
#include <stdio.h>
#include <string.h>
#pragma GCC poison gets strcpy sprintf

int main(void) {
    char buf[8];
    strcpy(buf, "ab");
    return 0;
}
```

`#pragma GCC visibility push(hidden)` и `pop` делают все объявленные между ними символы скрытыми, а единичные экспортируемые помечают `[[gnu::visibility("default")]]` ([API и модули](07-api-and-modules.md)). На практике то же достигается флагом `-fvisibility=hidden`; прагма нужна, когда нельзя менять флаги всего проекта.

```c
#pragma GCC visibility push(hidden)
int hidden_fn(int x) { return x + 1; }
#pragma GCC visibility pop

[[gnu::visibility("default")]] int shown_fn(int x) { return x + 2; }
```

Остальное одной строкой. `#pragma STDC FP_CONTRACT OFF` запрещает компилятору сливать умножение и сложение в одну операцию с другим округлением: Clang её принимает, а GCC 14 игнорирует (под `-Werror` это ошибка `unknown-pragmas`), поэтому для GCC используют флаг `-ffp-contract=off`. `#pragma omp simd` просит векторизовать цикл и нужна только вместе с флагом `-fopenmp-simd`: без него GCC 14 под `-Werror` отвечает ошибкой, а Clang молча принимает.

### Оборачивай встроенные функции в макросы с запасным вариантом

Встроенные функции дают то, что не выразить обычным кодом: подсказки оптимизатору, быстрые инструкции процессора, сведения, известные только компилятору. Это расширения GCC и Clang, поэтому их проверяют через `__has_builtin`, а новые C23-аналоги (`<stdbit.h>`, `<stdckdint.h>`, `unreachable()`) предпочтительнее: стандартные имена переносимы.

```c
#include <stdio.h>
#include <stdbit.h>

int main(void) {
    unsigned x = 0x00f0;
    printf("%d %d %d\n", __builtin_clz(x), __builtin_ctz(x), __builtin_popcount(x)); /* вывод: 24 4 4 */
    printf("%u\n", stdc_leading_zeros(0u)); /* вывод: 32 */
    printf("%d\n", __builtin_types_compatible_p(int, const int)); /* вывод: 1 */
    printf("%d\n", __builtin_constant_p(sizeof x)); /* вывод: 1 */
    __builtin_prefetch(&x, 0, 3);
    return 0;
}
```

| Встроенная | Что делает | Заметка |
| --- | --- | --- |
| `__builtin_clz`, `__builtin_ctz`, `__builtin_popcount` | число ведущих и хвостовых нулей, число единиц | `clz(0)` и `ctz(0)` дают UB; `stdc_leading_zeros(0u)` определена и вернула 32 ([типы и UB](03-types-and-ub.md)) |
| `__builtin_prefetch(p, rw, locality)` | подсказка подгрузить данные в кэш | не меняет смысл программы; выигрыш проверяют измерением |
| `__builtin_constant_p(x)` | 1, если `x` известно при компиляции | зависит от уровня оптимизации: годится для выбора ветки в макросе, не для логики |
| `__builtin_types_compatible_p(T1, T2)` | совместимость типов, константа | верхние квалификаторы игнорирует: `(int, const int)` даёт 1; выбор по типу обычно делают `_Generic` ([контейнеры](11-containers-and-generics.md)) |
| `__builtin_trap()` | немедленное аварийное завершение без обработчиков | сигнал SIGILL на x86-64; точка останова для отладчика `__builtin_debugtrap` описана в [отладке](10-debugging.md) |
| `__builtin_dump_struct(&obj, printf)` | печатает поля структуры с именами и значениями | только Clang (в GCC 14 имени нет), для отладочной печати |
| `no_sanitize("address")` | выключает инструментацию санитайзера у функции | атрибут, не встроенная; в Clang пишется `__attribute__`, форма `[[gnu::no_sanitize]]` там не распознаётся |

Глубоко стоит разобрать три встроенные, которым нет места в таблице: `__builtin_expect`, `assume` вместе с `unreachable` и `__builtin_object_size`.

#### Подсказывай ветвление через `__builtin_expect` и измеряй эффект

`__builtin_expect(условие, ожидаемое)` говорит компилятору, какая ветка вероятна, и тот раскладывает код так, чтобы частый путь шёл без переходов; смысл программы подсказка не меняет, поэтому она безопасна. Применяй для редких путей ошибок в горячем коде, а не по всей программе: неверная подсказка замедляет, а выигрыш от верной проверяют измерением.

```c
#include <stdio.h>
#include <stdlib.h>

#if __has_builtin(__builtin_expect)
#define UNLIKELY(c) __builtin_expect(!!(c), 0)
#else
#define UNLIKELY(c) (c)
#endif

[[noreturn, gnu::cold]] static void die(const char *msg) {
    fprintf(stderr, "%s\n", msg);
    abort();
}

static int checked_div(int a, int b) {
    if (UNLIKELY(b == 0)) die("division by zero");
    return a / b;
}

int main(void) {
    printf("%d\n", checked_div(84, 2)); /* вывод: 42 */
    return 0;
}
```

Двойное отрицание `!!` приводит условие к 0 или 1: подсказка сравнивается с числом. Атрибут `cold` у функции ошибки сообщает, что она вызывается редко. `[[likely]]` и `[[unlikely]]` в C23 не входят: под `-Wall` Clang отвечает «unknown attribute 'likely' ignored», GCC 14 «'likely' attribute ignored». Для вероятности есть `__builtin_expect_with_probability`.

#### Сообщай оптимизатору проверенное через `assume`, а не угадай

Подсказка «здесь это верно» позволяет компилятору упростить код: для `x >= 0` деление на два превращается в один сдвиг без поправки на знак. Ошибочная подсказка неопределённое поведение, поэтому её пишут только там, где условие следует из проверки выше или контракта, и в отладочной сборке проверяют `assert`. Расширения для нескольких компиляторов: `__attribute__((assume(c)))` как отдельный оператор принимают Clang 19, 20 и GCC 14 (в проверке деление на два стало короче на обоих); `__builtin_assume(c)` есть только в Clang, а на GCC работает запасной вариант `if (!(c)) __builtin_unreachable();`.

```c
/* assume.c */
#include <assert.h>

#if __has_attribute(assume)
#define ASSUME(c) __attribute__((assume(c)))
#else
#define ASSUME(c) do { if (!(c)) __builtin_unreachable(); } while (0)
#endif

#ifdef NDEBUG
#define CHECKED_ASSUME(c) ASSUME(c)
#else
#define CHECKED_ASSUME(c) assert(c)
#endif

int half_checked(int x) {
    CHECKED_ASSUME(x >= 0);
    return x / 2;
}

int half(int x) {
    ASSUME(x >= 0);
    return x / 2;
}
```

`__builtin_unreachable()` и `unreachable()` из `<stddef.h>` (C23) делают одно и то же: управление сюда не придёт, ошибка неопределённое поведение; в C23 пиши стандартный `unreachable()` ([типы и UB](03-types-and-ub.md)), а `__builtin_unreachable` оставляй для запасного варианта в макросе. Условие в `assume` не должно иметь побочных эффектов: Clang такое предупреждает и не учитывает.

#### Узнавай размер объекта через `__builtin_object_size` и его динамический вариант

`__builtin_object_size(p, 0)` возвращает, сколько байт осталось от `p` до конца объекта, если это известно компилятору, и `(size_t)-1` (максимум), если нет; на этом построены проверки `_FORTIFY_SOURCE` ([набор флагов](01-toolchain.md)). Динамический вариант `__builtin_dynamic_object_size` умеет и размер, вычисленный во время выполнения (`malloc(n)`).

```c
#include <stdio.h>
#include <stdlib.h>

static char table[64];

int main(int argc, char **argv) {
    (void)argv;
    char local[16];
    char *heap = malloc((size_t)argc * 32);
    if (!heap) abort();
    printf("%zu\n", __builtin_object_size(local, 0)); /* вывод: 16 */
    printf("%zu\n", __builtin_object_size(table + 8, 0)); /* вывод: 56 */
    printf("%zu\n", __builtin_dynamic_object_size(heap, 0)); /* вывод: 32 */
    free(heap);
    return 0;
}
```

Для массива, видимого прямо в выражении, ответ известен и при `-O0`. Для указателя, пришедшего через вызов функции, и для результата `malloc` нужна оптимизация (при `-O0` в проверке пришло «неизвестно»), а через границу неподставленной функции (`noinline`) ответ «неизвестно» всегда; поэтому по нему не строят обязательные проверки, а размер сравнивают с длиной сами.

#### Подводные камни

- `__builtin_constant_p` и `__builtin_object_size` зависят от оптимизации. Годятся для отладочных проверок и выбора быстрого пути, но не для логики программы.
- `__builtin_clz(0)` и `__builtin_ctz(0)` неопределены. Проверяй аргумент или используй `stdc_leading_zeros`.
- Встроенная без запасного варианта ломает сборку на третьем компиляторе: оборачивай в макрос по `__has_builtin`, как `COMPAT_UNLIKELY` выше.
- Неверное `assume` даёт неопределённое поведение. Запасной вариант с `__builtin_unreachable` UBSan ловит на всех трёх («execution reached an unreachable program point»), а сам атрибут `assume` на Clang 19 в проверке не сообщил ничего (Clang 20 и GCC 14 сообщили); в отладочной сборке то же условие проверяет `assert`.

## Оптимизация

### Управляй подстановкой и циклами атрибутами и прагмами, проверяй отчётом компилятора

Компилятор оптимизирует и без подсказок, а они нужны, когда измерение показало, что он ошибается, или для отладки. Для циклов у Clang и GCC общая `#pragma GCC unroll N` (Clang принимает её тоже); у Clang есть `#pragma clang loop vectorize(enable) unroll_count(4) interleave_count(2)`, у GCC `#pragma GCC ivdep`; каждая из двух последних в чужом компиляторе даёт `unknown-pragmas` (под `-Werror` ошибку), поэтому `clang loop` ставь под `#ifdef __clang__`, а `ivdep` под `#if defined(__GNUC__) && !defined(__clang__)`. Для функций: `noinline` оставляет вызов видимым в профиле и в стеке; `always_inline` вместе со `static inline` принуждает подстановку; `cold` помечает функцию как редко вызываемую; `flatten` подставляет в функцию всё, что она вызывает; для отладки оптимизированного кода `optnone` (Clang) или `optimize("O0")` (GCC).

```c
#include <stdio.h>
#include <stdlib.h>

#if __has_attribute(optnone)
#define NO_OPT __attribute__((optnone))
#elif __has_attribute(optimize)
#define NO_OPT __attribute__((optimize("O0")))
#else
#define NO_OPT
#endif

[[gnu::always_inline]] static inline int clamp(int v, int lo, int hi) {
    return v < lo ? lo : v > hi ? hi : v;
}

[[gnu::flatten]] static int process(const int *a, int n) {
    int s = 0;
#pragma GCC unroll 4
    for (int i = 0; i < n; i++) s += clamp(a[i], 0, 100);
    return s;
}

[[gnu::noinline]] NO_OPT static int probe(int x) { return x * 2 + 1; }

int main(void) {
    int a[5] = {-5, 10, 200, 30, 40};
    printf("%d %d\n", process(a, 5), probe(3)); /* вывод: 180 7 */
    return 0;
}
```

Что компилятор действительно сделал, показывают отчёты об оптимизации. У Clang это `-Rpass` (выполнено), `-Rpass-missed` (не выполнено) и `-Rpass-analysis` (почему), у GCC `-fopt-info`; полную запись для разбора инструментами даёт Clang `-fsave-optimization-record`. Подстановку показывает `-Rpass=inline`, пример в [производительности](20-performance.md).

```c
/* remark.c */
#include <stddef.h>

void scale(float *restrict dst, const float *restrict src, size_t n) {
    for (size_t i = 0; i < n; i++) dst[i] = src[i] * 2.0f;
}

[[gnu::noinline]] static int step(int x) { return x * 3 + 1; }

int twice_step(int x) { return step(x) + step(x + 1); }
```

```sh
clang -std=c23 -O2 -Rpass='loop-vectorize|inline' -Rpass-missed='loop-vectorize|inline' -c remark.c
gcc -std=c23 -O3 -fopt-info-vec-optimized -fopt-info-vec-missed -c remark.c
clang -std=c23 -O2 -fsave-optimization-record -c remark.c
```

Clang печатает `remark: vectorized loop (vectorization width: 4, interleaved count: 2)` для цикла `scale` и `remark: 'step' not inlined into 'twice_step' because it should never be inlined` из-за `noinline`; GCC печатает `optimized: loop vectorized using 16 byte vectors`, но только с `-O3`: на `-O2` его осторожная модель стоимости этот цикл не векторизовала (в отчёте `missed: couldn't vectorize loop`). Числа зависят от целевого процессора, поэтому сверяй по фразе, а не по ширине. Запись `-fsave-optimization-record` ложится рядом с объектным файлом (`remark.opt.yaml`).

#### Подводные камни

- Несколько `-Rpass=` подряд: действует последний, остальные теряются. Объединяй в одно регулярное выражение, как выше, иначе цикл «не векторизован» потому, что отчёт о нём выключен.
- Отчёт сверяют с `-O2` без санитайзеров. При `-O1` с ASan remark о векторизации не печатается; это не значит, что цикл не векторизован в боевой сборке.
- `always_inline` на большой функции раздувает код и вытесняет кэш инструкций; ставь по профилю ([производительность](20-performance.md)).
- Забытый `optnone` или `optimize("O0")` оставляет функцию без оптимизации и в боевой сборке; ставь их только в отладочной.

## Символы, секции и конструкторы

### Управляй символами атрибутами, а порядок инициализации не принимай за гарантию

Символьные атрибуты (расширения GCC и Clang) меняют, что видит компоновщик, поэтому они относятся к тому, что при отсутствии даёт `#error`. `weak` делает определение или ссылку слабой: слабое определение уступает обычному из другого файла, а слабая неопределённая ссылка равна нулю, и её проверяют перед вызовом. `alias` создаёт второе имя того же определения, это способ сохранить старое имя функции после переименования.

```c
#include <stdio.h>

int log_level(void) { return 2; }
[[gnu::alias("log_level")]] int loglevel_compat(void);
[[gnu::weak]] int feature_enabled(void) { return 0; }
[[gnu::weak]] void optional_hook(void);

int main(void) {
    printf("%d %d %d\n", log_level(), loglevel_compat(), feature_enabled()); /* вывод: 2 2 0 */
    if (optional_hook) optional_hook(); else puts("no hook"); /* вывод: no hook */
    return 0;
}
```

Если другой объектный файл определит `feature_enabled` обычным образом, программа вернёт его значение без правок этого файла: так делают подменяемые заглушки. `alias` требует, чтобы цель была определена в том же файле.

Секции собирают разбросанные по проекту объекты в один массив: `[[gnu::section("имя")]]` кладёт объект в именованную секцию, а компоновщик для имени, являющегося допустимым идентификатором C (например `reg_entries`), определяет границы `__start_reg_entries` и `__stop_reg_entries`. Так устроена статическая регистрация без центрального списка; единственный полный пример приведён в [паттернах](21-patterns.md), а здесь ловушки механизма.

#### Подводные камни секций

- Имя секции с точкой (`.reg.entries`) границ не получает: в проверке `__start_reg_entries` остался неопределённым символом на обоих компиляторах. Используй имя без точки.
- Порядок элементов не определён. GCC 14 при `-O2` в проверке разместил три объекта в обратном порядке (при `-O0` в прямом), Clang в прямом; не строй логику на порядке и не клади в секцию объекты разного типа или выравнивания.
- Сборка с `--gc-sections` удаляет «неиспользуемые» объекты. С GNU ld 2.42 хватило `used` (обращение к `__start_` сохраняет секцию целиком); lld оставил только объект с `retain`, остальные потерялись, а при пустой секции компоновка упала с «undefined symbol: __start_...». Пиши `[[gnu::used, gnu::retain]]` у каждого элемента (макрос по `__has_attribute(retain)`) или передавай `-Wl,-z,nostart-stop-gc`.
- `used` и `retain` у части элементов секции: GCC предупреждает, что объекты с `retain` и без него попали в секцию с одним именем. Атрибуты у всех элементов одинаковые.

`constructor` и `destructor` запускают функцию до `main` и после `exit`. Приоритет в скобках задаёт порядок: меньшие числа выполняются раньше, а без приоритета функции идут после всех приоритетных; приоритеты от 0 до 100 зарезервированы (GCC предупреждает `-Wprio-ctor-dtor`), деструкторы идут в обратном порядке.

```c
#include <stdio.h>
#include <stdlib.h>

static void at_exit(void);

[[gnu::constructor(101)]] static void c101(void) { puts("c101"); /* вывод: c101 */ }
[[gnu::constructor(200)]] static void c200(void) { puts("c200"); /* вывод: c200 */ }
[[gnu::constructor]] static void c_default(void) { puts("c_default"); /* вывод: c_default */ }

int main(void) {
    atexit(at_exit);
    puts("main"); /* вывод: main */
    return 0;
}

static void at_exit(void) { puts("atexit"); /* вывод: atexit */ }
[[gnu::destructor(200)]] static void d200(void) { puts("d200"); /* вывод: d200 */ }
[[gnu::destructor(101)]] static void d101(void) { puts("d101"); /* вывод: d101 */ }
```

Определения здесь записаны в порядке выполнения: приоритетные конструкторы, затем без приоритета, `main`, обработчик `atexit`, деструкторы (в обратном порядке приоритетов); на всех трёх компиляторах порядок оказался таким.

#### Подводные камни конструкторов

- Порядок между файлами. Конструкторы одного приоритета и без приоритета из разных файлов идут в порядке, который определяет компоновщик; зависимость одного от другого задавай приоритетами или явной функцией `init`.
- Статическая библиотека. Объектный файл, на который никто не ссылается, в программу не попадает, и его конструктор не запустится; в проверке при обычной компоновке конструктор из архива не сработал, а с `-Wl,--whole-archive` сработал. Регистрацию через конструктор из библиотеки подключай явной ссылкой или компонуй целиком.
- Работа до `main`. Конструктор не может полагаться на то, что другие модули уже инициализированы; в разделяемой библиотеке он запускается при загрузке (`dlopen`), что полезно для плагинов ([паттерны](21-patterns.md)).

## Диспетчеризация по процессору

### Выбирай реализацию по процессору во время выполнения и держи расширения на отдельной функции

Один бинарник должен работать на любом процессоре семейства, а векторные инструкции есть не везде, поэтому быструю версию собирают с расширением процессора только в отдельной функции и выбирают версию при запуске. Примеры ниже написаны для x86-64 с glibc; различия aarch64 и musl вынесены в подводные камни. Флаг `-march=native` или `-mavx2` на всей единице трансляции опасен: компилятор вставит такие инструкции куда угодно, и программа упадёт с SIGILL на старом процессоре ([кросс-компиляция](22-cross-compilation.md)).

Самый переносимый способ: функция с `[[gnu::target("avx2")]]` и обычный выбор через указатель. Атрибут `target` включает расширение только для этой функции, а интринсики из `<immintrin.h>` (на ARM `<arm_neon.h>`) в ней допустимы. Проверка `__builtin_cpu_supports("avx2")` возвращает ненулевое число, но не обязательно 1 (GCC в проверке вернул 1024, Clang 1): используй как условие.

```c
#include <stddef.h>
#include <stdio.h>
#if defined(__x86_64__)
#include <immintrin.h>
#endif

static int sum_plain(const int *a, size_t n) {
    int s = 0;
    for (size_t i = 0; i < n; i++) s += a[i];
    return s;
}

#if defined(__x86_64__)
[[gnu::target("avx2")]] static int sum_avx2(const int *a, size_t n) {
    __m256i acc = _mm256_setzero_si256();
    size_t i = 0;
    for (; i + 8 <= n; i += 8)
        acc = _mm256_add_epi32(acc, _mm256_loadu_si256((const __m256i *)(const void *)(a + i)));
    int lanes[8];
    _mm256_storeu_si256((__m256i *)(void *)lanes, acc);
    int s = 0;
    for (int k = 0; k < 8; k++) s += lanes[k];
    for (; i < n; i++) s += a[i];
    return s;
}
#endif

int main(void) {
    int a[20];
    for (int i = 0; i < 20; i++) a[i] = i;
    int (*sum)(const int *, size_t) = sum_plain;
#if defined(__x86_64__)
    if (__builtin_cpu_supports("avx2")) sum = sum_avx2;
#endif
    printf("%d\n", sum(a, 20)); /* вывод: 190 */
    return 0;
}
```

Когда выбор нужен без изменения мест вызова, компилятор строит его сам. `target_clones` (в GCC и Clang) создаёт несколько вариантов одной функции и выбирает лучший при загрузке; `ifunc` сопоставляет имя функции с функцией-резолвером, которая возвращает указатель на нужную реализацию.

```c
#include <stdio.h>

[[gnu::target_clones("avx2", "default")]] int sum(const int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += a[i];
    return s;
}

int main(void) {
    int a[4] = {1, 2, 3, 4};
    printf("%d\n", sum(a, 4)); /* вывод: 10 */
    return 0;
}
```

```c
#include <stdio.h>

typedef int (*sum_fn)(const int *, int);

static int sum_plain(const int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += a[i];
    return s;
}

[[gnu::target("avx2")]] static int sum_avx2(const int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += a[i];
    return s;
}

__attribute__((no_sanitize("address", "undefined")))
static sum_fn resolve_sum(void) {
    __builtin_cpu_init();
    return __builtin_cpu_supports("avx2") ? sum_avx2 : sum_plain;
}

int sum(const int *a, int n) __attribute__((ifunc("resolve_sum")));

int main(void) {
    int a[4] = {1, 2, 3, 4};
    printf("%d\n", sum(a, 4)); /* вывод: 10 */
    return 0;
}
```

#### Подводные камни

- Резолвер `ifunc` выполняется при разборе перемещений, до инициализации санитайзеров и libc. На GCC 14 с ASan резолвер без `no_sanitize` вызвал аварийное завершение (SIGSEGV), с атрибутом работает; в резолвере не вызывай `printf` и не трогай глобальные объекты, а `__builtin_cpu_init()` вызывай явно.
- musl не поддерживает `ifunc` и `target_clones`: динамический загрузчик отвечает «unsupported relocation type 37», а статическая сборка через `musl-gcc -static` результата не напечатала и завершилась с ненулевым кодом. Для переносимого бинарника выбирай через указатель и `__builtin_cpu_supports`: на musl (статически) это работало и вернуло ненулевое значение.
- aarch64: `__builtin_cpu_init` отсутствует и в GCC 14 («implicit declaration»), и в Clang 19 («builtin is not supported on this target»), `__builtin_cpu_supports` отсутствует в GCC 14, а Clang 19 принимает с ним имена расширений ARM (`"sve2"`, под qemu вернул ненулевое значение). `target_clones` на aarch64 принимает имена расширений ARM (`"sve2"`, `"dotprod"`) и работал под qemu, а x86-имена вроде `"avx2"` GCC отклоняет («invalid feature modifier»). Переносимо признаки процессора на ARM читают через `getauxval(AT_HWCAP)` и бит, например `HWCAP_ASIMD`: на qemu это дало 1 на обоих компиляторах.
- Вызов функции с `target("avx2")` без проверки процессора это SIGILL на старом железе. Помеченную `always_inline` функцию с `target("avx2")` нельзя подставить в вызывающего без этого расширения: Clang 19 и GCC 14 отвечают ошибкой.
- Проверяется только выбранная версия. Тест диспетчеризации вызывает оба варианта (быстрый и обычный) напрямую на одних данных и сверяет результаты, а не полагается на выбор текущего процессора.

## Коротко

### Диагностика и контракты Clang: diagnose_if, enable_if, musttail

`diagnose_if(условие, "сообщение", "error" или "warning")` проверяет аргументы вызова при компиляции и выдаёт собственную диагностику; условие вычисляется только для постоянных аргументов, как `v > 100` ниже. `enable_if(условие, "сообщение")` выключает функцию, если условие не доказано, и вызов с непостоянным аргументом отвергается («no matching function»). В C без `overloadable` `enable_if` работает; в проверке его пришлось писать на единственном определении функции: когда атрибут стоял на объявлении, а определение шло следом без него, Clang отверг даже допустимый вызов. `[[clang::musttail]]` на `return` требует хвостового вызова: при несовпадении сигнатур вызывающей и вызываемой функций Clang отказывает в компиляции (в проверке «signature is incompatible»). Всё это расширения Clang: GCC 14 их не знает, поэтому оба макроса при отсутствии атрибута пустые (проверка пропадает вместе с диагностикой), а `musttail` заменён обычным вызовом.

```c
/* clang_ext.c */
#include <stdio.h>

#if __has_attribute(diagnose_if)
#define DIAGNOSE_IF(c, msg, kind) __attribute__((diagnose_if(c, msg, kind)))
#else
#define DIAGNOSE_IF(c, msg, kind)
#endif

#if __has_attribute(enable_if)
#define ENABLE_IF(c, msg) __attribute__((enable_if(c, msg)))
#else
#define ENABLE_IF(c, msg)
#endif

static int pct(int v) DIAGNOSE_IF(v > 100, "percent above 100", "error") { return v; }

static int div_exact(int a, int b) ENABLE_IF(b != 0, "divisor must be constant and nonzero") { return a / b; }

static int gcd(int a, int b) {
    if (b == 0) return a;
#if __has_c_attribute(clang::musttail)
    [[clang::musttail]] return gcd(b, a % b);
#else
    return gcd(b, a % b);
#endif
}

int main(void) {
    printf("%d %d %d\n", pct(50), div_exact(10, 2), gcd(12, 18)); /* вывод: 50 5 6 */
    return 0;
}
```

Вызов `pct(150)` на Clang даёт ошибку `percent above 100`, а `div_exact(1, 0)` ошибку «no matching function ... divisor must be constant and nonzero». Аннотации допустимости нулевого указателя `_Nullable` и `_Nonnull` описаны в [API и модулях](07-api-and-modules.md).

### Стандартные атрибуты fallthrough и noreturn

`[[fallthrough]];` перед следующей меткой `case` сообщает, что проход в неё намеренный, и убирает предупреждение `-Wimplicit-fallthrough`; `[[noreturn]]` на функции, которая не возвращает управление (обёртка над `exit`), убирает предупреждение `-Wreturn-type` о конце функции без возврата после её вызова. Оба атрибута стандартные: пиши только `[[...]]`, макрос не нужен.

```c
#include <stdio.h>
#include <stdlib.h>

[[noreturn]] static void fail(void) { exit(1); }

static int weight(int kind) {
    int w = 0;
    switch (kind) {
    case 1: w += 1; [[fallthrough]];
    case 2: w += 2; break;
    default: fail();
    }
    return w;
}

int main(void) {
    printf("%d %d\n", weight(1), weight(2)); /* вывод: 3 2 */
    return 0;
}
```

## Что ещё стоит знать

Встроенные `__builtin_return_address(0)` и `__builtin_frame_address(0)` возвращают адрес возврата и кадр текущей функции и нужны профилировщикам и самодельным трассировщикам стека. Директива `#warning "текст"` входит в C23 и печатает предупреждение при сборке. Атрибут `symver` (GCC; в Clang 19 и 20 `__has_attribute(symver)` равно 0) привязывает символ к версии для библиотек с несколькими версиями интерфейса.