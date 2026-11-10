# Логирование

## Простой логгер

### Пиши лог как побочный канал и выводи его в stderr или одним write

Лог не должен менять поведение программы: не падать, не блокировать горячий путь и не портить `errno`, иначе `LOG(...); return -errno;` вернёт код ошибки логгера, а не исходной операции. Простого логгера (одна функция, один приёмник, порог уровня задаётся при сборке) хватает, пока программа - один процесс, вывод идёт в stderr или файл под присмотром супервизора (systemd, docker), строк в секунду немного, а порог не нужно менять без перезапуска. Когда нужен чужой приёмник, уровень по модулям на ходу, поля или неблокирующая запись, переходи к части «Мощный логгер».

Уровни расставляй по последствиям. `ERROR`: операция не выполнена и вызывающий не повторит. `WARN`: сбой, с которым программа справилась, или деградация. `INFO`: события жизненного цикла (старт, остановка, смена конфигурации). `DEBUG`: подробности для разработчика, в поставляемой сборке отсечены. В запись кладут, что делали, над чем (путь, идентификатор запроса или соединения) и причину ошибки текстом; секреты и персональные данные не пишут (см. [безопасность](19-security.md)).

Умолчание: stderr или `write` в дескриптор. Поток `stdout`, перенаправленный в канал или файл, буферизуется полностью: записи лежат в буфере и пропадают, если процесс падает или завершается через `_exit`.

```c
/* Ошибка: stdout в канал буферизуется полностью, запись пропадает при падении */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    printf("INFO старт\n");
    abort();
}
```

Причину ошибки берут из `errno` в момент сбоя. Функция `strerror` не гарантирует потокобезопасности, поэтому нужна `strerror_r`; под `_GNU_SOURCE` она возвращает `char *` (результат может указывать на статическую строку, а не в твой буфер). Под `-std=c23` без `_GNU_SOURCE` функция вообще не объявлена, а в POSIX-режиме (`_POSIX_C_SOURCE`) та же функция возвращает `int`: присваивание результата указателю не компилируется.

```c
/* errstr.c */
#define _GNU_SOURCE
#include <string.h>

/* log_errstr - текст ошибки err; строка живёт до следующего вызова в этом потоке */
const char *log_errstr(int err) {
    static thread_local char buf[64];
    return strerror_r(err, buf, sizeof buf);
}
```

Расширение glibc `%m` (подставляет текст `errno`) короче, но GCC под `-Wpedantic` отвергает его в проверке формата («ISO C does not support the %m gnu_printf format»), а Clang принимает; чтобы код собирался одинаково под обоими, в книге причину передаёт `log_errstr`.

### Делай синхронный логгер: макрос, буфер на стеке, один write

Запись целиком собирается в буфер на стеке и уходит одним `write`, поэтому строки разных потоков не рвут друг друга, а куча и блокировки не нужны. Макрос подставляет `__FILE_NAME__` (расширение GCC/Clang: имя файла без каталога; работает на Clang 19, 20 и GCC 14) и номер строки. Отсечение уровня сделано обычным `if` с константным условием: компилятор выбрасывает ветку целиком, но проверка формата и использование переменных остаются, а вариант через `#if` оставил бы переменную, нужную только логу, неиспользованной (`-Wunused-variable` под `-Werror`).

```c
/* log.h */
#ifndef LOG_H
#define LOG_H

enum { LV_DEBUG, LV_INFO, LV_WARN, LV_ERROR };

#ifndef LOG_MIN
#define LOG_MIN LV_INFO
#endif

extern int log_fd;

/* gnu::format: расширение GCC/Clang, проверка формата как у printf */
[[gnu::format(printf, 4, 5)]]
void log_write(int lvl, const char *file, long line, const char *fmt, ...);

const char *log_errstr(int err);

#define LOG(lvl, fmt, ...)                                                          \
    do {                                                                            \
        if ((lvl) >= LOG_MIN)                                                       \
            log_write((lvl), __FILE_NAME__, (long)__LINE__, fmt __VA_OPT__(,) __VA_ARGS__); \
    } while (0)
#endif
```

```c
/* log.c */
#include "log.h"
#include <errno.h>
#include <stdarg.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int log_fd = STDERR_FILENO;

void log_write(int lvl, const char *file, long line, const char *fmt, ...) {
    static const char *const names[] = {"DEBUG", "INFO", "WARN", "ERROR"};
    int saved = errno;
    char buf[256];
    int n = snprintf(buf, sizeof buf, "%s %s:%ld: ", names[lvl], file, line);
    if ((size_t)n >= sizeof buf) n = (int)sizeof buf - 1;
    va_list ap;
    va_start(ap, fmt);
    int m = vsnprintf(buf + n, sizeof buf - (size_t)n, fmt, ap);
    va_end(ap);
    size_t len = (size_t)n + (m > 0 ? (size_t)m : 0);
    if (len > sizeof buf - 2) {
        len = sizeof buf - 2;
        memcpy(buf + len - 3, "...", 3); /* пометка усечения */
    }
    buf[len++] = '\n';
    ssize_t r;
    do r = write(log_fd, buf, len); while (r < 0 && errno == EINTR);
    errno = saved;
}
```

Вызов видит `errno` от неудавшейся операции, а после возврата из `LOG` он прежний: `errno` сохраняется и восстанавливается внутри `log_write`.

```c
/* main.c */
#include "log.h"
#include <errno.h>
#include <stdio.h>
#include <unistd.h>

static int load(const char *path) {
    FILE *f = fopen(path, "r");
    if (!f) {
        LOG(LV_ERROR, "open %s: %s", path, log_errstr(errno));
        return -errno;
    }
    fclose(f);
    return 0;
}

int main(void) {
    log_fd = STDOUT_FILENO;
    LOG(LV_DEBUG, "скрыто: ниже порога");
    LOG(LV_INFO, "старт"); /* вывод: INFO main.c:20: старт */
    int rc = load("/no/such/file"); /* вывод: ERROR main.c:10: open /no/such/file: No such file or directory */
    printf("rc=%d\n", rc); /* вывод: rc=-2 */
    return 0;
}
```

#### Подводные камни

- Запись длиннее `PIPE_BUF` (4096 байт в Linux) в канал не обязана быть неделимой: проверено, что две записи по 70000 байт от разных процессов перемешиваются построчно, а записи до `PIPE_BUF` остаются целыми. Поэтому буфер записи ограничен, а лишнее обрезается с пометкой `...`. Для файла с `O_APPEND` гарантирована позиция (конец файла), но не полнота: частичную запись при нехватке места или сигнале логгер не повторяет.
- Запись в канал, у которого закрыт читатель, убивает процесс сигналом `SIGPIPE` (проверено: код выхода 141). Программа, пишущая лог в канал или сокет, на старте делает `signal(SIGPIPE, SIG_IGN)`; тогда `write` возвращает `-1` с `EPIPE`, а логгер молча теряет запись, но процесс живёт.
- Синхронная запись блокируется, если читатель лога не успевает (полный канал, медленный диск): горячий путь встаёт на `write`. Это главный довод за асинхронный логгер из части «Мощный логгер».
- Аргументы отключённого уровня не вычисляются, а `lvl` в макросе вычисляется дважды: передавай константу, а не выражение с побочным эффектом, и не клади в аргументы `LOG` работу, без которой программа не обойдётся.
- Формат обязан быть литералом: `LOG(LV_INFO, user_text)` даёт `%n` и `%s` из данных; атрибут `format` вместе с `-Wformat=2` ловит такой вызов. Правила форматов подробно в главе про [строки](06-strings-and-buffers.md).
- Построение одной строки лога из нескольких вызовов `write` или `printf` рвётся при записи из нескольких потоков; один вызов на запись, как здесь, устраняет это (общие правила в главе про [потоки](12-concurrency.md)).

### Пиши записи в logfmt и экранируй всё, что приходит снаружи

Лог читают построчно (`grep`, парсеры), поэтому данные снаружи (имя пользователя, путь, заголовок запроса), попавшие в запись как есть, позволяют подделать чужую запись переводом строки, а управляющие символы терминала (ESC) управляют экраном того, кто читает лог через `cat`: это инъекция в лог. Правило: одна запись - одна строка; значение сообщения в кавычках, в нём экранированы `"`, `\`, переводы строк и все управляющие символы; время в UTC по ISO 8601 с миллисекундами (`clock_gettime`, `gmtime_r`), рядом `pid`, `tid`, уровень и место. Формат logfmt - пары `ключ=значение` через пробел; экранирование JSON описано в главе про [сериализацию](14-serialization.md), здесь только logfmt.

```c
/* logfmt.h */
#ifndef LOGFMT_H
#define LOGFMT_H
#include <stddef.h>
#include <time.h>

typedef struct {
    struct timespec ts;
    int level, pid, tid;
    const char *file;
    long line;
    const char *msg;
    bool truncated; /* msg уже обрезано вызывающим */
} log_rec;

/* log_format - пишет запись logfmt с '\n' в out (не NUL-терминирует).
 * cap: не меньше 256. msg экранируется; не поместившееся отбрасывается, в конец
 * добавляется trunc=1. Возврат: длина записи. */
size_t log_format(char *out, size_t cap, const log_rec *r);

/* log_escape - дописывает s в out[*n..limit) с экранированием; false, если не поместилось */
bool log_escape(char *out, size_t *n, size_t limit, const char *s);
#endif
```

```c
/* escape.c */
#include "logfmt.h"
#include <stdio.h>
#include <string.h>

bool log_escape(char *out, size_t *n, size_t limit, const char *s) {
    for (; *s; s++) {
        unsigned char c = (unsigned char)*s;
        char t[5] = {(char)c, 0};
        const char *e = t;
        switch (c) {
        case '"':  e = "\\\""; break;
        case '\\': e = "\\\\"; break;
        case '\n': e = "\\n"; break;
        case '\r': e = "\\r"; break;
        case '\t': e = "\\t"; break;
        default:
            if (c < 0x20 || c == 0x7f) snprintf(t, sizeof t, "\\x%02x", c);
        }
        size_t k = strlen(e);
        if (*n + k > limit) return false;
        memcpy(out + *n, e, k);
        *n += k;
    }
    return true;
}
```

```c
/* logfmt.c */
#define _GNU_SOURCE
#include "logfmt.h"
#include <stdio.h>
#include <string.h>

enum { RESERVE = 12 }; /* закрывающая кавычка, " trunc=1", '\n', NUL и запас */

static const char *const names[] = {"debug", "info", "warn", "error"};

size_t log_format(char *out, size_t cap, const log_rec *r) {
    struct tm tm;
    char ts[32];
    gmtime_r(&r->ts.tv_sec, &tm);
    strftime(ts, sizeof ts, "%Y-%m-%dT%H:%M:%S", &tm);
    int h = snprintf(out, cap, "ts=%s.%03ldZ level=%s pid=%d tid=%d src=%.64s:%ld msg=\"",
                     ts, r->ts.tv_nsec / 1000000, names[r->level], r->pid, r->tid, r->file, r->line);
    size_t n = (size_t)h;
    bool ok = log_escape(out, &n, cap - RESERVE, r->msg);
    out[n++] = '"';
    if (!ok || r->truncated) n += (size_t)snprintf(out + n, 9, " trunc=1");
    out[n++] = '\n';
    return n;
}
```

Функция `log_write` теперь только собирает поля и отдаёт их форматтеру; часы и идентификаторы берутся здесь, поэтому сам форматтер детерминирован и проверяется с фиксированной записью.

```c
/* log.c */
#define _GNU_SOURCE
#include "log.h"
#include "logfmt.h"
#include <errno.h>
#include <stdarg.h>
#include <stdio.h>
#include <unistd.h>

int log_fd = STDERR_FILENO;

void log_write(int lvl, const char *file, long line, const char *fmt, ...) {
    int saved = errno;
    char msg[400], out[1024];
    va_list ap;
    va_start(ap, fmt);
    int m = vsnprintf(msg, sizeof msg, fmt, ap);
    va_end(ap);
    log_rec r = {.level = lvl, .pid = getpid(), .tid = gettid(), .file = file,
                 .line = line, .msg = msg, .truncated = m >= (int)sizeof msg};
    clock_gettime(CLOCK_REALTIME, &r.ts);
    size_t n = log_format(out, sizeof out, &r);
    ssize_t w;
    do w = write(log_fd, out, n); while (w < 0 && errno == EINTR);
    errno = saved;
}
```

```c
/* main.c */
#include "logfmt.h"
#include <stdio.h>
#include <string.h>

int main(void) {
    log_rec r = {.ts = {1700000000, 123000000}, .level = 2, .pid = 42, .tid = 43,
                 .file = "auth.c", .line = 7,
                 .msg = "login failed: bob\nINFO forged \"ok\"\x1b[31m"};
    char out[256];
    fwrite(out, 1, log_format(out, sizeof out, &r), stdout); /* вывод: ts=2023-11-14T22:13:20.123Z level=warn pid=42 tid=43 src=auth.c:7 msg="login failed: bob\nINFO forged \"ok\"\x1b[31m" */
    char big[301];
    memset(big, 'x', 300);
    big[300] = '\0';
    r.msg = big;
    size_t n = log_format(out, sizeof out, &r);
    printf("%d %d\n", n <= sizeof out, memcmp(out + n - 9, " trunc=1\n", 9) == 0); /* вывод: 1 1 */
    return 0;
}
```

Подставленные в сообщение перевод строки и `ESC` превратились в `\n` и `\x1b`, подделать вторую запись не получится; кавычка внутри значения экранирована, поэтому конец значения нельзя изобразить данными. Лишнее не помещается в `cap` и отбрасывается, и признак `trunc=1` предупреждает читателя, что сообщение неполное.

#### Подводные камни

- Ключи полей берутся только из литералов в коде: экранируется значение, а не ключ, поэтому ключ из данных снова открывает инъекцию.
- Экранируй один раз, при построении записи. Приёмник получает готовую строку: например `syslog(3)` пересылает перевод строки как есть (проверено слушателем на `/dev/log`), поэтому ему нельзя отдавать сырой текст.
- Байты не меньше `0x80` проходят без изменений, чтобы не портить UTF-8, а обрезка по байту может разрезать многобайтный символ на границе `trunc=1`; читатель лога должен это терпеть.
- Строки журнала не переводят: запись читают разработчик и скрипты, поэтому текст сообщений и ключи полей неизменны в любой локали. Числа и время в записи форматируются в локали `C` (после `setlocale(LC_NUMERIC, "C")`, как в программах с gettext), иначе десятичная запятая сломает разбор; правило локали описано в главе про [локализацию](18-localization.md).
- Время `CLOCK_REALTIME` может прыгнуть назад при коррекции часов; длительности измеряй через `CLOCK_MONOTONIC` и пиши их отдельным полем, а не разностью меток записей.

## Мощный логгер

### Отдавай вывод в приёмник за интерфейсом, а не пиши в stderr из библиотеки

Библиотека, которая сама пишет в stderr, отнимает у приложения решение, куда и в каком формате писать, поэтому она принимает обработчик `log_fn` с пользовательским указателем (приём обратных вызовов описан в главе про [API и модули](07-api-and-modules.md)), а без обработчика молчит. Приёмник логгера приложения имеет тот же тип `log_fn`, так что подключается к библиотеке напрямую, без адаптера. Сам логгер получает приёмник и часы через таблицу операций `log_ops`: в проверяемых примерах подставляются фиксированные часы и приёмник в stdout.

```c
/* lib.h */
#ifndef LIB_H
#define LIB_H

enum { LIB_DEBUG, LIB_INFO, LIB_WARN, LIB_ERROR }; /* та же шкала, что у приложения */

typedef void (*log_fn)(void *user, int level, const char *msg);

/* lib_set_log - приёмник сообщений библиотеки; fn == nullptr возвращает молчание.
 * msg валидна только на время вызова. Вызывай до первого использования библиотеки. */
void lib_set_log(log_fn fn, void *user);

/* lib_parse - 0 или -EINVAL для пустого ввода */
int lib_parse(const char *text);
#endif
```

```c
/* lib.c */
#include "lib.h"
#include <errno.h>
#include <stdarg.h>
#include <stdio.h>

static log_fn g_fn;
static void *g_user;

void lib_set_log(log_fn fn, void *user) { g_fn = fn; g_user = user; }

[[gnu::format(printf, 2, 3)]]
static void say(int level, const char *fmt, ...) {
    if (!g_fn) return; /* без приёмника запись даже не форматируется */
    char msg[200];
    va_list ap;
    va_start(ap, fmt);
    vsnprintf(msg, sizeof msg, fmt, ap);
    va_end(ap);
    g_fn(g_user, level, msg);
}

int lib_parse(const char *text) {
    if (!*text) {
        say(LIB_WARN, "пустой ввод");
        return -EINVAL;
    }
    say(LIB_DEBUG, "разобрано %d байт", 1);
    return 0;
}
```

Приёмник, который приложение даёт и библиотеке, и своему логгеру, - функция записи в дескриптор; таблица операций собирает её с часами.

```c
/* log.h */
#ifndef LOG_H
#define LOG_H
#include <time.h>

enum { LV_DEBUG, LV_INFO, LV_WARN, LV_ERROR, LV_FATAL };

typedef void (*log_fn)(void *user, int level, const char *msg);

/* Таблица операций логгера: write принимает готовую запись без '\n', now отдаёт часы. */
typedef struct log_ops {
    log_fn write;
    struct timespec (*now)(void *ctx);
} log_ops;

/* Приёмник в дескриптор: ctx - int *fd; запись и '\n' уходят одним writev. */
void log_fd_write(void *ctx, int level, const char *line);
struct timespec log_real_now(void *ctx);
#endif
```

```c
/* sink_fd.c */
#define _GNU_SOURCE
#include "log.h"
#include <string.h>
#include <sys/uio.h>
#include <unistd.h>

void log_fd_write(void *ctx, int level, const char *line) {
    (void)level;
    struct iovec v[2] = {{(void *)line, strlen(line)}, {(void *)"\n", 1}};
    ssize_t r = writev(*(int *)ctx, v, 2);
    (void)r;
}

struct timespec log_real_now(void *ctx) {
    (void)ctx;
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts;
}
```

```c
/* main.c */
#include "lib.h"
#include "log.h"
#include <unistd.h>

int main(void) {
    int out = STDOUT_FILENO;
    lib_parse(""); /* приёмника нет: тишина */
    lib_set_log(log_fd_write, &out);
    lib_parse(""); /* вывод: пустой ввод */
    return 0;
}
```

#### Подводные камни

- Обработчик вызывается в потоке, где вызвана библиотека, и может вызываться из нескольких потоков сразу: он обязан быть потокобезопасным, а библиотека не должна вызывать его под собственным мьютексом (обработчик, вернувшийся в библиотеку, получит взаимную блокировку).
- Строка `msg` живёт только на время вызова: обработчик, который кладёт её в очередь, копирует.
- Приёмник устанавливают до создания потоков: `lib_set_log` пишет два глобальных указателя без синхронизации.
- Уровни библиотека объявляет в своём заголовке и документирует их соответствие шкале приложения; вызывающий переводит их в свои при необходимости.
- Две библиотеки с одним `log_fn` и общим `user` не различают источник: приложение даёт каждой свой `user` (например, имя библиотеки).

### Задавай уровень по модулям и меняй его на ходу, а поля передавай массивом пар

Один глобальный порог заставляет перезапускать программу ради отладки одного подсистемы, поэтому у каждого модуля свой дескриптор с именем и атомарным уровнем: его меняют из другого потока (обработчик команды, сигнал) без блокировок. Дескриптор объявляется явно в файле модуля и регистрируется в таблице при старте, чтобы уровень можно было менять по имени; регистрация не потокобезопасна и делается до запуска потоков (автоматическая регистрация через секцию описана в главе про [возможности компилятора](08-compiler-features.md)). Поля записи передаются массивом пар на стеке: сообщение остаётся константой, данные идут отдельно, и приёмник может разобрать их по ключам.

```c
/* log.h */
#ifndef LOG_H
#define LOG_H
#include <stdatomic.h>
#include <time.h>

enum { LV_DEBUG, LV_INFO, LV_WARN, LV_ERROR, LV_FATAL };
#ifndef LOG_MIN
#define LOG_MIN LV_DEBUG
#endif

typedef void (*log_fn)(void *user, int level, const char *msg);
typedef struct log_ops { log_fn write; struct timespec (*now)(void *ctx); } log_ops;
typedef struct log_module { const char *name; _Atomic int level; } log_module;
typedef struct log_kv {
    const char *key;
    enum { KV_END, KV_S, KV_I } kind;
    union { const char *s; long long i; };
} log_kv;

#define KV_STR(k, v) {.key = (k), .kind = KV_S, .s = (v)}
#define KV_INT(k, v) {.key = (k), .kind = KV_I, .i = (v)}

void log_init(const log_ops *vt, void *ctx); /* до запуска потоков */
void log_register(log_module *m);            /* до запуска потоков */
int log_set_level(const char *name, int level); /* 0 или -ENOENT; потокобезопасна */
void log_emit(log_module *m, int lvl, const char *file, long line, const char *msg, const log_kv *kv);
void log_fd_write(void *ctx, int level, const char *line);
struct timespec log_real_now(void *ctx);

#define LOG_KV(m, lvl, msg, ...)                                                      \
    do {                                                                              \
        if ((lvl) >= LOG_MIN &&                                                       \
            (lvl) >= atomic_load_explicit(&(m)->level, memory_order_relaxed))         \
            log_emit((m), (lvl), __FILE_NAME__, (long)__LINE__, (msg),                \
                     (log_kv[]){__VA_ARGS__ __VA_OPT__(,) {0}});                      \
    } while (0)
#endif
```

Первая версия `log.h` из предыдущего раздела заменена: добавились дескриптор модуля, пара «ключ-значение» и макрос `LOG_KV`. Массив пар заканчивается нулевым элементом-ограничителем, поэтому количество полей считать не нужно; сам массив живёт на стеке вызывающего и не требует выделения памяти. Макрос проверяет уровень по обоим порогам (сборка и модуль), и аргументы полей вычисляются только при включённом уровне.

```c
/* fixed.h */
#ifndef FIXED_H
#define FIXED_H
#include <time.h>

/* фиксированные часы для проверяемых примеров: 2023-11-14T22:13:20.123Z */
static inline struct timespec fixed_now(void *ctx) {
    (void)ctx;
    return (struct timespec){1700000000, 123000000};
}
#endif
```

Сборка записи вынесена в маленький буфер с контролем границы: всё, что не помещается, отбрасывается (обрыв может прийти и посреди значения, тогда оно закрывается кавычкой), а в конец ставится `trunc=1`; значение в кавычках экранируется теми же правилами, что и в простом логгере (код сокращён, но правила те же).

```c
/* logbuf.h */
#ifndef LOGBUF_H
#define LOGBUF_H
#include <stdio.h>
#include <string.h>

typedef struct { char *p; size_t n, cap; bool over; } lbuf;
enum { LB_RESERVE = 10 }; /* закрывающая кавычка (1), " trunc=1" (8) и NUL (1) */

static inline void lb_put(lbuf *b, const char *s) {
    size_t k = strlen(s);
    if (b->over || b->n + k > b->cap - LB_RESERVE) { b->over = true; return; }
    memcpy(b->p + b->n, s, k + 1);
    b->n += k;
}

static inline void lb_quoted(lbuf *b, const char *s) {
    lb_put(b, "\"");
    if (b->over) return;
    for (; *s; s++) {
        unsigned char c = (unsigned char)*s;
        char t[5] = {(char)c, 0};
        if (c == '"' || c == '\\') snprintf(t, sizeof t, "\\%c", c);
        else if (c == '\n') strcpy(t, "\\n");
        else if (c == '\r') strcpy(t, "\\r");
        else if (c == '\t') strcpy(t, "\\t");
        else if (c < 0x20 || c == 0x7f) snprintf(t, sizeof t, "\\x%02x", c);
        lb_put(b, t);
    }
    b->p[b->n++] = '"'; /* места хватает: закрывающая кавычка в запасе */
    b->p[b->n] = '\0';
}
#endif
```

```c
/* log.c */
#define _GNU_SOURCE
#include "log.h"
#include "logbuf.h"
#include <errno.h>
#include <time.h>

static const log_ops *g_vt;
static void *g_ctx;

void log_init(const log_ops *vt, void *ctx) { g_vt = vt; g_ctx = ctx; }

void log_emit(log_module *m, int lvl, const char *file, long line, const char *msg, const log_kv *kv) {
    static const char *const names[] = {"debug", "info", "warn", "error", "fatal"};
    if (!g_vt) return; /* логгер не настроен: молчим */
    int saved = errno;
    struct timespec ts = g_vt->now(g_ctx);
    struct tm tm;
    char out[1024], tmp[160], day[32];
    gmtime_r(&ts.tv_sec, &tm);
    strftime(day, sizeof day, "%Y-%m-%dT%H:%M:%S", &tm);
    snprintf(tmp, sizeof tmp, "ts=%s.%03ldZ level=%s mod=%.32s src=%.40s:%ld msg=", day,
             ts.tv_nsec / 1000000, names[lvl], m->name, file, line);
    lbuf b = {out, 0, sizeof out, false};
    lb_put(&b, tmp);
    lb_quoted(&b, msg);
    for (; kv->key; kv++) {
        lb_put(&b, " ");
        lb_put(&b, kv->key);
        lb_put(&b, "=");
        if (kv->kind == KV_I) {
            snprintf(tmp, sizeof tmp, "%lld", kv->i);
            lb_put(&b, tmp);
        } else lb_quoted(&b, kv->s ? kv->s : "");
    }
    if (b.over) memcpy(out + b.n, " trunc=1", 9);
    g_vt->write(g_ctx, lvl, out);
    errno = saved;
}
```

```c
/* log_level.c */
#include "log.h"
#include <errno.h>
#include <string.h>

static log_module *g_mods[32];
static size_t g_nmods;

void log_register(log_module *m) {
    if (g_nmods < sizeof g_mods / sizeof *g_mods) g_mods[g_nmods++] = m;
}

int log_set_level(const char *name, int level) {
    for (size_t i = 0; i < g_nmods; i++) {
        if (strcmp(g_mods[i]->name, name) == 0) {
            atomic_store_explicit(&g_mods[i]->level, level, memory_order_relaxed);
            return 0;
        }
    }
    return -ENOENT;
}
```

```c
/* main.c */
#include "fixed.h"
#include "log.h"
#include <unistd.h>

static log_module net = {"net", LV_INFO};
static log_module db = {"db", LV_WARN};

int main(void) {
    int out = STDOUT_FILENO;
    static const log_ops ops = {log_fd_write, fixed_now};
    log_init(&ops, &out);
    log_register(&net);
    log_register(&db);
    LOG_KV(&net, LV_INFO, "connected", KV_STR("peer", "10.0.0.1"), KV_INT("fd", 7)); /* вывод: ts=2023-11-14T22:13:20.123Z level=info mod=net src=main.c:15 msg="connected" peer="10.0.0.1" fd=7 */
    LOG_KV(&db, LV_INFO, "slow query"); /* уровень модуля WARN: тишина */
    log_set_level("db", LV_DEBUG);
    LOG_KV(&db, LV_DEBUG, "query", KV_STR("sql", "a\"b\nc")); /* вывод: ts=2023-11-14T22:13:20.123Z level=debug mod=db src=main.c:18 msg="query" sql="a\"b\nc" */
    return 0;
}
```

#### Подводные камни

- Атомарность уровня не делает запись потокобезопасной: порядок записей из разных потоков определяет приёмник, а не `log_emit`.
- Регистрация модуля из потока при работающих других потоках гонится с `log_set_level`: регистрируй всё в `main` до запуска потоков или защищай таблицу мьютексом.
- Значение поля `KV_STR` должно пережить вызов `LOG_KV`, но не дольше: массив пар живёт только на время вызова макроса (блок do-while), указатель в нём нельзя сохранять в приёмнике.
- Ключи - литералы из кода: они не экранируются (см. выше про инъекцию); значения экранируются всегда.
- `lvl` вычисляется несколько раз, а аргументы полей только при включённом уровне; не клади в них вызовы с побочными эффектами.
- Сообщения библиотеки (`log_fn` из предыдущего раздела) приложение пропускает через собственный модуль: адаптер с `user`, равным дескриптору модуля, вызывает `log_emit`, и тогда уровень библиотеки меняется по имени так же, как у остальных.

### Выбирай приёмник по среде запуска: stderr под супервизором, syslog, journald или файл

Под супервизором (systemd, docker) лучший приёмник - stderr или stdout: сбор, хранение и ротацию делает среда, а программа ничего не знает о файлах. Собственные приёмники нужны, когда такой среды нет или записи должны идти в общую систему. Каждый приёмник - `log_fn`, подключается он через таблицу `log_ops`; объявления собраны в одном заголовке.

```c
/* sinks.h */
#ifndef SINKS_H
#define SINKS_H
#include "log.h"
#include <pthread.h>

typedef struct { const char *path; int fd; pthread_mutex_t mu; } file_sink;

/* file_sink_init - открывает path на дописывание; path заимствуется и живёт дольше приёмника.
 * 0 или -errno. file_sink_write потокобезопасна; если после прошлой записи пришёл
 * SIGHUP, перед записью файл открывается заново. */
int file_sink_init(file_sink *s, const char *path);
void file_sink_write(void *ctx, int level, const char *line);
void file_sink_close(file_sink *s);
void file_sink_on_sighup(int sig); /* обработчик SIGHUP: только ставит флаг */

/* syslog_sink_open - openlog; ident не копируется и должна жить до закрытия */
void syslog_sink_open(const char *ident);
void syslog_sink_write(void *ctx, int level, const char *line);
void journal_sink_write(void *ctx, int level, const char *line);
int log_prio(int level); /* уровень логгера в приоритет syslog */
#endif
```

```c
/* sink_file.c */
#define _GNU_SOURCE
#include "sinks.h"
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <string.h>
#include <sys/uio.h>
#include <unistd.h>

static volatile sig_atomic_t reopen_requested;

void file_sink_on_sighup(int sig) { (void)sig; reopen_requested = 1; }

static int open_append(const char *p) { return open(p, O_WRONLY | O_APPEND | O_CREAT | O_CLOEXEC, 0644); }

int file_sink_init(file_sink *s, const char *path) {
    s->path = path;
    s->fd = open_append(path);
    if (s->fd < 0) return -errno;
    return -pthread_mutex_init(&s->mu, NULL);
}

void file_sink_write(void *ctx, int level, const char *line) {
    file_sink *s = ctx;
    (void)level;
    pthread_mutex_lock(&s->mu);
    if (reopen_requested) {
        reopen_requested = 0;
        int nfd = open_append(s->path);
        if (nfd >= 0) { close(s->fd); s->fd = nfd; } /* не открылся: пишем в старый */
    }
    struct iovec v[2] = {{(void *)line, strlen(line)}, {(void *)"\n", 1}};
    ssize_t r = writev(s->fd, v, 2);
    (void)r;
    pthread_mutex_unlock(&s->mu);
}

void file_sink_close(file_sink *s) { close(s->fd); pthread_mutex_destroy(&s->mu); }
```

```c
/* sink_syslog.c */
#include "sinks.h"
#include <syslog.h>

int log_prio(int level) {
    static const int prio[] = {LOG_DEBUG, LOG_INFO, LOG_WARNING, LOG_ERR, LOG_CRIT};
    return prio[level];
}

void syslog_sink_open(const char *ident) { openlog(ident, LOG_PID, LOG_USER); }

void syslog_sink_write(void *ctx, int level, const char *line) {
    (void)ctx;
    syslog(log_prio(level), "%s", line); /* запись - данные, а не формат */
}
```

```c
/* sink_journal.c */
#define SD_JOURNAL_SUPPRESS_LOCATION /* без CODE_FILE=sink_journal.c в каждой записи */
#include "sinks.h"
#include <systemd/sd-journal.h>

void journal_sink_write(void *ctx, int level, const char *line) {
    (void)ctx;
    sd_journal_send("MESSAGE=%s", line, "PRIORITY=%i", log_prio(level), NULL);
}
```

Файл: ротацию делает logrotate, а программа только открывает файл заново по `SIGHUP`. Обработчик сигнала лишь ставит флаг (`volatile sig_atomic_t`), а сам `open` делается в приёмнике перед следующей записью, потому что в обработчике нельзя брать мьютекс, а подмена дескриптора без него гонится с писателем, который в этот момент пишет в старый. Устанавливается он через `sigaction(SIGHUP, ...)` с `sa_handler = file_sink_on_sighup` и `SA_RESTART`, чтобы сигнал не прерывал чужие системные вызовы. Свою ротацию (переименования, счётчики) не пиши: она гонится с logrotate и теряет записи между переименованием и новым `open`.

```text
/var/log/app/app.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    postrotate
        kill -HUP "$(cat /run/app.pid)"
    endscript
}
```

syslog: `openlog` не копирует `ident` (проверено: если буфер с именем перезаписать, следующие записи уходят уже с новым именем), поэтому передавай строковый литерал или строку со статической длительностью жизни. Запись уходит датаграммой в сокет `/dev/log` в виде `<приоритет>Mon DD hh:mm:ss ident[pid]: текст`; время, имя и pid добавляет сама libc, так что поле `ts` в записи дублирует их. Перевод строки в тексте libc пересылает как есть (проверено слушателем на `/dev/log`), поэтому приёмник получает уже экранированную запись; формат всегда `"%s"`. Без слушателя на сокете запись теряется без ошибки.

journald: есть два пути. Первый: писать в stdout или stderr юнита systemd, начиная строку с префикса `<N>` из `sd-daemon.h` (например `<4>` для `SD_WARNING`); он понимается только там, где вывод сервиса systemd направляет в журнал, а в docker, терминале или файле останется текстом. Второй: `sd_journal_send` с полями `MESSAGE=` и `PRIORITY=`; в приёмнике выше передаются только они, потому что `log_fn` получает готовую строку, а для настоящих полей журнала приёмник должен принимать массив `log_kv`. Без запущенного journald `sd_journal_send` возвращает 0 и не падает, но запись не доходит; в контейнере проверить вывод журнала нельзя, это возможно только на хосте с systemd.

#### Подводные камни

- Свои уровни не называй `LOG_INFO`, `LOG_DEBUG` и подобно: `<syslog.h>` определяет их макросами, и `enum { LOG_DEBUG }` после включения заголовка не компилируется. Поэтому в книге уровни логгера называются `LV_*`, а в приёмнике их переводит `log_prio`.
- Обработчик `SIGHUP` не переоткрывает файл сам: мьютекс в обработчике небезопасен, а подмена дескриптора без него гонится с писателем (сам `open` для сигналов безопасен); обработчик только ставит флаг.
- Приёмник файла сериализует потоки мьютексом, поэтому при синхронной записи медленный диск стопорит все потоки; для горячего пути ставь его за асинхронной очередью из следующего раздела.
- Строка `ident` в `openlog` и путь файла живут дольше приёмника: передавай литералы или статические строки.
- Мьютекс файлового приёмника в момент `fork` может быть захвачен другим потоком, и потомок, пишущий в тот же приёмник, зависнет: переинициализируй его в обработчике `pthread_atfork` потомка, как это сделано для очереди ниже.

### Не блокируй горячий путь: записывай лог в очередь, а пиши из отдельного потока

Синхронный `write` блокируется на медленном приёмнике (полный канал, диск, сокет syslog), а асинхронный логгер превращает запись в копирование строки в кольцевой буфер, и настоящую запись делает поток-писатель. Взамен приходится решать четыре вещи: что делать при переполнении, как не потерять хвост при падении, что происходит при `fork` и в каком порядке останавливать. Логгер строится декоратором: `async_write` - тоже `log_fn`, а настоящий приёмник стоит за ним; примитивы потоков описаны в главе про [потоки](12-concurrency.md).

Политика переполнения: очередь не растёт, новая запись отбрасывается, а счётчик потерь превращается в одну запись «потеряно N», которую писатель выдаёт после всего принятого. Блокировать производителя вместо потерь значило бы вернуться к синхронному логгеру. Память под очередь выделяется один раз в `async_init`: логгер, которому нужна память на горячем пути, замолчит именно тогда, когда памяти нет.

```c
/* ring.h */
#ifndef RING_H
#define RING_H
#include <stddef.h>

enum { RING_LINE = 1024 }; /* слот равен максимальной записи: обрезки нет */
typedef struct { int level; char line[RING_LINE]; } slot;
typedef struct { slot *s; size_t cap, head, count; unsigned long dropped, reported; } ring;

int ring_init(ring *r, size_t cap); /* cap > 0; единственное выделение; 0 или -ENOMEM */
void ring_free(ring *r);
/* ring_push - копирует запись; в полную очередь не принимает: false и dropped++ */
bool ring_push(ring *r, int level, const char *line);
/* ring_pop - старейшая запись; когда очередь пуста, а потери не сообщены, запись о них */
bool ring_pop(ring *r, slot *out);
#endif
```

```c
/* ring.c */
#include "ring.h"
#include "log.h"
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

int ring_init(ring *r, size_t cap) {
    *r = (ring){.s = calloc(cap, sizeof(slot)), .cap = cap};
    return r->s ? 0 : -ENOMEM;
}

void ring_free(ring *r) { free(r->s); }

bool ring_push(ring *r, int level, const char *line) {
    if (r->count == r->cap) { r->dropped++; return false; }
    slot *s = &r->s[(r->head + r->count++) % r->cap];
    s->level = level;
    snprintf(s->line, sizeof s->line, "%s", line);
    return true;
}

bool ring_pop(ring *r, slot *out) {
    if (r->count) {
        *out = r->s[r->head];
        r->head = (r->head + 1) % r->cap;
        r->count--;
        return true;
    }
    if (r->dropped == r->reported) return false;
    out->level = LV_WARN;
    snprintf(out->line, sizeof out->line, "msg=\"log overflow\" lost=%lu", r->dropped - r->reported);
    r->reported = r->dropped;
    return true;
}
```

```c
/* async.h */
#ifndef ASYNC_H
#define ASYNC_H
#include "log.h"
#include "ring.h"
#include <pthread.h>

typedef enum { AS_NEW, AS_RUNNING, AS_SYNC } as_state; /* буфер до старта, писатель, прямая запись */

typedef struct {
    const log_ops *target; /* настоящий приёмник */
    void *target_ctx;
    pthread_mutex_t mu;
    pthread_cond_t wake, idle;
    pthread_t th;
    as_state state;
    bool stop, busy;
    ring q;
} async_log;

/* Один экземпляр на процесс, со статическим временем жизни (его видят обработчики fork).
 * async_write - приёмник log_fn, ctx - async_log *; до async_start записи копятся.
 * async_flush ждёт, пока всё принятое записано; запись уровня LV_FATAL сначала сбрасывает очередь,
 * а затем идёт прямо в приёмник из вызывающего потока.
 * async_stop дописывает очередь, останавливает писателя и переводит логгер в прямую запись. */
int async_init(async_log *a, const log_ops *target, void *ctx, size_t cap);
int async_start(async_log *a); /* 0 или -errno */
void async_write(void *ctx, int level, const char *line);
void async_flush(async_log *a);
void async_stop(async_log *a);
void async_destroy(async_log *a); /* после async_stop и после того, как производители закончили */
void async_atfork(async_log *a);
#endif
```

```c
/* async.c */
#define _GNU_SOURCE
#include "async.h"

int async_init(async_log *a, const log_ops *target, void *ctx, size_t cap) {
    *a = (async_log){.target = target, .target_ctx = ctx};
    pthread_mutex_init(&a->mu, NULL);
    pthread_cond_init(&a->wake, NULL);
    pthread_cond_init(&a->idle, NULL);
    return ring_init(&a->q, cap);
}

void async_write(void *ctx, int level, const char *line) {
    async_log *a = ctx;
    if (level >= LV_FATAL) { /* минует очередь: её переполнение не может съесть эту запись */
        async_flush(a);
        a->target->write(a->target_ctx, level, line);
        return;
    }
    pthread_mutex_lock(&a->mu);
    if (a->state == AS_SYNC) { /* писателя нет: пишем в вызывающем потоке */
        pthread_mutex_unlock(&a->mu);
        a->target->write(a->target_ctx, level, line);
        return;
    }
    ring_push(&a->q, level, line);
    pthread_cond_signal(&a->wake);
    pthread_mutex_unlock(&a->mu);
}

void async_flush(async_log *a) {
    slot s;
    pthread_mutex_lock(&a->mu);
    if (a->state == AS_RUNNING) {
        while (a->q.count || a->busy) pthread_cond_wait(&a->idle, &a->mu);
    } else {
        while (ring_pop(&a->q, &s)) a->target->write(a->target_ctx, s.level, s.line);
    }
    pthread_mutex_unlock(&a->mu);
}
```

```c
/* async_thread.c */
#define _GNU_SOURCE
#include "async.h"
#include <errno.h>

static void *writer(void *arg) {
    async_log *a = arg;
    slot s;
    pthread_mutex_lock(&a->mu);
    for (;;) {
        while (!a->q.count && a->q.dropped == a->q.reported && !a->stop)
            pthread_cond_wait(&a->wake, &a->mu);
        if (!ring_pop(&a->q, &s)) break; /* очередь пуста и дана команда остановки */
        a->busy = true;
        pthread_mutex_unlock(&a->mu);
        a->target->write(a->target_ctx, s.level, s.line);
        pthread_mutex_lock(&a->mu);
        a->busy = false;
        pthread_cond_broadcast(&a->idle);
    }
    pthread_mutex_unlock(&a->mu);
    return nullptr;
}

int async_start(async_log *a) {
    async_atfork(a);
    pthread_mutex_lock(&a->mu);
    int rc = a->state == AS_NEW ? pthread_create(&a->th, NULL, writer, a) : EINVAL;
    if (rc == 0) a->state = AS_RUNNING;
    pthread_mutex_unlock(&a->mu);
    return -rc;
}
```

Писатель берёт запись под мьютексом, а пишет в приёмник уже без него, поэтому производители не ждут диска; `busy` нужен, чтобы `async_flush` не считал очередь сброшенной, пока последняя запись ещё в пути.

```c
/* async_stop.c */
#define _GNU_SOURCE
#include "async.h"

void async_stop(async_log *a) {
    pthread_mutex_lock(&a->mu);
    bool running = a->state == AS_RUNNING;
    a->stop = true;
    pthread_cond_signal(&a->wake);
    pthread_mutex_unlock(&a->mu);
    if (running) pthread_join(a->th, NULL);
    pthread_mutex_lock(&a->mu);
    a->state = AS_SYNC;
    pthread_mutex_unlock(&a->mu);
    async_flush(a); /* остаток, пришедший между выходом писателя и сменой режима */
}

void async_destroy(async_log *a) {
    ring_free(&a->q);
    pthread_cond_destroy(&a->idle);
    pthread_cond_destroy(&a->wake);
    pthread_mutex_destroy(&a->mu);
}
```

```c
/* async_fork.c */
#define _GNU_SOURCE
#include "async.h"

static async_log *g_inst;

static void at_prepare(void) { pthread_mutex_lock(&g_inst->mu); }
static void at_parent(void) { pthread_mutex_unlock(&g_inst->mu); }
static void at_child(void) { /* потомок: потока-писателя нет, очередь принадлежит родителю */
    async_log *a = g_inst;
    if (a->state == AS_RUNNING) a->state = AS_SYNC;
    a->busy = false;
    a->q.count = 0;
    a->q.reported = a->q.dropped;
    pthread_mutex_init(&a->mu, NULL);
    pthread_cond_init(&a->wake, NULL);
    pthread_cond_init(&a->idle, NULL);
}

static void install(void) { pthread_atfork(at_prepare, at_parent, at_child); }

void async_atfork(async_log *a) {
    static pthread_once_t once = PTHREAD_ONCE_INIT;
    g_inst = a;
    pthread_once(&once, install);
}
```

Обработчики `fork` вынесены в отдельный файл. Деление на три состояния закрывает три случая сразу: до `async_start` записи копятся, после остановки и в потомке `fork` логгер пишет прямо в приёмник. Мьютекс берётся на время `fork`, чтобы потомок не унаследовал его захваченным другим потоком.

```c
/* main.c */
#include "async.h"
#include "fixed.h"
#include <unistd.h>

static log_module app = {"app", LV_INFO};
static async_log a;

int main(void) {
    int out = STDOUT_FILENO;
    static const log_ops direct = {log_fd_write, fixed_now};
    static const log_ops queued = {async_write, fixed_now};
    if (async_init(&a, &direct, &out, 2) != 0) return 1;
    log_init(&queued, &a);
    log_register(&app);
    LOG_KV(&app, LV_INFO, "tick", KV_INT("i", 0)); /* вывод: ts=2023-11-14T22:13:20.123Z level=info mod=app src=main.c:16 msg="tick" i=0 */
    LOG_KV(&app, LV_INFO, "tick", KV_INT("i", 1)); /* вывод: ts=2023-11-14T22:13:20.123Z level=info mod=app src=main.c:17 msg="tick" i=1 */
    LOG_KV(&app, LV_INFO, "tick", KV_INT("i", 2)); /* очередь на два слота полна: потеря */
    async_start(&a);
    async_stop(&a); /* вывод: msg="log overflow" lost=1 */
    async_destroy(&a);
    return 0;
}
```

Пример запускает писателя после трёх записей, поэтому переполнение и счётчик потерь детерминированы: первые две записи дошли, третья посчитана как потерянная.

```c
/* main.c */
#include "async.h"
#include "fixed.h"
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

static log_module app = {"app", LV_INFO};
static async_log a;

int main(void) {
    int out = STDOUT_FILENO;
    static const log_ops direct = {log_fd_write, fixed_now};
    static const log_ops queued = {async_write, fixed_now};
    if (async_init(&a, &direct, &out, 64) != 0 || async_start(&a) != 0) return 1;
    log_init(&queued, &a);
    log_register(&app);
    LOG_KV(&app, LV_INFO, "before"); /* вывод: ts=2023-11-14T22:13:20.123Z level=info mod=app src=main.c:18 msg="before" */
    async_flush(&a); /* без сброса порядок записей родителя и потомка не определён */
    if (fork() == 0) {
        LOG_KV(&app, LV_INFO, "child"); /* вывод: ts=2023-11-14T22:13:20.123Z level=info mod=app src=main.c:21 msg="child" */
        _exit(0);
    }
    wait(NULL);
    LOG_KV(&app, LV_INFO, "parent"); /* вывод: ts=2023-11-14T22:13:20.123Z level=info mod=app src=main.c:25 msg="parent" */
    LOG_KV(&app, LV_FATAL, "fatal"); /* вывод: ts=2023-11-14T22:13:20.123Z level=fatal mod=app src=main.c:26 msg="fatal" */
    _exit(0); /* без async_stop: запись FATAL уже сброшена */
}
```

В потомке `fork` потока-писателя нет, и без обработчиков `pthread_atfork` запись ушла бы в очередь, из которой никто не читает; здесь потомок переключается на прямую запись. Запись уровня `LV_FATAL` минует очередь: сначала `async_flush` ждёт, пока записано всё принятое, затем запись идёт прямо в приёмник, поэтому следующий за ней `abort` или `_exit` не потеряет ни принятый хвост, ни саму фатальную запись даже при полной очереди.

В обработчике сигнала очередь недоступна: мьютекс и копирование в неё не async-signal-safe. Единственное, что допустимо, - `write` заранее подготовленной строки; обычный путь - флаг и запись в основном цикле (глава про [системный уровень](13-system-io.md)).

```c
#define _GNU_SOURCE
#include <errno.h>
#include <signal.h>
#include <unistd.h>

static void on_term(int sig) {
    static const char msg[] = "level=fatal msg=\"terminated by signal\"\n";
    int saved = errno;
    (void)sig;
    ssize_t r = write(STDOUT_FILENO, msg, sizeof msg - 1);
    (void)r;
    errno = saved;
}

int main(void) {
    struct sigaction sa = {.sa_handler = on_term};
    sigaction(SIGTERM, &sa, NULL);
    raise(SIGTERM); /* вывод: level=fatal msg="terminated by signal" */
    return 0;
}
```

#### Подводные камни

- Падение по сигналу (`SIGSEGV`) теряет очередь: сброс по `LV_FATAL` покрывает только путь, где программа сама решила умереть. Для аварий используй обработчик, пишущий подготовленную строку прямо в дескриптор, и дамп памяти.
- Время записи - момент постановки в очередь, а не записи на диск; порядок записей разных потоков определяет очередь.
- Приёмник за декоратором вызывается из потока-писателя, из потока вызывающего при записи `LV_FATAL`, а после `async_stop` и в потомке `fork` из любого потока: он должен быть потокобезопасным (приёмники из предыдущего раздела такие).
- Экземпляр один на процесс и со статическим временем жизни: обработчики `pthread_atfork` снять нельзя, а они держат указатель на него.
- Память очереди - `cap` килобайт (слот равен максимальной записи); считай её при выборе `cap`, а при потерях увеличивай очередь или ускоряй приёмник, а не блокируй производителя.
- Порядок остановки: сначала перестают логировать производители (или принимают, что поздние записи пойдут синхронно), затем `async_stop`, затем закрывается настоящий приёмник (файл, сокет), и только потом `async_destroy`.

## Коротко

### Выбирай между своим логгером и готовой библиотекой

Своего логгера из первой части хватает на 100 строк и ничего не тянет; готовая библиотека нужна, когда нужны её приёмники и настройка без перекомпиляции. В образе книги собираются GLib (`g_log`), `syslog` и `sd_journal_send`; остальные строки таблицы в образ не входят и здесь не проверялись.

| Вариант | Класс | Где взять | Главный контракт |
| --- | --- | --- | --- |
| Свой макрос и `log_write` | простой | свой код | один `write` на запись, уровень отсекается при сборке |
| log.c (библиотека rxi) | простой | вне apt, исходники копируются в проект | макросы по уровням и приёмники-обратные вызовы (в образе не собирался) |
| GLib `g_log` | мощный | `libglib2.0-dev` | домен `G_LOG_DOMAIN`, глобальный или доменный обработчик, структурированные поля |
| `syslog(3)` | мощный | libc | `openlog`, `syslog(prio, "%s", ...)`, `setlogmask`; запись уходит в `/dev/log` |
| journald | мощный | `libsystemd-dev` | `sd_journal_send` с полями `ИМЯ=значение`, либо префикс `<N>` в выводе юнита |
| zlog | мощный | вне apt | категории и правила вывода из файла настроек (в образе не собирался) |

### Подключай GLib g_log одним обработчиком

`g_warning`, `g_message` и `g_debug` пишут в домен `G_LOG_DOMAIN`; обработчик по умолчанию печатает `g_message` в stderr, а `g_debug` в stdout и скрывает `g_debug`, пока переменная окружения `G_MESSAGES_DEBUG` не содержит домен или `all`. `g_log_set_default_handler` подменяет этот обработчик своим, и тогда фильтр `G_MESSAGES_DEBUG` не действует: свой обработчик получает и отладочные сообщения (проверено), так что фильтровать по уровню нужно в нём самом. В примере сообщение по-английски: `g_print` в локали `C` заменяет не-ASCII символы на `?`.

```c
#define G_LOG_DOMAIN "app"
#include <glib.h>

static void h(const gchar *domain, GLogLevelFlags level, const gchar *msg, gpointer user) {
    (void)level; (void)user;
    g_print("%s: %s\n", domain, msg);
}

int main(void) {
    g_log_set_default_handler(h, NULL);
    g_warning("disk %d%% full", 91); /* вывод: app: disk 91% full */
    return 0;
}
```

## Что ещё стоит знать

OpenTelemetry связывает логи с трассами и метриками одним идентификатором запроса (trace id): его кладут в каждую запись как обычное поле. Агент сбора вроде fluent-bit читает stdout контейнера или файл и пересылает записи в хранилище, поэтому приложению не нужен сетевой приёмник. Автоматическую регистрацию модулей логгера через секцию и конструкторы можно сделать по схеме из глав про [возможности компилятора](08-compiler-features.md) и [паттерны](21-patterns.md), а в многопроцессных серверах каждый процесс получает свой экземпляр логгера и общий приёмник.
