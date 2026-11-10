# Аргументы командной строки и конфигурация

## Выбор способа разбора

### Бери getopt_long, а библиотеку выбирай по контракту выхода и владения

Парсер аргументов стоит на границе между программой и скриптами, поэтому важнее удобства описания его контракт: завершает ли он процесс сам, кому принадлежат строки, можно ли вызвать его дважды. `getopt_long` входит в glibc и musl и не вызывает `exit`: он возвращает код опции (`?` для ошибки), печатает сообщение в stderr только при ненулевом `opterr`, а решение остаётся за вызывающим. Минусы: состояние глобальное (`optind`, `optarg`, `optopt`), поэтому разбор не потокобезопасен, а таблицу опций и справку приходится строить самому (следующий раздел). На Windows `getopt_long` поставляет mingw-w64 (Clang и GCC под `x86_64-w64-mingw32` собирают и линкуют программу; запуск не проверялся); для других компиляторов под Windows берут библиотеку из таблицы.

| Вариант | Завершает процесс сам | Значения | Состояние | Лицензия, источник |
| --- | --- | --- | --- | --- |
| `getopt_long` | Нет: код `?` или `:`, сообщение при `opterr != 0` | `optarg` указывает в `argv` | Глобальное | Часть libc |
| GOptionContext (GLib) | На `--help` (код 0); ошибка возвращается как `GError` | Копии, освобождать `g_free` | Объект-контекст | LGPL, нужна GLib |
| popt | На `--help` при `POPT_AUTOHELP`; ошибка - отрицательный код `poptGetNextOpt` | Копии, освобождает вызывающий | Объект-контекст | MIT, пакет `libpopt-dev` |
| optparse | Нет: текст ошибки в поле структуры | Указатели в `argv` | Структура вызывающего | Общественное достояние, один заголовок, вне apt |
| cargs | Нет: итератор по аргументам, печать в переданный `FILE` | Указатели в `argv` | Структура вызывающего | MIT, вне apt |
| argparse (cofyc) | Да: на `--help` и на ошибке разбора (флаг `ARGPARSE_IGNORE_UNKNOWN_ARGS` смягчает только неизвестные опции) | Указатели в `argv` | Структура вызывающего | MIT, вне apt |
| argtable3 | Нет: `arg_parse` возвращает число ошибок | Объекты библиотеки, `arg_freetable` | Внутри использует `getopt_long`, наследует глобальное состояние | BSD, вне apt |

Для утилит выбор простой: нужна переносимость на glibc, musl и mingw-w64 и ничего лишнего, берёшь `getopt_long` с таблицей опций; приложение уже на GLib, берёшь GOptionContext. Библиотеки, которые завершают процесс сами, не годятся для кода внутри библиотеки или сервера и мешают тестам: тест разбора завершится вместе с процессом. Когда разбор вызывают из нескольких потоков или вкладывают в библиотеку, нужна структура состояния вместо глобальных переменных: optparse или cargs.

Все строки, которые вернул `getopt_long`, и значения optparse или cargs заимствованы из `argv`, а `argv` живёт до выхода из `main`; GNU-вариант при этом переставляет сам массив. Копировать значения стоит, когда конфигурация переживает разбор (так сделан пример ниже).

## Таблица опций

### Описывай опции таблицей данных, из которой строятся getopt_long, справка и запись в структуру

Массив `option`, строка `optstring` и текст справки расходятся при первой же правке, поэтому опции описывает одна таблица: длинное имя, короткое имя, форма аргумента, поле-назначение и справка. Поле задаёт дескриптор `fdesc` из [сериализации](14-serialization.md) (имя, тип, смещение через `offsetof`), поэтому запись значения в структуру делает один обобщённый код, а не `switch` по опциям. Строки справки помечены `N_()`: перевод при выводе описан в [локализации](18-localization.md).

```c
/* opts.h */
#ifndef OPTS_H
#define OPTS_H
#include <getopt.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>

/* как в главе про сериализацию (копия, чтобы пример собирался сам) */
typedef enum { F_I32, F_U32, F_U8, F_STR } ftype;
typedef struct { const char *name; ftype type; size_t off; } fdesc;
#define TYPE_OF(T, f) _Generic(((T *)0)->f, int32_t: F_I32, uint32_t: F_U32, uint8_t: F_U8, const char *: F_STR)
#define FIELD(T, f) {#f, TYPE_OF(T, f), offsetof(T, f)}

#define N_(s) s /* пометка для xgettext, механизм: [локализация](18-localization.md) */

typedef struct {
    const char *name; /* длинное имя, обязательно */
    int shortc;       /* короткое имя или 0 */
    int arg;          /* no_argument, required_argument, optional_argument */
    fdesc dst;        /* поле-назначение */
    const char *help;
    int cli_only;     /* ключ нельзя задавать из файла и окружения */
} opt;

typedef struct { void *dst; unsigned set; int rest; char err[96]; } opt_run;

/* opt_store - разбирает text и записывает в поле d структуры base; text == nullptr означает флаг (1).
 * Строки копируются, прежнее значение освобождается. Возврат: 0, -EINVAL, -ERANGE, -ENOMEM; текст в err. */
int opt_store(void *base, const fdesc *d, const char *text, char *err, size_t cap);
size_t opt_size(ftype t);
void opts_free(const opt *t, size_t n, void *base); /* освобождает все строковые поля */

/* opts_parse - разбирает argv по таблице t[n] в структуру r->dst; r->set получает бит i для заданной t[i],
 * r->rest - индекс первого позиционного аргумента. Возврат: 0 или -EINVAL (текст в r->err). Не потокобезопасна. */
int opts_parse(const opt *t, size_t n, int argc, char **argv, opt_run *r);
void opts_usage(const opt *t, size_t n, FILE *out);
#endif
```

```c
/* store.c */
#include "opts.h"
#include <errno.h>
#include <stdlib.h>
#include <string.h>

size_t opt_size(ftype t) {
    static const size_t sz[] = {[F_I32] = 4, [F_U32] = 4, [F_U8] = 1, [F_STR] = sizeof(const char *)};
    return sz[t];
}

int opt_store(void *base, const fdesc *d, const char *text, char *err, size_t cap) {
    char *p = (char *)base + d->off;
    if (d->type == F_STR) {
        const char *old, *copy = strdup(text ? text : "");
        if (!copy) return -ENOMEM;
        memcpy(&old, p, sizeof old);
        free((void *)old);
        memcpy(p, &copy, sizeof copy);
        return 0;
    }
    const char *s = text ? text : "1";
    long long lo = d->type == F_I32 ? INT32_MIN : 0;
    long long hi = d->type == F_I32 ? INT32_MAX : d->type == F_U32 ? UINT32_MAX : UINT8_MAX;
    char *end;
    errno = 0;
    long long v = strtoll(s, &end, 10);
    if (end == s || *end) { snprintf(err, cap, "'%s' is not a number (%s)", s, d->name); return -EINVAL; }
    if (errno == ERANGE || v < lo || v > hi) { snprintf(err, cap, "'%s' is out of range (%s)", s, d->name); return -ERANGE; }
    switch (d->type) {
    case F_I32: { int32_t x = (int32_t)v; memcpy(p, &x, sizeof x); break; }
    case F_U32: { uint32_t x = (uint32_t)v; memcpy(p, &x, sizeof x); break; }
    default:    { uint8_t x = (uint8_t)v; memcpy(p, &x, sizeof x); }
    }
    return 0;
}
```

```c
/* opts.c */
#include "opts.h"
#include <errno.h>

#define OPTS_MAX 32

int opts_parse(const opt *t, size_t n, int argc, char **argv, opt_run *r) {
    struct option lo[OPTS_MAX + 1] = {0};
    char sopt[3 * OPTS_MAX + 2] = ":"; /* ведущий ':' - см. нюансы getopt_long */
    size_t k = 1;
    if (n > OPTS_MAX) return -E2BIG;
    for (size_t i = 0; i < n; i++) {
        lo[i] = (struct option){t[i].name, t[i].arg, nullptr, t[i].shortc ? t[i].shortc : 256 + (int)i};
        if (!t[i].shortc) continue;
        sopt[k++] = (char)t[i].shortc;
        for (int a = 0; a < t[i].arg; a++) sopt[k++] = ':'; /* 1 - ":", 2 - "::" */
    }
    opterr = 0; optind = 0; optopt = 0;
    for (int c; (c = getopt_long(argc, argv, sopt, lo, nullptr)) != -1;) {
        int bad = c == '?' || c == ':', key = bad ? optopt : c;
        size_t i = 0;
        while (i < n && lo[i].val != key) i++;
        if (bad) {
            if (c == ':') snprintf(r->err, sizeof r->err, "option '--%s' needs a value", t[i].name);
            else if (i < n) snprintf(r->err, sizeof r->err, "option '--%s' takes no value", t[i].name);
            else if (optopt) snprintf(r->err, sizeof r->err, "unknown option '-%c'", optopt);
            else snprintf(r->err, sizeof r->err, "unknown or ambiguous option '%s'", argv[optind - 1]);
            return -EINVAL;
        }
        int rc = opt_store(r->dst, &t[i].dst, optarg, r->err, sizeof r->err);
        if (rc < 0) return rc;
        r->set |= 1u << i;
    }
    r->rest = optind;
    return 0;
}
```

```c
/* opts_misc.c */
#include "opts.h"
#include <stdlib.h>
#include <string.h>

void opts_free(const opt *t, size_t n, void *base) {
    for (size_t i = 0; i < n; i++) {
        if (t[i].dst.type != F_STR) continue;
        char *p = (char *)base + t[i].dst.off;
        const char *s;
        memcpy(&s, p, sizeof s);
        free((void *)s);
        s = nullptr;
        memcpy(p, &s, sizeof s);
    }
}

void opts_usage(const opt *t, size_t n, FILE *out) {
    for (size_t i = 0; i < n; i++) {
        int w = t[i].shortc ? fprintf(out, "  -%c, ", t[i].shortc) : fprintf(out, "      ");
        w += fprintf(out, "--%s%s", t[i].name, t[i].arg == required_argument ? "=ARG" : t[i].arg == optional_argument ? "[=ARG]" : "");
        fprintf(out, "%*s%s\n", w < 26 ? 26 - w : 1, "", t[i].help);
    }
}
```

`opts_parse` строит `struct option[]` и `optstring` из таблицы: число двоеточий после буквы равно форме аргумента (одно для обязательного, два для необязательного), а ведущий `:` отличает пропущенный аргумент от неизвестной опции (подробнее в следующем разделе). `opterr = 0` выключает печать самого `getopt_long`, потому что она начинается с `argv[0]`; текст ошибки формирует `opts_parse`. Значение разбирает `opt_store`: `strtoll` с `endptr` (весь текст обязан быть числом, пустая строка и хвост вроде `80x` отвергаются), проверка диапазона по типу поля и `ERANGE`; строки копируются, поэтому структура не зависит от `argv`. Опция без аргумента записывает 1 в поле. Маска `r->set` фиксирует, какие поля заданы: она понадобится слоям конфигурации.

Структура и таблица лежат в общем заголовке: на них опираются разбор, проверка и загрузка из файла ниже.

```c
/* cfg.h */
#ifndef CFG_H
#define CFG_H
#include "opts.h"

typedef struct {
    const char *config, *host; /* строки принадлежат cfg: освобождать opts_free */
    uint32_t port;
    int32_t retries;
    uint8_t verbose, quiet, help;
} cfg;

static const opt cfg_opts[] = {
    {"config",  'c', required_argument, FIELD(cfg, config),  N_("config file"), 1},
    {"host",    'H', required_argument, FIELD(cfg, host),    N_("server name (required)"), 0},
    {"port",    'p', required_argument, FIELD(cfg, port),    N_("TCP port, 1..65535"), 0},
    {"retries", 'r', required_argument, FIELD(cfg, retries), N_("attempts, -1 for no limit"), 0},
    {"verbose", 'v', no_argument,       FIELD(cfg, verbose), N_("more output"), 0},
    {"quiet",   'q', no_argument,       FIELD(cfg, quiet),   N_("no output"), 0},
    {"help",    'h', no_argument,       FIELD(cfg, help),    N_("show this help"), 1},
};
#define CFG_NOPTS (sizeof cfg_opts / sizeof cfg_opts[0])
#endif
```

Разбор оформлен функцией `parse` без `exit`: она возвращает код и текст ошибки, а `main` подаёт синтетический `argv` и печатает результат.

```c
/* main.c */
#include "cfg.h"
#include <stdio.h>

#define ARGC(a) ((int)(sizeof a / sizeof a[0]))

static int parse(int argc, char **argv, cfg *c, char *msg, size_t cap) {
    opt_run r = {.dst = c};
    int rc = opts_parse(cfg_opts, CFG_NOPTS, argc, argv, &r);
    snprintf(msg, cap, "%s", rc < 0 ? r.err : argv[r.rest < argc ? r.rest : 0]);
    return rc;
}

static void run(int argc, char **argv) {
    cfg c = {.port = 80, .retries = 3};
    char msg[96];
    if (parse(argc, argv, &c, msg, sizeof msg) < 0) printf("error: %s\n", msg);
    else printf("host=%s port=%u retries=%d verbose=%u file=%s\n", c.host, c.port, c.retries, c.verbose, msg);
    opts_free(cfg_opts, CFG_NOPTS, &c);
}

int main(void) {
    char *a1[] = {"tool", "-v", "-r", "-1", "--port=8080", "-H", "example.org", "data.txt"};
    char *a2[] = {"tool", "--port=80x"};
    char *a3[] = {"tool", "-p", "70000000000"};
    char *a4[] = {"tool", "--verbose=1"};
    char *a5[] = {"tool", "-H"};
    char *a6[] = {"tool", "--bogus"};
    run(ARGC(a1), a1); /* вывод: host=example.org port=8080 retries=-1 verbose=1 file=data.txt */
    run(ARGC(a2), a2); /* вывод: error: '80x' is not a number (port) */
    run(ARGC(a3), a3); /* вывод: error: '70000000000' is out of range (port) */
    run(ARGC(a4), a4); /* вывод: error: option '--verbose' takes no value */
    run(ARGC(a5), a5); /* вывод: error: option '--host' needs a value */
    run(ARGC(a6), a6); /* вывод: error: unknown or ambiguous option '--bogus' */
    return 0;
}
```

#### Подводные камни

- Тип поля вне списка `_Generic`. Поле `long`, `size_t`, `bool` или `uint16_t` в `FIELD` не компилируется: в таблицу идут только `int32_t`, `uint32_t`, `uint8_t` и `const char *`; новый тип добавляют в `ftype`, `TYPE_OF` и `opt_store` вместе.
- Литерал в строковом поле. Первая запись из `opt_store` освободит прежнее значение через `free`, и литерал по умолчанию, лежащий в поле, приведёт к аварии: строковые поля стартуют с `nullptr`, умолчания задаёт слой (раздел про конфигурацию).
- Повтор опции. Побеждает последнее значение, прежняя копия строки освобождается; флаги-счётчики (`-vv`) требуют отдельного типа поля.
- Предел таблицы. `opts_parse` держит массивы на стеке и принимает до 32 опций (маска `set` - один `unsigned`), больше возвращает `-E2BIG`.
- Таблица `static const` в заголовке. Каждая единица трансляции получает свою копию; для крупных таблиц выноси её в один `.c` и объявляй `extern`.

## Нюансы getopt_long

### Знай, чем getopt_long отличается от ожиданий

Поведение определяют мелочи, которых не видно в одном вызове, поэтому каждое правило ниже подтверждено запуском на glibc 2.39. Функция `trace` печатает коды возврата, значения `optarg` и остаток `argv` после `|`; `opterr = 0`, потому что сообщение `getopt_long` начинается с `argv[0]`, а в выводе примера его быть не должно.

```c
/* trace.h */
#define _GNU_SOURCE
#include <getopt.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct { const char *sopt; int posix; char *argv[8]; } tcase;

static const struct option lo[] = {
    {"opt", optional_argument, nullptr, 'o'}, {"num", required_argument, nullptr, 'n'},
    {"verbose", no_argument, nullptr, 'v'}, {"version", no_argument, nullptr, 'V'}, {0}};

/* trace: один разбор; печатает коды getopt_long, затем остаток argv после "|" */
static void trace(tcase *t) {
    int argc = 0;
    while (t->argv[argc]) argc++;
    if (t->posix) setenv("POSIXLY_CORRECT", "1", 1); else unsetenv("POSIXLY_CORRECT");
    opterr = 0; optind = 0; optopt = 0;
    for (int c; (c = getopt_long(argc, t->argv, t->sopt, lo, nullptr)) != -1;) {
        printf("%c", c);
        if (optarg) printf("=%s", optarg);
        if (c == '?' || c == ':') printf("(%c)", optopt ? optopt : '-');
        printf(" ");
    }
    printf("|");
    for (int i = optind; i < argc; i++) printf(" %s", t->argv[i]);
    printf("\n");
}
```

Строка таблицы содержит `optstring`, признак `POSIXLY_CORRECT` и аргументы; результат каждого разбора аннотирован на своей строке.

```c
/* main.c */
#include "trace.h"

int main(void) {
    tcase cases[] = {
        {"vn:",  0, {"p", "a", "-v", "b", nullptr}},    /* вывод: v | a b */
        {"+vn:", 0, {"p", "a", "-v", "b", nullptr}},    /* вывод: | a -v b */
        {"vn:",  1, {"p", "a", "-v", "b", nullptr}},    /* вывод: | a -v b */
        {"vn:",  0, {"p", "-v", "--", "-v", nullptr}},  /* вывод: v | -v */
        {"vn:",  0, {"p", "-n", nullptr}},              /* вывод: ?(n) | */
        {":vn:", 0, {"p", "-n", nullptr}},              /* вывод: :(n) | */
        {":vn:", 0, {"p", "-x", nullptr}},              /* вывод: ?(x) | */
    };
    for (size_t i = 0; i < sizeof cases / sizeof cases[0]; i++) trace(&cases[i]);
    return 0;
}
```

Без `+` GNU-реализация переставляет аргументы: опция после позиционного аргумента всё равно разбирается, а остаток (`a b`) собирается в конце (первая строка). Префикс `+` в `optstring` или переменная окружения `POSIXLY_CORRECT` останавливают разбор на первом не-опционном слове (вторая и третья строки); `--` завершает разбор явно (четвёртая). Пропущенный аргумент без ведущего `:` даёт `?`, неотличимый от неизвестной опции; с ведущим `:` он даёт `:`, неизвестная опция остаётся `?`, а `optopt` в обоих случаях хранит букву опции (три последние строки).

Вторая серия проверяет формы значения.

```c
/* main.c */
#include "trace.h"

int main(void) {
    tcase cases[] = {
        {"o::", 0, {"p", "-ox", nullptr}},              /* вывод: o=x | */
        {"o::", 0, {"p", "-o", "x", nullptr}},          /* вывод: o | x */
        {"o::", 0, {"p", "--opt=x", nullptr}},          /* вывод: o=x | */
        {"o::", 0, {"p", "--opt", "x", nullptr}},       /* вывод: o | x */
        {"n:",  0, {"p", "-n", "-5", nullptr}},         /* вывод: n=-5 | */
        {"n:v", 0, {"p", "-n", "-v", nullptr}},         /* вывод: n=-v | */
        {"",    0, {"p", "--verb", nullptr}},           /* вывод: v | */
        {"",    0, {"p", "--ver", nullptr}},            /* вывод: ?(-) | */
        {"",    0, {"p", "--verbose=1", nullptr}},      /* вывод: ?(v) | */
    };
    for (size_t i = 0; i < sizeof cases / sizeof cases[0]; i++) trace(&cases[i]);
    return 0;
}
```

Необязательный аргумент принимается только слитно: `-ox` и `--opt=x`; формы `-o x` и `--opt x` оставляют `x` позиционным аргументом. Обязательный аргумент забирает следующее слово как есть, даже если оно начинается с `-`: `-n -5` даёт значение `-5` (поэтому отрицательные числа в таблице опций работают), но пропущенное значение в `-n -v` молча съест `-v`. Длинную опцию можно сократить до однозначного префикса (`--verb`); при двух совпадениях (`--ver` подходит и под `verbose`, и под `version`) возвращается `?` с `optopt == 0`, а `--verbose=1` для опции без аргумента тоже даёт `?`, но с `optopt`, равным коду опции: по нему `opts_parse` различает эти случаи.

Состояние глобальное, поэтому второй разбор в том же процессе (подкоманды, тесты, перечитывание конфигурации) начинают с `optind = 0`: это расширение glibc, которое сбрасывает всё внутреннее состояние, в том числе позицию в кластере `-vx`. Присваивание `optind = 1` её не сбрасывает: если первый разбор остановился внутри `-vx` после `v`, то второй разбор по `{"p", "-v"}` с `optind = 1` вернул `x` вместо `v` (проверено на glibc). `optopt` между разборами не обнуляется, его сбрасывают вручную.

#### Подводные камни

- Печать getopt при ненулевом `opterr`. Сообщение вида `./prog: invalid option -- 'x'` начинается с `argv[0]` и идёт в stderr; для своего текста и единообразных ошибок ставь `opterr = 0` и разбирай `?` и `:` сам.
- `POSIXLY_CORRECT` в окружении пользователя. Он меняет порядок разбора; если логика программы зависит от перестановки или от её отсутствия, ставь `+` явно. musl переменную не читает и переставляет всегда (проверено на `musl-gcc`: третья строка первого примера там даёт `v | a b`), поэтому для одинакового поведения на glibc и musl полагайся только на `+`.
- Перестановка меняет `argv`. GNU-реализация переставляет сам массив на месте: указатели на элементы, сохранённые до разбора, после него указывают на другие слова.
- Двусмысленность префикса. Сокращённые длинные опции удобны в интерактивной работе, но в скриптах ломаются при появлении новой опции с тем же началом; в скриптах пиши полные имена.
- Значение, похожее на опцию. Обязательный аргумент забирает следующее слово без разбора, поэтому `-n -v` не ошибка разбора, а значение `-v`; проверяй само значение после разбора, а не полагайся на форму.

## Валидация и вывод

### Проверяй ввод целиком до начала работы и различай коды выхода

Утилиту вызывают скрипты, поэтому её контракт состоит из кодов выхода и разделения потоков: результат и справка по запросу `--help` идут в stdout, диагностика и подсказка при ошибке в stderr. Книга принимает соглашение: 0 успех, 1 сбой при выполнении (файл не открылся), 2 ошибка использования (неверная опция, значение или сочетание опций); так на неверную опцию отвечают `ls` и `grep`. Альтернатива, когда нужен более подробный перечень, - коды `EX_USAGE` (64) и соседние из `sysexits.h`.

Значения проверяет `opt_store`, а условия между полями проверяет `cfg_check`: обязательное поле, диапазон, несовместимые опции. Проверка работает с готовой структурой, поэтому та же функция служит загрузке из слоёв ниже. `--help` обрабатывается до проверки обязательных полей, иначе справку нельзя получить без остальных опций.

```c
/* check.h */
#ifndef CHECK_H
#define CHECK_H
#include "cfg.h"

/* cfg_check - проверяет конфигурацию целиком: обязательные поля, диапазоны, несовместимые сочетания.
 * Возврат: 0 или -EINVAL, текст первой найденной ошибки в err. */
int cfg_check(const cfg *c, char *err, size_t cap);
#endif
```

```c
/* check.c */
#include "check.h"
#include <errno.h>
#include <stdio.h>

int cfg_check(const cfg *c, char *err, size_t cap) {
    if (!c->host || !*c->host) { snprintf(err, cap, "--host is required"); return -EINVAL; }
    if (c->port < 1 || c->port > 65535) { snprintf(err, cap, "port must be in 1..65535"); return -EINVAL; }
    if (c->verbose && c->quiet) { snprintf(err, cap, "--verbose and --quiet exclude each other"); return -EINVAL; }
    return 0;
}
```

Утилита целиком: ошибки использования дают 2 и пишутся в stderr вместе с подсказкой, сбой при выполнении даёт 1, `--help` и результат идут в stdout с кодом 0.

```c
/* tool.c */
#include "check.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv) {
    cfg c = {.port = 80, .retries = 3};
    opt_run r = {.dst = &c};
    char err[96] = "";
    int code = 0;
    if (opts_parse(cfg_opts, CFG_NOPTS, argc, argv, &r) < 0) {
        snprintf(err, sizeof err, "%s", r.err);
        code = 2;
    } else if (c.help) {
        printf("usage: tool [options] FILE\n");
        opts_usage(cfg_opts, CFG_NOPTS, stdout);
    } else if (cfg_check(&c, err, sizeof err) < 0 || r.rest >= argc) {
        if (!*err) snprintf(err, sizeof err, "FILE is required");
        code = 2;
    } else {
        FILE *f = fopen(argv[r.rest], "rb");
        if (c.verbose) fprintf(stderr, "tool: reading %s\n", argv[r.rest]);
        if (!f) { snprintf(err, sizeof err, "cannot open %s: %s", argv[r.rest], strerror(errno)); code = 1; }
        else {
            size_t n = 0, k;
            char buf[512];
            while ((k = fread(buf, 1, sizeof buf, f)) > 0) n += k;
            fclose(f);
            if (!c.quiet) printf("%s:%u %s %zu bytes\n", c.host, c.port, argv[r.rest], n);
        }
    }
    if (code) fprintf(stderr, "tool: %s%s\n", err, code == 2 ? "\ntry 'tool --help'" : "");
    opts_free(cfg_opts, CFG_NOPTS, &c);
    return code;
}
```

```text
$ tool -H example.org -p 8080 data.txt
example.org:8080 data.txt 12 bytes
$ tool -H example.org missing.txt; echo $?
tool: cannot open missing.txt: No such file or directory
1
$ tool --port=80x data.txt; echo $?
tool: '80x' is not a number (port)
try 'tool --help'
2
```

Справка по `tool --help` целиком выглядит так.

```text
# file: help.txt
usage: tool [options] FILE
  -c, --config=ARG        config file
  -H, --host=ARG          server name (required)
  -p, --port=ARG          TCP port, 1..65535
  -r, --retries=ARG       attempts, -1 for no limit
  -v, --verbose           more output
  -q, --quiet             no output
  -h, --help              show this help
```

`argv` - недоверенный ввод: имена и значения не обязаны быть UTF-8, `argv[0]` может быть любым (при `execve` с пустым массивом Linux 6.8 передаёт `argc == 1` и пустую строку в `argv[0]`, поэтому не разыменовывай `argv[0]` без проверки на пустоту, а имя программы для сообщений бери из константы). Командную строку любого процесса читает любой пользователь системы через `/proc/PID/cmdline` (права 0444 при обычном монтировании `/proc`, окружение `environ` - 0400), поэтому пароли и токены в аргументах не передают: источник секрета - файл с жёсткими правами или окружение, подробности в [безопасности](19-security.md).

#### Подводные камни

- `atoi` и `strtol` без `endptr`. `80x` молча становится 80, переполнение не сообщается; разбирай `strtoll` с `endptr`, `errno` и диапазоном (как в `opt_store`), правила числовых преобразований в [строках](06-strings-and-buffers.md).
- Ведущие пробелы и знак. `strtoll` принимает ` 5` и `+5`; если формат строже, проверь первый символ до вызова.
- Диагностика в stdout. Она портит конвейер `tool | next`: stdout только для результата, всё остальное в stderr; при ошибке выводят одну строку и подсказку, а не всю справку.
- `exit` внутри разбора. Такая функция не проверяется тестом и не годится для библиотеки; `parse` возвращает код, решение о выходе принимает `main`.
- Несколько ошибок в одном вводе. Валидация, которая останавливается на первой, заставляет пользователя исправлять по одной; для небольшого числа условий достаточно сообщения о первой, для сложных набирай список ошибок.

## Подкоманды

### Разбирай общие опции до имени подкоманды, а хвост отдавай ей второй серией

Общие опции стоят до имени подкоманды, остальное принадлежит ей, поэтому первый разбор обязан остановиться на первом слове, которое не опция: это делает префикс `+` в `optstring`. Без него GNU переставит аргументы, и первый разбор заберёт `-f`, принадлежащий подкоманде, как неизвестную общую опцию. Подкоманда получает `argv + optind` (её имя становится `argv[0]`) и запускает вторую серию `getopt_long` с `optind = 0`. Таблица `command` с указателями на функции строится так же, как диспетчеризация через таблицу указателей в [паттернах](21-patterns.md).

```c
#include <getopt.h>
#include <stdio.h>
#include <string.h>
typedef struct { const char *name; int (*run)(int argc, char **argv, int verbose); } command;

static int cmd_add(int argc, char **argv, int verbose) {
    static const struct option lo[] = {{"force", no_argument, nullptr, 'f'}, {0}};
    int force = 0;
    optind = 0; /* вторая серия: без сброса разбор продолжился бы с места первой */
    for (int c; (c = getopt_long(argc, argv, ":f", lo, nullptr)) != -1;) {
        if (c != 'f') return 2;
        force = 1;
    }
    printf("add verbose=%d force=%d file=%s\n", verbose, force, optind < argc ? argv[optind] : "-");
    return 0;
}
static const command cmds[] = {{"add", cmd_add}};
static int dispatch(const char *sopt, int argc, char **argv) {
    static const struct option lo[] = {{"verbose", no_argument, nullptr, 'v'}, {0}};
    int verbose = 0;
    opterr = 0; optind = 0; optopt = 0;
    for (int c; (c = getopt_long(argc, argv, sopt, lo, nullptr)) != -1;) {
        if (c != 'v') { printf("global: bad option -%c\n", optopt); return 2; }
        verbose = 1;
    }
    for (size_t i = 0; optind < argc && i < sizeof cmds / sizeof cmds[0]; i++)
        if (!strcmp(argv[optind], cmds[i].name)) return cmds[i].run(argc - optind, argv + optind, verbose);
    printf("unknown command\n");
    return 2;
}

int main(void) {
    char *a1[] = {"tool", "-v", "add", "-f", "a.txt", nullptr};
    char *a2[] = {"tool", "-v", "add", "-f", "a.txt", nullptr};
    char *a3[] = {"tool", "-v", "del", nullptr};
    dispatch("+:v", 5, a1); /* вывод: add verbose=1 force=1 file=a.txt */
    dispatch(":v", 5, a2);  /* вывод: global: bad option -f */
    dispatch("+:v", 3, a3); /* вывод: unknown command */
    return 0;
}
```

Вторая строка вывода показывает ошибку без `+`: общий разбор забрал `-f`. `opterr` и `optopt` общие для обеих серий, поэтому подкоманда может положиться на `opterr = 0`, выставленный диспетчером, но надёжнее выставлять его в начале каждой серии. Эта идиома проверена на glibc и на musl.

## Конфигурация

### Складывай конфигурацию слоями с признаком «задано» по каждому полю

Значение поля берётся из самого приоритетного слоя, где оно задано: умолчания, затем файл, затем окружение, затем аргументы. Если слой хранит только значения, «не указано» неотличимо от «указан ноль», и слияние затирает файл умолчаниями; поэтому слой несёт маску `set` (бит i означает, что задано поле `cfg_opts[i]`), а `cfg_apply` копирует только заданные поля. Аргументы разбираются первыми: из них известен путь `--config`, по нему читается файл, и приоритет не нарушается. Слои собирает `cfg_load`, вызывающий получает либо проверенную целиком конфигурацию, либо ошибку; частично заполненной структуры наружу нет.

```c
/* layers.h */
#ifndef LAYERS_H
#define LAYERS_H
#include "check.h"

typedef struct { cfg v; unsigned set; } cfg_layer; /* бит i: задано поле cfg_opts[i] */

/* cfg_layer_set - разбирает val и кладёт в поле с именем key, помечая его заданным.
 * Возврат: 0, -ENOENT (нет такого поля), -EPERM (ключ только для командной строки), -EINVAL, -ERANGE, -ENOMEM. */
int cfg_layer_set(cfg_layer *l, const char *key, const char *val, char *err, size_t cap);
/* cfg_layer_env - читает переменные PREFIX_ИМЯПОЛЯ; значения копируются. */
int cfg_layer_env(cfg_layer *l, const char *prefix, char *err, size_t cap);
int cfg_layer_file(cfg_layer *l, const char *path, char *err, size_t cap);
/* cfg_apply - накладывает на dst только заданные поля слоя; строки копируются. */
int cfg_apply(cfg *dst, const cfg_layer *l);
/* cfg_load - умолчания < файл < окружение < аргументы; результат проверен целиком. */
int cfg_load(cfg *out, int argc, char **argv, char *err, size_t cap);
#endif
```

```c
/* layers.c */
#include "layers.h"
#include <ctype.h>
#include <errno.h>
#include <stdlib.h>
#include <string.h>

int cfg_layer_set(cfg_layer *l, const char *key, const char *val, char *err, size_t cap) {
    for (size_t i = 0; i < CFG_NOPTS; i++) {
        if (strcmp(cfg_opts[i].dst.name, key)) continue;
        if (cfg_opts[i].cli_only) {
            snprintf(err, cap, "'%s' is allowed only on the command line", key);
            return -EPERM;
        }
        int rc = opt_store(&l->v, &cfg_opts[i].dst, val, err, cap);
        if (rc == 0) l->set |= 1u << i;
        return rc;
    }
    snprintf(err, cap, "unknown key '%s'", key);
    return -ENOENT;
}

int cfg_layer_env(cfg_layer *l, const char *prefix, char *err, size_t cap) {
    for (size_t i = 0; i < CFG_NOPTS; i++) {
        char name[64];
        if (cfg_opts[i].cli_only) continue;
        snprintf(name, sizeof name, "%s_%s", prefix, cfg_opts[i].dst.name);
        for (char *p = name; *p; p++) *p = (char)toupper((unsigned char)*p);
        const char *v = getenv(name);
        int rc = v ? cfg_layer_set(l, cfg_opts[i].dst.name, v, err, cap) : 0;
        if (rc < 0) return rc;
    }
    return 0;
}
```

```c
/* apply.c */
#include "layers.h"
#include <string.h>

int cfg_apply(cfg *dst, const cfg_layer *l) {
    for (size_t i = 0; i < CFG_NOPTS; i++) {
        const fdesc *d = &cfg_opts[i].dst;
        const char *from = (const char *)&l->v + d->off;
        if (!(l->set >> i & 1u)) continue;
        if (d->type != F_STR) {
            memcpy((char *)dst + d->off, from, opt_size(d->type));
            continue;
        }
        const char *s;
        memcpy(&s, from, sizeof s);
        int rc = opt_store(dst, d, s, nullptr, 0); /* копия строки: слой освободится раньше итога */
        if (rc < 0) return rc;
    }
    return 0;
}
```

```c
/* cfg_load.c */
#include "layers.h"

int cfg_load(cfg *out, int argc, char **argv, char *err, size_t cap) {
    cfg_layer def = {0}, file = {0}, env = {0}, args = {0};
    opt_run r = {.dst = &args.v};
    cfg res = {0};
    int rc;
    if ((rc = cfg_layer_set(&def, "port", "80", err, cap)) < 0) goto out;
    if ((rc = cfg_layer_set(&def, "retries", "3", err, cap)) < 0) goto out;
    if ((rc = opts_parse(cfg_opts, CFG_NOPTS, argc, argv, &r)) < 0) { snprintf(err, cap, "%s", r.err); goto out; }
    args.set = r.set;
    /* путь из --config известен до чтения файла, поэтому не мешает приоритету */
    if (args.v.config && (rc = cfg_layer_file(&file, args.v.config, err, cap)) < 0) goto out;
    if ((rc = cfg_layer_env(&env, "APP", err, cap)) < 0) goto out;
    if ((rc = cfg_apply(&res, &def)) < 0 || (rc = cfg_apply(&res, &file)) < 0) goto out;
    if ((rc = cfg_apply(&res, &env)) < 0 || (rc = cfg_apply(&res, &args)) < 0) goto out;
    rc = res.help ? 0 : cfg_check(&res, err, cap);
out:
    opts_free(cfg_opts, CFG_NOPTS, &def.v); opts_free(cfg_opts, CFG_NOPTS, &file.v);
    opts_free(cfg_opts, CFG_NOPTS, &env.v); opts_free(cfg_opts, CFG_NOPTS, &args.v);
    if (rc < 0) opts_free(cfg_opts, CFG_NOPTS, &res); else *out = res;
    return rc;
}
```

Строки принадлежат тому, кто их записал: каждый слой и итог владеют своими копиями (`opt_store` копирует), поэтому слои освобождаются сразу после слияния, а итоговую структуру освобождает `opts_free`. Ключи `config` и `help` помечены в таблице как `cli_only`: файл и окружение не могут их задать (`cfg_layer_set` отвечает `-EPERM`, слой окружения такие поля пропускает), иначе строка `help = 1` в файле превратила бы запуск в вывод справки. Неизвестный ключ в файле считается ошибкой: опечатка не должна молча оставлять умолчание. Окружение - недоверенный ввод, а указатель из `getenv` недолговечен: слой копирует значение сразу и проверяет его тем же `opt_store`; правила для окружения описаны в [безопасности](19-security.md) и [системном уровне](13-system-io.md).

#### Подводные камни

- Значение по умолчанию как литерал в строковом поле. Слияние освободит его через `free`; строковые умолчания кладут в слой умолчаний, как `cfg_load` делает для чисел.
- Приоритет «файл ниже окружения» - соглашение, а не закон. Выбери порядок один раз, запиши в справке и не меняй: смена порядка меняет поведение у всех, кто настроил программу через окружение.
- Слой без маски. Явный ноль из аргументов неотличим от «не указано»: слияние либо затирает файл нулём, либо игнорирует явный ноль; маска нужна всегда, когда ноль допустим.
- Путь к файлу из самого файла. Ключ `config` внутри файла не имеет смысла: файл уже выбран; поэтому такие ключи `cli_only`.
- Секреты в файле. Права 0600 и владелец процесса; значение пароля не попадает в сообщение об ошибке и в лог ([логирование](16-logging.md)).

### Читай файл готовой библиотекой и перечитывай его на новой структуре

Формат файла проще всего взять готовый: inih (лицензия BSD, пакет `libinih-dev`) разбирает `ключ = значение` с секциями и комментариями и вызывает обработчик для каждой пары; своя читалка той же формы занимает около двадцати строк, но берёт на себя пробелы, комментарии, кавычки и сообщения об ошибках. TOML кратко упомянут в [сериализации](14-serialization.md); здесь только чтение конфигурации. Значения в обработчик приходят указателями, живущими до возврата из него, поэтому обработчик копирует их (`cfg_layer_set` через `opt_store` так и делает).

```c
/* cfg_ini.c */
#include "layers.h"
#include <errno.h>
#include <ini.h>
#include <stdio.h>

typedef struct { cfg_layer *l; int rc; char msg[96]; } ini_ctx;

static int on_pair(void *user, const char *section, const char *name, const char *value) {
    ini_ctx *x = user;
    (void)section; /* секции не используются: ключи плоские */
    if (x->rc) return 0;
    x->rc = cfg_layer_set(x->l, name, value, x->msg, sizeof x->msg);
    return x->rc == 0; /* 0 - ошибка, inih запомнит номер строки */
}

int cfg_layer_file(cfg_layer *l, const char *path, char *err, size_t cap) {
    ini_ctx x = {.l = l};
    int line = ini_parse(path, on_pair, &x);
    if (line == 0) return 0;
    if (line < 0) snprintf(err, cap, "cannot read or allocate (%s)", path);
    else snprintf(err, cap, "%s:%d: %s", path, line, x.rc ? x.msg : "syntax error");
    return line < 0 ? -ENOENT : -EINVAL;
}
```

```text
# file: app.ini
; значения файла перекрываются окружением и аргументами
host = file.example
port = 8081
retries = 5
```

Перечитывание по `SIGHUP` строится на правиле из [системного уровня](13-system-io.md): обработчик только ставит флаг, а основной цикл собирает новую конфигурацию в отдельной структуре, проверяет её целиком и подменяет рабочую только при успехе; при ошибке остаётся прежняя, а причина уходит в лог. Демонстрация перезаписывает `app.ini` испорченным значением и посылает себе сигнал.

```c
/* cfg_demo.c */
#define _GNU_SOURCE
#include "layers.h"
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
static volatile sig_atomic_t reload;
static void on_hup(int sig) { (void)sig; reload = 1; }
static void show(const char *tag, const cfg *c) {
    printf("%s host=%s port=%u retries=%d verbose=%u\n", tag, c->host, c->port, c->retries, c->verbose);
}

int main(void) {
    char *argv[] = {"app", "--port=9090", "-v", "--config=app.ini", nullptr};
    char err[96];
    cfg cur = {0};
    setenv("APP_HOST", "env.example", 1);
    if (cfg_load(&cur, 4, argv, err, sizeof err) < 0) return 1;
    show("start", &cur); /* вывод: start host=env.example port=9090 retries=5 verbose=1 */
    FILE *f = fopen("app.ini", "w"); /* файл испорчен, пока программа работает */
    if (!f) return 1;
    fputs("retries = many\n", f);
    fclose(f);
    sigaction(SIGHUP, &(struct sigaction){.sa_handler = on_hup}, nullptr);
    raise(SIGHUP);
    if (reload) {
        cfg next = {0};
        if (cfg_load(&next, 4, argv, err, sizeof err) < 0) printf("reload rejected: %s\n", err); /* вывод: reload rejected: app.ini:1: 'many' is not a number (retries) */
        else { opts_free(cfg_opts, CFG_NOPTS, &cur); cur = next; }
    }
    show("now", &cur); /* вывод: now host=env.example port=9090 retries=5 verbose=1 */
    opts_free(cfg_opts, CFG_NOPTS, &cur);
    return 0;
}
```

Значения слоёв видны в первой строке вывода: хост пришёл из окружения (файл `file.example` перекрыт), порт из аргументов (`9090` против `8081` из файла), число попыток из файла, а `verbose` задан только аргументами.

Готовые библиотеки для других форматов различаются владением и тем, кто объявляет схему.

| Библиотека | Формат | Владение значениями | Лицензия, источник |
| --- | --- | --- | --- |
| inih | INI, обработчик на каждую пару | Указатели живут до возврата из обработчика | BSD, пакет `libinih-dev` |
| libconfuse | Иерархический, схема объявляется заранее | Строки принадлежат `cfg_t`, освобождение `cfg_free` | ISC, пакет `libconfuse-dev` |
| libconfig | Иерархический, типизированное дерево | Строки принадлежат `config_t`, освобождение `config_destroy` | LGPL-2.1, пакет `libconfig-dev` |
| tomlc99 | TOML, дерево таблиц | Таблица освобождается `toml_free` | MIT, вне apt |

#### Подводные камни

- inih продолжает разбор после ошибки обработчика и возвращает номер первой плохой строки. Обработчик, который не запомнил первую ошибку и продолжил применять значения, оставляет слой частично заполненным: в примере после первой ошибки он ничего не делает.
- Многострочные значения. Строка, начинающаяся с пробела, продолжает предыдущее значение и вызывает обработчик ещё раз с тем же именем (проверено на пакете `libinih-dev`); для строкового поля победит последняя часть.
- Комментарий после пробела. В значении `a = x ; y` inih отбрасывает `; y`, и кавычки от этого не защищают (`"p ; z"` превращается в `"p`, сами кавычки тоже остаются в значении); значения с ` ;` внутри (пароли, ссылки) передают окружением или файлом другого формата.
- Запись файла на месте. Читатель может застать его на середине записи и получить половину настроек; файл заменяют целиком через временный файл и `rename` (приём из [системного уровня](13-system-io.md)).
- Перечитывание того, что нельзя менять на лету. Порт уже открытого сокета не меняется присваиванием: сравни поле со старым значением и отклони или перезапусти нужную часть.

## Коротко

### argp: вариант в glibc

`argp_parse` описывает опции структурой и сам печатает справку, но по умолчанию завершает процесс на `--help` (код 0) и на ошибке (код 64); флаг `ARGP_NO_EXIT` оставляет решение вызывающему (при ошибке возвращается `EINVAL`). Он есть только в glibc: в musl `argp.h` отсутствует, поэтому для переносимого кода берут `getopt_long`.

### popt: таблица указателей на поля

popt (пакет `libpopt-dev`) принимает таблицу `poptOption` с указателями на целевые переменные; ошибки возвращаются отрицательным кодом `poptGetNextOpt` с текстом от `poptStrerror`, строки копируются и освобождаются вызывающим, а `POPT_AUTOHELP` добавляет `--help`, который печатает справку и завершает процесс.

## Что ещё стоит знать

Автодополнение оболочки (bash, zsh) генерируют из той же таблицы опций отдельным режимом программы или шагом сборки, а не пишут руками: тогда список опций не расходится с кодом. Справочную страницу man и текст `--help` тоже можно собирать из таблицы как генерируемый файл, чтобы установленная документация совпадала с бинарником; как сборка порождает такие файлы, показано в [сборке](02-build.md).
