# Тесты и отладка

## Модульные тесты

### Пиши юнит-тесты на cmocka: один тест - одна функция, общее состояние в fixture

cmocka даёт набор `assert_*`, запуск тестов по одному с отчётом и fixtures: `setup` создаёт объект перед каждым тестом, `teardown` освобождает его, а объект передаётся тесту через `void **state`. Тесты ниже проверяют урезанный модуль `buf` из [главы про интерфейсы](06-api-and-modules.md).

```c
/* buf.h */
#ifndef BUF_H
#define BUF_H
#include <stddef.h>

typedef struct buf buf;

buf *buf_create(void);
void buf_destroy(buf *b);
int buf_append(buf *b, const void *data, size_t n);
const char *buf_data(const buf *b);
size_t buf_len(const buf *b);
#endif
```

```c
/* buf.c */
#include "buf.h"
#include <errno.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

struct buf { char *data; size_t len, cap; };

buf *buf_create(void) { return calloc(1, sizeof(buf)); }
void buf_destroy(buf *b) { if (b) free(b->data); free(b); }
const char *buf_data(const buf *b) { return b->data ? b->data : ""; }
size_t buf_len(const buf *b) { return b->len; }

int buf_append(buf *b, const void *data, size_t n) {
    if (n >= SIZE_MAX - b->len) return -ENOMEM;
    size_t need = b->len + n + 1; /* +1 под завершающий NUL */
    if (need > b->cap) {
        size_t cap = b->cap ? b->cap : 16;
        while (cap < need && cap <= SIZE_MAX / 2) cap *= 2;
        char *p = cap < need ? nullptr : realloc(b->data, cap);
        if (!p) return -ENOMEM;
        b->data = p;
        b->cap = cap;
    }
    if (n) memcpy(b->data + b->len, data, n);
    b->len += n;
    b->data[b->len] = '\0';
    return 0;
}
```

```c
/* main.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>
#include "buf.h"

static int setup(void **state) { return (*state = buf_create()) ? 0 : -1; }
static int teardown(void **state) { buf_destroy(*state); return 0; }

static void test_new_buf_is_empty(void **state) {
    buf *b = *state;
    assert_int_equal(buf_len(b), 0);
    assert_string_equal(buf_data(b), "");
}

static void test_append_accumulates_and_grows(void **state) {
    buf *b = *state;
    assert_int_equal(buf_append(b, "ab", 2), 0);
    assert_int_equal(buf_append(b, "cd", 2), 0);
    assert_memory_equal(buf_data(b), "abcd", 4);
    for (int i = 0; i < 100; i++) assert_int_equal(buf_append(b, "x", 1), 0);
    assert_int_equal(buf_len(b), 104); /* больше начальной ёмкости: realloc отработал */
    assert_int_equal(buf_data(b)[103], 'x');
}

#define TEST(f) cmocka_unit_test_setup_teardown(f, setup, teardown)

int main(void) {
    const struct CMUnitTest tests[] = {TEST(test_new_buf_is_empty), TEST(test_append_accumulates_and_grows)};
    return cmocka_run_group_tests(tests, nullptr, nullptr);
}
```

```sh
cc -std=c23 -g buf.c main.c -lcmocka -o test_buf && ./test_buf
```

Тест, упавший на `assert_*`, прерывается сразу (внутри `longjmp`), остальные продолжают работу, процесс завершается ненулевым кодом. Заголовки `<stdarg.h>`, `<stddef.h>` и `<setjmp.h>` подключают до `<cmocka.h>`: сам он их не включает. Тест без состояния всё равно принимает `void **state`; неиспользуемый параметр гасят `(void)state;`.

Для повседневной работы хватает `assert_true`/`assert_false`, `assert_int_equal`, `assert_string_equal`, `assert_memory_equal`, `assert_null`/`assert_non_null`, `assert_ptr_equal`, `assert_in_range`. Подготовку на всю группу задают `group_setup` и `group_teardown` в `cmocka_run_group_tests`; формат вывода `CMOCKA_MESSAGE_OUTPUT` (например `TAP` для CI).

#### Подводные камни

- `setup` вернул ненулевое значение или не записал `*state`. Тест считается упавшим; создавай в `setup` один объект и не оставляй полусозданного состояния.
- Тесты зависят друг от друга через глобальную переменную или файл. Один упавший тест роняет следующие; всё общее создаётся в `setup` и уничтожается в `teardown`.
- `assert_int_equal` сравнивает как `uintmax_t`: отрицательные коды (`-ENOMEM`) в сообщении выглядят как огромные числа (`0xfffffffffffffff4`), смысл остаётся прежним.
- `assert_*` внутри вспомогательной функции указывает на строку в ней; вызывай `assert_*` в самом тесте.
- Один тест проверяет десять вещей подряд. Первое падение скрывает остальные; один тест - одно поведение.

### Собирай тесты с ASan и UBSan и заводи regression-тест на каждый баг

Санитайзеры превращают молчаливую порчу памяти и неопределённое поведение в падение теста на той же строке, поэтому тесты всегда собирают с `-fsanitize=address,undefined` и `-fno-sanitize-recover=all` (без него UBSan лишь печатает диагностику, и тест проходит; подробности о флагах в [главе про сборку](01-toolchain.md)).

```sh
cc -std=c23 -g -O1 -fsanitize=address,undefined -fno-sanitize-recover=all buf.c main.c -lcmocka -o test_buf
```

Утечки ловит LSan, входящий в ASan: он проверяет их при выходе процесса и меняет код возврата.

Каждый найденный дефект получает тест, который падает на старой версии кода и проходит на исправленной; тест называют по номеру задачи и поведению. Ниже два теста для `buf`: передача `nullptr` при нулевой длине и длина, близкая к `SIZE_MAX`.

```c
/* main.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <stdint.h>
#include <cmocka.h>
#include <errno.h>
#include "buf.h"

/* Баг 412: buf_append(b, nullptr, 0) вызывал memcpy(nullptr, nullptr, 0) */
static void regress_412_append_null_with_zero_len(void **state) {
    (void)state;
    buf *b = buf_create();
    assert_non_null(b);
    assert_int_equal(buf_append(b, nullptr, 0), 0);
    assert_int_equal(buf_len(b), 0);
    buf_destroy(b);
}

/* Баг 437: длина около SIZE_MAX переполняла len + n + 1 */
static void regress_437_append_huge_len_fails_cleanly(void **state) {
    (void)state;
    buf *b = buf_create();
    assert_int_equal(buf_append(b, "ab", 2), 0);
    assert_int_equal(buf_append(b, "", SIZE_MAX - 1), -ENOMEM);
    assert_string_equal(buf_data(b), "ab"); /* буфер не изменён */
    buf_destroy(b);
}

int main(void) {
    const struct CMUnitTest tests[] = {cmocka_unit_test(regress_412_append_null_with_zero_len),
                                       cmocka_unit_test(regress_437_append_huge_len_fails_cleanly)};
    return cmocka_run_group_tests(tests, nullptr, nullptr);
}
```

Если убрать из `buf_append` защиту `if (n)` перед `memcpy`, первый тест падает под UBSan, хотя без санитайзера он проходит:

```text
[ RUN      ] regress_412_append_null_with_zero_len
buf.c:25:30: runtime error: null pointer passed as argument 2, which is declared to never be null
/usr/include/string.h:44:28: note: nonnull attribute specified here
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior buf.c:25:30
```

#### Подводные камни

- Тест проходит без санитайзера, а ошибка видна только под ним. Прогон без санитайзеров ничего не доказывает про память; в CI другой конфигурации нет.
- Regression-тест написан после исправления и ни разу не падал. Сначала запусти его на версии с багом и убедись, что он красный.
- ASan не видит выход за границы поля внутри структуры (соседнее поле остаётся допустимой памятью): такие ошибки ловит `-fsanitize=array-bounds` в UBSan или наблюдательная точка в отладчике ([отладка](08-debugging.md)).
- Тестируемый код и тест компилируют с одними и теми же флагами санитайзеров.

## Подмена зависимостей

### Подменяй зависимости таблицей операций: тестовая реализация читает из памяти и подсовывает ошибки

Частичные чтения и ошибки ввода-вывода на реальном сокете или файле не воспроизвести надёжно, а тестовая реализация [таблицы операций](06-api-and-modules.md) выдаёт любую последовательность ответов из памяти. Функция принимает `stream` по значению, и тест подставляет двойник; полный пример внедрения зависимостей в [главе про паттерны](14-patterns.md), здесь только тестовая сторона.

Проверяемая функция `slurp` читает поток до конца в буфер вызывающего и возвращает число байт или `-E...`.

```c
/* stream.h */
#ifndef STREAM_H
#define STREAM_H
#include <stddef.h>

typedef struct ops { int (*read)(void *ctx, void *buf, size_t n); void (*close)(void *ctx); } ops;
typedef struct stream { const ops *vt; void *ctx; } stream;

static inline int stream_read(stream s, void *buf, size_t n) { return s.vt->read(s.ctx, buf, n); }

/* slurp - читает поток до конца в dst (не более cap байт).
 * Возврат: число прочитанных байт, -EFBIG если данные не помещаются, или -E... из read. */
int slurp(stream s, char *dst, size_t cap);
#endif
```

```c
/* slurp.c */
#include "stream.h"
#include <errno.h>

int slurp(stream s, char *dst, size_t cap) {
    size_t total = 0;
    for (;;) {
        if (total == cap) { /* место кончилось: данные закончились тоже? */
            char probe;
            int n = stream_read(s, &probe, 1);
            return n == 0 ? (int)total : n < 0 ? n : -EFBIG;
        }
        int n = stream_read(s, dst + total, cap - total);
        if (n < 0) return n;
        if (n == 0) return (int)total;
        total += (size_t)n;
    }
}
```

Двойник хранит данные, размер порции и ошибку, которую нужно вернуть после исчерпания данных. Он лежит в тестовом коде и не попадает в продукт.

```c
/* fake.h */
#ifndef FAKE_H
#define FAKE_H
#include "stream.h"
#include <string.h>

typedef struct {
    const char *data;
    size_t len, pos;
    size_t chunk;  /* не больше стольких байт за вызов: частичные чтения */
    int fail_with; /* если не 0: после исчерпания данных вернуть эту ошибку */
} fake;

static inline int fake_read(void *ctx, void *buf, size_t n) {
    fake *f = ctx;
    if (f->pos == f->len) return f->fail_with ? f->fail_with : 0;
    size_t k = f->len - f->pos;
    if (k > n) k = n;
    if (k > f->chunk) k = f->chunk;
    memcpy(buf, f->data + f->pos, k);
    f->pos += k;
    return (int)k;
}

static const ops fake_ops = {.read = fake_read};
#endif
```

```c
/* main.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <errno.h>
#include <cmocka.h>
#include "fake.h"

static void test_partial_reads_are_assembled(void **state) {
    (void)state;
    fake f = {.data = "hello world", .len = 11, .chunk = 3};
    char out[16];
    assert_int_equal(slurp((stream){&fake_ops, &f}, out, sizeof out), 11);
    assert_memory_equal(out, "hello world", 11);
}

static void test_read_error_is_propagated(void **state) {
    (void)state;
    fake f = {.data = "abc", .len = 3, .chunk = 3, .fail_with = -EIO};
    char out[16];
    assert_int_equal(slurp((stream){&fake_ops, &f}, out, sizeof out), -EIO);
}

static void test_too_big_input_is_rejected(void **state) {
    (void)state;
    fake f = {.data = "0123456789", .len = 10, .chunk = 4};
    char out[8];
    assert_int_equal(slurp((stream){&fake_ops, &f}, out, sizeof out), -EFBIG);
}

int main(void) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test(test_partial_reads_are_assembled),
        cmocka_unit_test(test_read_error_is_propagated),
        cmocka_unit_test(test_too_big_input_is_rejected),
    };
    return cmocka_run_group_tests(tests, nullptr, nullptr);
}
```

Тесты проверяют то, что в реальном окружении воспроизводится плохо: короткие чтения, ошибку после части данных и вход, ровно не помещающийся в буфер. Тот же двойник годится для любого кода поверх `stream`.

#### Подводные камни

- Двойник умнее реализации и повторяет её ошибки. Делай его глупым: данные из массива, счётчик и код ошибки.
- Двойник всегда отдаёт всё запрошенное. Частичные чтения (`chunk`) и `0` до конца данных проверяют цикл чтения.
- Двойник лежит в одном файле с продуктовым кодом и попадает в библиотеку; держи его в каталоге тестов.

## Моки

### Мокай функции cmocka, только когда подмена через таблицу невозможна или важен порядок вызовов

Если зависимость уже оформлена как таблица операций или аллокатор, хватает подмены через `ops`: двойник просто работает, а не сверяется со сценарием. Мок cmocka нужен для зависимости, объявленной свободной функцией и подставляемой на этапе сборки: тест определяет функцию с тем же именем, и компоновщик берёт её вместо настоящей. Мок заодно проверяет взаимодействие: какие аргументы переданы и в каком порядке шли вызовы.

Системный вызов оборачивают платформенной функцией со схемой ошибок проекта (`-errno`); в тесте её заменяет мок.

```c
/* plat.h */
#ifndef PLAT_H
#define PLAT_H
#include <stddef.h>

/* plat_read - read(2): число байт, 0 на конце данных или -errno. */
int plat_read(int fd, void *buf, size_t n);

/* cfg_load_fd - читает дескриптор до конца в dst (не более cap байт), повторяя чтение при -EINTR.
 * Возврат: число байт или -E... */
int cfg_load_fd(int fd, char *dst, size_t cap);
#endif
```

```c
/* cfg.c */
#include "plat.h"
#include <errno.h>

int cfg_load_fd(int fd, char *dst, size_t cap) {
    size_t total = 0;
    while (total < cap) {
        int n = plat_read(fd, dst + total, cap - total);
        if (n == -EINTR) continue;
        if (n < 0) return n;
        if (n == 0) break;
        total += (size_t)n;
    }
    return (int)total;
}
```

Мок читает ожидания из очередей cmocka. `expect_value(функция, параметр, значение)` задаёт, что `check_expected(параметр)` внутри мока должен увидеть это значение; `will_return(функция, значение)` кладёт в очередь то, что мок получит через `mock_type(тип)` или `mock_ptr_type(тип)`. Каждый `check_expected` берёт одно значение, каждый `mock_type` или `mock_ptr_type` один элемент `will_return`: в примере вызов мока берёт то один элемент, то два.

```c
/* main.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <string.h>
#include <errno.h>
#include <cmocka.h>
#include "plat.h"

int plat_read(int fd, void *buf, size_t n) {
    (void)n;
    check_expected(fd);
    int rc = mock_type(int);
    if (rc > 0) memcpy(buf, mock_ptr_type(const char *), (size_t)rc);
    return rc;
}

static void test_cfg_load_fd_retries_after_eintr(void **state) {
    (void)state;
    for (int i = 0; i < 3; i++) expect_value(plat_read, fd, 7);
    will_return(plat_read, -EINTR);
    will_return(plat_read, 4);
    will_return(plat_read, "abcd");
    will_return(plat_read, 0);
    char out[16];
    assert_int_equal(cfg_load_fd(7, out, sizeof out), 4);
    assert_memory_equal(out, "abcd", 4);
}

int main(void) {
    const struct CMUnitTest tests[] = {cmocka_unit_test(test_cfg_load_fd_retries_after_eintr)};
    return cmocka_run_group_tests(tests, nullptr, nullptr);
}
```

Тест задаёт сценарий: прерывание сигналом, затем четыре байта, затем конец данных, все три вызова с дескриптором 7. Если убрать из `cfg_load_fd` повтор при `-EINTR`, тест падает (`0xfffffffffffffffc != 0x4`); оставшиеся в очередях значения cmocka тоже считает ошибкой, так что лишний или недостающий вызов мока не проходит незамеченным.

Когда вызов `read` стоит в чужом объектном файле или библиотеке и обёртки нет, функции libc подменяют флагом компоновщика `-Wl,--wrap=read` (работает с `ld` и `lld`): неопределённые ссылки на `read` уходят на `__wrap_read`, а настоящая функция доступна как `__real_read`. Вызовы внутри того же объектного файла, где `read` определена, подмена не затрагивает.

```c
#include <errno.h>
#include <sys/types.h>

ssize_t __real_read(int fd, void *buf, size_t n);
int read_fails; /* тест выставляет 1: следующий read вернёт EIO */

ssize_t __wrap_read(int fd, void *buf, size_t n) {
    if (read_fails) { errno = EIO; return -1; }
    return __real_read(fd, buf, n);
}
```

```sh
cc -std=c23 -Wl,--wrap=read prod.c wrap_read.c test.c -o test
```

#### Подводные камни

- Мок фиксирует порядок и число вызовов, то есть реализацию. Рефакторинг без смены поведения (чтение с другим размером порции) ломает тесты; пиши ожидания только там, где порядок часть контракта.
- Мок заменяет функцию целиком: тест линкуется без настоящей `plat_read`, то есть отдельной целью тестов без `plat.c`.

## Инжекция отказов

### Проверяй пути отказа аллокатором, который отказывает на N-м вызове

Ветки `if (!p) return -ENOMEM;` в обычном прогоне не выполняются никогда, поэтому именно в них прячутся утечки и разрушенные инварианты. [Структура аллокатора](06-api-and-modules.md) позволяет подменить `malloc`: аллокатор считает вызовы и возвращает `nullptr` на заданном, а цикл по всем N проходит каждый путь отказа по разу. После каждого прогона проверяют два свойства: утечек нет (число живых блоков равно нулю) и при ошибке объект остался в прежнем состоянии. Правила очистки при ошибках в [главе про ошибки](04-errors-and-cleanup.md).

```c
/* note.h */
#ifndef NOTE_H
#define NOTE_H
#include <stddef.h>

typedef struct allocator {
    void *(*alloc)(void *ctx, size_t size);
    void (*free)(void *ctx, void *p);
    void *ctx;
} allocator;

typedef struct { allocator mem; char *title, *body; } note;

/* note_set - заменяет заголовок и текст копиями аргументов.
 * Возврат: 0 или -ENOMEM; при ошибке прежнее содержимое не изменено. */
int note_set(note *nt, const char *title, const char *body);
void note_clear(note *nt);
#endif
```

```c
/* note.c */
#include "note.h"
#include <errno.h>
#include <string.h>

int note_set(note *nt, const char *title, const char *body) {
    const allocator *a = &nt->mem;
    char *t = a->alloc(a->ctx, strlen(title) + 1);
    char *b = t ? a->alloc(a->ctx, strlen(body) + 1) : nullptr;
    if (!b) {
        a->free(a->ctx, t);
        return -ENOMEM;
    }
    a->free(a->ctx, nt->title);
    a->free(a->ctx, nt->body);
    nt->title = strcpy(t, title);
    nt->body = strcpy(b, body);
    return 0;
}

void note_clear(note *nt) {
    nt->mem.free(nt->mem.ctx, nt->title);
    nt->mem.free(nt->mem.ctx, nt->body);
    nt->title = nt->body = nullptr;
}
```

```c
/* main.c */
#include "note.h"
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct { int calls, fail_at, live; } faulty;

static void *faulty_alloc(void *ctx, size_t n) {
    faulty *f = ctx;
    if (++f->calls == f->fail_at) return nullptr; /* fail_at == 0: не отказывать */
    f->live++;
    return malloc(n);
}

static void faulty_free(void *ctx, void *p) {
    if (p) ((faulty *)ctx)->live--;
    free(p);
}

int main(void) {
    for (int n = 1;; n++) {
        faulty f = {};
        note nt = {.mem = {faulty_alloc, faulty_free, &f}};
        int rc = note_set(&nt, "old", "first");
        assert(rc == 0); /* вызов вне assert: при -DNDEBUG он не пропадёт */
        f.fail_at = f.calls + n; /* отказать на n-м выделении после этой точки */
        rc = note_set(&nt, "new", "second");
        if (rc != 0) assert(!strcmp(nt.title, "old") && !strcmp(nt.body, "first"));
        note_clear(&nt);
        assert(f.live == 0); /* утечка при отказе на n-м выделении */
        printf("n=%d rc=%d\n", n, rc);
        if (rc == 0) break; /* отказ не сработал: все пути пройдены */
    }
}
```

```text
n=1 rc=-12
n=2 rc=-12
n=3 rc=0
```

Цикл заканчивается, когда отказ перестал срабатывать: у `note_set` два выделения, поэтому проверены два пути (`-12` это `-ENOMEM`). Если убрать из `note_set` освобождение `t` в ветке ошибки, проход с `n=2` падает на `assert(f.live == 0)`: счётчик проверяет утечку на каждой итерации внутри одного процесса, а LSan сообщает об утечках только при выходе и суммарно, поэтому не скажет, на каком n они появились. Системные вызовы отказывают так же: `plat_read` или `__wrap_read` со счётчиком возвращает `-EIO` на N-м вызове.

#### Подводные камни

- Тест проверяет только возврат ошибки и не смотрит на состояние. Отказ в середине операции оставляет объект полуобновлённым; проверяй инвариант (прежнее содержимое, согласованные длины) после каждого отказа.
- Цикл остановился раньше, чем кончились пути: например, по первому отказу. Условие выхода должно быть «отказ не сработал» (успех при `fail_at` за последним выделением).
- Код выделяет память мимо переданного аллокатора (прямой `malloc` внутри). Инжекция его не видит; все выделения объекта идут через его аллокатор.
- Отказ срабатывает в самом тесте (подготовка объекта). Включай отказ только после подготовки, как `f.fail_at = f.calls + n` выше.

## Фаззинг

### Фаззи разбор внешних данных через libFuzzer и пиши цель без состояния

Фаззер перебирает входные данные, ведомый покрытием кода, и за секунды находит нечётную длину, граничные символы, цепочки переходов. Подходит любая функция, разбирающая байты из внешнего мира. libFuzzer входит в Clang и запускает `LLVMFuzzerTestOneInput` в одном процессе миллионы раз, а ASan превращает порчу памяти в падение. Ниже цель для разбора шестнадцатеричной записи.

```c
/* hex.h */
#ifndef HEX_H
#define HEX_H
#include <stddef.h>

/* hex_encode - пишет n байт src в dst как C-строку; dst вмещает 2 * n + 1 байт. */
void hex_encode(char *dst, const void *src, size_t n);

/* hex_decode - разбирает строку из n шестнадцатеричных цифр в байты dst (cap байт).
 * Возврат: число байт или -EINVAL (нечётная длина, не цифра); -ENOSPC, если dst мал. */
int hex_decode(void *dst, size_t cap, const char *src, size_t n);
#endif
```

```c
/* hex.c */
#include "hex.h"
#include <errno.h>

void hex_encode(char *dst, const void *src, size_t n) {
    const unsigned char *p = src;
    for (size_t k = 0; k < n; k++) {
        dst[2 * k] = "0123456789abcdef"[p[k] >> 4];
        dst[2 * k + 1] = "0123456789abcdef"[p[k] & 15];
    }
    dst[2 * n] = '\0';
}

static int nibble(char c) {
    if (c >= '0' && c <= '9') return c - '0';
    if (c >= 'a' && c <= 'f') return c - 'a' + 10;
    if (c >= 'A' && c <= 'F') return c - 'A' + 10;
    return -1;
}

int hex_decode(void *dst, size_t cap, const char *src, size_t n) {
    unsigned char *out = dst;
    if (n % 2) return -EINVAL;
    if (n / 2 > cap) return -ENOSPC;
    for (size_t i = 0; i < n; i += 2) {
        int hi = nibble(src[i]), lo = nibble(src[i + 1]);
        if (hi < 0 || lo < 0) return -EINVAL;
        out[i / 2] = (unsigned char)(hi << 4 | lo);
    }
    return (int)(n / 2);
}
```

Обвязка получает указатель и длину, не владеет данными и не хранит состояние между вызовами. Выходной буфер точного размера позволяет ASan видеть каждый байт сверх. Кроме отсутствия падений, цель проверяет свойство: разобранное при кодировании даёт исходный текст (с точностью до регистра цифр).

```c
/* fuzz.c */
#include "hex.h"
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    unsigned char *out = malloc(size / 2 + 1); /* точный размер: ASan видит каждый байт сверх */
    if (!out) abort();
    int n = hex_decode(out, size / 2 + 1, (const char *)data, size);
    if (n >= 0) { /* свойство: кодирование разобранного даёт исходный текст */
        char *back = malloc(2 * (size_t)n + 1);
        if (!back) abort();
        hex_encode(back, out, (size_t)n);
        if ((size_t)n * 2 != size) abort();
        for (size_t i = 0; i < size; i++)
            if ((back[i] | 0x20) != ((char)data[i] | 0x20)) abort();
        free(back);
    }
    free(out);
    return 0;
}
```

```sh
clang -g -O1 -fsanitize=fuzzer,address hex.c fuzz.c -o fz
./fz -runs=200000
```

Флаг `-fsanitize=fuzzer` подключает `main` libFuzzer и счётчики покрытия; для файла со своим `main` при компиляции пишут `-fsanitize=fuzzer-no-link`, при линковке `-fsanitize=fuzzer`. Линковка требует `libstdc++` (в Ubuntu 24.04 пакет `libstdc++-14-dev`); если Clang выбрал каталог GCC без неё, помогает `--gcc-install-dir=` с путём к каталогу версии, где она есть. Если убрать строку `if (n % 2) return -EINVAL;`, декодер читает `src[i + 1]` за концом входа, и первый же запуск находит это на входе из одного байта:

```text
==ERROR: AddressSanitizer: heap-buffer-overflow on address ... 
READ of size 1 at ... thread T0
    #0 ... in hex_decode hex.c:24:46
    #1 ... in LLVMFuzzerTestOneInput fuzz.c:10:13
    ...
... is located 0 bytes after 1-byte region [...)
SUMMARY: AddressSanitizer: heap-buffer-overflow hex.c:24:46 in hex_decode
artifact_prefix='./'; Test unit written to ./crash-adc83b19e793491b1c6ea0fd8b46cd9f32e592fc
Base64: Cg==
```

Файл `crash-...` содержит найденный вход; его воспроизводят запуском `./fz crash-adc83b...` и после исправления добавляют в корпус или превращают в regression-тест. Корпус начинают с образцов настоящих входных данных, а не с пустого каталога; словарь подсказывает фрагменты формата (ключевые слова, магические числа), по одному в строке в кавычках.

```sh
mkdir -p corpus new min
printf 'deadbeef' > corpus/seed1
printf '"dead"\n"beef"\n"cafe"\n' > hex.dict
./fz -max_len=64 -dict=hex.dict -max_total_time=60 new corpus
./fz -merge=1 min new corpus
```

Первый каталог в аргументах получает новые интересные входы, остальные только читаются. `-max_len` ограничивает длину входа (по умолчанию до 4096 байт; для коротких записей малый предел ускоряет поиск), `-max_total_time` и `-runs` ограничивают время или число запусков. `-merge=1` переписывает в первый каталог минимальный набор файлов с тем же покрытием: корпус сохраняют в репозитории и прогоняют в CI командой `./fz -runs=0 corpus` как regression-набор.

#### Подводные камни

- Цель вызывает `exit()`, читает время или случайные числа, пишет в файл или держит глобальное состояние. Падение невоспроизводимо, а фаззер ложно считает вход новым; цель детерминирована, быстра, без побочных эффектов и освобождает выделенное.
- Выходной буфер больше нужного (стек или статический массив). ASan не видит выход за пределы нужного размера; выделяй `malloc` ровно по контракту.
- Цель проверяет только отсутствие падений. Добавь свойство (обратимость, совпадение с эталонной реализацией): так находят логические ошибки.
- Фаззинг запущен без санитайзера. Выход за границы остаётся тихим; сборка всегда `-fsanitize=fuzzer,address`, при необходимости с `undefined`.
- GCC не поддерживает `-fsanitize=fuzzer`; в сборке с GCC цель исключают.

## Разбор падений

### Читай отчёт ASan по частям: что и где повреждено, кто выделил, кто освободил

Отчёт ASan содержит тип ошибки, место доступа и истории выделения и освобождения блока, поэтому отладчик часто не нужен. Сборка с `-g` и `-fno-omit-frame-pointer` ([глава про сборку](01-toolchain.md)) даёт имена функций и строки.

```c
/* Ошибка: UB, чтение после free */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct { char *name; } user;

static user *user_create(const char *name) {
    user *u = malloc(sizeof *u);
    if (!u) abort();
    u->name = strdup(name);
    if (!u->name) abort();
    return u;
}

static void user_destroy(user *u) {
    free(u->name);
    free(u);
}

int main(void) {
    user *u = user_create("alice");
    user_destroy(u);
    printf("%s\n", u->name);
}
```

```text
==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000010 at pc ...
READ of size 8 at 0x502000000010 thread T0
    #0 ... in main uaf.c:24:23
    ...
0x502000000010 is located 0 bytes inside of 8-byte region [0x502000000010,0x502000000018)
freed by thread T0 here:
    #0 ... in free
    #1 ... in user_destroy uaf.c:18:5
    #2 ... in main uaf.c:23:5
    ...
previously allocated by thread T0 here:
    #0 ... in malloc
    #1 ... in user_create uaf.c:9:15
    #2 ... in main uaf.c:22:15
    ...
SUMMARY: AddressSanitizer: heap-use-after-free uaf.c:24:23 in main
```

Адреса, идентификаторы сборки и таблица теневой памяти опущены. Читают отчёт сверху вниз. Первая строка называет класс ошибки (`heap-use-after-free`, `heap-buffer-overflow`, `stack-buffer-overflow`, `double-free`, `attempting free on address which was not malloc()-ed`), вторая: чтение или запись и сколько байт. Первый стек показывает место доступа (`main uaf.c:24`). Строка `is located 0 bytes inside of 8-byte region` описывает блок: смещение внутри него (или `N bytes after` для переполнения). Стек `freed by` показывает, кто освободил блок, стек `previously allocated by` кто выделил: ошибка почти всегда на пересечении трёх стеков, то есть в чьём-то неверном [владении](03-memory-ownership.md). Строка `SUMMARY` повторяет главное, по ней удобно искать в логах CI.

Отчёт без имён функций означает, что нет `-g` или не найден `llvm-symbolizer`; при суффиксе версии у утилиты путь задают явно:

```sh
ASAN_SYMBOLIZER_PATH=/usr/bin/llvm-symbolizer-19 ./prog
ASAN_OPTIONS=abort_on_error=1 ./prog
```

Вторая строка заставляет ASan вызвать `abort()` после отчёта: код возврата становится 134 (SIGABRT) вместо 1, и процесс оставляет дамп памяти или останавливается под отладчиком. В отладчике точка останова на `__asan::ReportGenericError` показывает локальные переменные в кадрах приложения в момент ошибки.

#### Подводные камни

- Отчёт пришёл от доступа, а причина в другом месте. Ошибка владения проявляется в точке использования; смотри стеки `freed by` и `allocated by`, а не только первый.
- Стек обрезан или без строк. Собери с `-g`, `-fno-omit-frame-pointer`, проверь `ASAN_SYMBOLIZER_PATH`; для длинных цепочек поднимай `ASAN_OPTIONS=malloc_context_size=30`.
- Отчёт появился только в оптимизированной сборке. Чаще всего это неопределённое поведение, проявившееся при другом порядке команд, а не «ошибка оптимизатора»; собери ту же версию с `-O1` и `-O2` и сравни.

Причину падения, которое санитайзеры не видят, ищут в отладчике: стек, кадр, значения, наблюдательная точка и дампы описаны в главе [отладка](08-debugging.md).

## Коротко

### Измеряй покрытие кода, чтобы видеть, что тесты не выполняют

Покрытие показывает строки и ветки, которых тесты не коснулись, но не доказывает корректность: выполненная строка не означает проверенную. Ищут непроверенные пути (чаще всего ветки отказа) и добавляют тест или инжекцию отказа. В GCC: `--coverage`, запуск тестов, затем `gcov` той же версии, что компилятор; метка `#####` в `.gcov` означает строку, которую тесты не выполнили.

```sh
gcc --coverage -O0 -g buf.c main.c -lcmocka -o t_gcov && ./t_gcov
gcov t_gcov-buf.gcda
grep -n '#####' buf.c.gcov
```

В Clang то же делают `-fprofile-instr-generate -fcoverage-mapping`: программа пишет профиль (файл задаёт `LLVM_PROFILE_FILE`), `llvm-profdata merge -sparse` сливает его, `llvm-cov show` печатает размеченный исходник (у невыполненных строк счётчик `0`). Собирай покрытие отдельной сборкой с `-O0`: оптимизация переставляет и удаляет строки. Сто процентов требовать бессмысленно, смотрят непокрытые ветки новых изменений.

### Для длительного фаззинга с мутацией на уровне процесса возьми AFL++

AFL++ запускает отдельный процесс на каждый вход и подходит для программ, читающих stdin или файл целиком (`afl-fuzz -i corpus -o out -- ./prog @@` после сборки `afl-cc`); для функций библиотеки удобнее libFuzzer.

## Что ещё стоит знать

Valgrind (`valgrind --leak-check=full ./prog`) находит порчу памяти и утечки без перекомпиляции, но на порядок медленнее санитайзеров; годится для чужих бинарников. Мутационное тестирование (запуск тестов на намеренно испорченной копии кода) проверяет, что сами тесты замечают изменения.
