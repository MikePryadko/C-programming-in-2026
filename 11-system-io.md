# Системный уровень

## Системные вызовы

### Оборачивай системный вызов в цикл, который доводит работу до конца

`read` и `write` вправе обработать меньше байт, чем просили (канал, сокет, терминал), или вернуть `-1` с `errno == EINTR`, если вызов прервал сигнал, поэтому одиночный вызов теряет данные при нагрузке, которой нет в тесте. Обёртку строят в два слоя: нижняя делает один вызов, повторяет его при `EINTR` и переводит `errno` в `-E...`; верхняя крутит цикл, пока не получит все `n` байт или конец данных. Верхний слой написан над `stream` из [главы про интерфейсы](06-api-and-modules.md), поэтому один цикл проверяется и на настоящем канале, и на тестовом источнике, который отдаёт по байту.

```c
/* io.h */
#ifndef IO_H
#define IO_H
#include <stddef.h>

typedef struct ops { int (*read)(void *ctx, void *buf, size_t n); void (*close)(void *ctx); } ops;
typedef struct stream { const ops *vt; void *ctx; } stream;

/* stream_read - читает до n байт (n <= INT_MAX).
 * Возврат: число прочитанных байт (0 - конец данных) или -E...; close может отсутствовать. */
static inline int stream_read(stream s, void *buf, size_t n) { return s.vt->read(s.ctx, buf, n); }

/* io_read_full - читает, пока не получит n байт или конец данных. n <= INT_MAX.
 * Возврат: число байт (меньше n только при конце данных; 0 - конец данных сразу) или -E...;
 * при ошибке прочитанная часть остаётся в buf, её размер не сообщается. */
int io_read_full(stream s, void *buf, size_t n);

/* io_write_full - пишет все n байт в блокирующий дескриптор. Возврат: 0 или -E.... */
int io_write_full(int fd, const void *buf, size_t n);

/* fd_ops - чтение из дескриптора; ctx указывает на int с номером дескриптора. */
extern const ops fd_ops;
#endif
```

```c
/* io.c */
#define _GNU_SOURCE
#include "io.h"
#include <errno.h>
#include <limits.h>
#include <unistd.h>

static int fd_read(void *ctx, void *buf, size_t n) {
    for (;;) {
        ssize_t r = read(*(int *)ctx, buf, n);
        if (r >= 0) return (int)r;
        if (errno != EINTR) return -errno; /* errno читается сразу после вызова */
    }
}
const ops fd_ops = {.read = fd_read};

int io_read_full(stream s, void *buf, size_t n) {
    if (n > INT_MAX) return -EINVAL;
    char *p = buf;
    size_t got = 0;
    while (got < n) {
        int r = stream_read(s, p + got, n - got);
        if (r < 0) return r;
        if (r == 0) break;
        got += (size_t)r;
    }
    return (int)got;
}

int io_write_full(int fd, const void *buf, size_t n) {
    const char *p = buf;
    while (n > 0) {
        ssize_t w = write(fd, p, n);
        if (w < 0 && errno == EINTR) continue;
        if (w < 0) return -errno;
        p += w;
        n -= (size_t)w;
    }
    return 0;
}
```

На канале: запись целиком, закрытие записывающего конца, чтение с запасом. Размер буфера больше данных, поэтому цикл завершается по концу данных, а не по счётчику.

```c
/* main.c */
#define _GNU_SOURCE
#include "io.h"
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    int p[2];
    if (pipe2(p, O_CLOEXEC) != 0) return 1;
    if (io_write_full(p[1], "hello, pipe", 11) != 0) return 1;
    close(p[1]);
    char buf[32];
    int n = io_read_full((stream){&fd_ops, &p[0]}, buf, sizeof buf);
    printf("%d %.*s\n", n, n, buf); /* вывод: 11 hello, pipe */
    close(p[0]);
}
```

Тот же `io_read_full` без изменений работает и над тестовым источником с собственной таблицей `ops`, который отдаёт по одному байту ([тесты](07-testing-and-debugging.md)).

#### Подводные камни

- Конец данных раньше `n` не ошибка обёртки. Ноль байт на границе сообщения означает штатное закрытие, недочитанное сообщение означает обрыв; решает вызывающий код, знающий формат.
- Дескриптор неблокирующий. `read` и `write` возвращают `EAGAIN`, и цикл выше превратится в активное ожидание; для таких дескрипторов остаток держат в состоянии и ждут готовности в цикле событий (раздел «События» ниже).
- Запись в канал или сокет, у которого закрыт читающий конец, по умолчанию убивает процесс сигналом `SIGPIPE`. Серверный процесс ставит `SIGPIPE` в игнорирование один раз при старте, и тогда `write` возвращает `-EPIPE`.
- `close` при `EINTR` не повторяют: на Linux дескриптор уже закрыт, и повтор способен закрыть дескриптор, который успел открыть другой поток. Результат `close` проверяют у дескрипторов, в которые писали.
- Один дескриптор читают либо через `read`, либо через `FILE *`: буфер `stdio` уже забрал часть данных, и `read` их не увидит.
- Размер из `read` приводят к `size_t` только после проверки на отрицательное значение ([целые типы и преобразования](02-types-and-ub.md)), а код ошибки вызова обрабатывают по правилам [главы про ошибки](04-errors-and-cleanup.md).

## Бинарные данные

### Читай поля из буфера по одному, а не накладывай на него struct

Формат на проводе и в файле задаёт спецификация, а раскладка `struct` в памяти зависит от выравнивания, padding и порядка байт процессора, поэтому приведение указателя на буфер к `struct *` читает не те байты и нарушает правила доступа. Поля извлекают явно: проверяют, что в буфере хватает байт, затем собирают значение из байт в заданном порядке или копируют `memcpy` в локальную переменную. Побайтовая сборка не зависит ни от выравнивания, ни от порядка байт хоста, а `memcpy` годится только для данных, которые не покидают машину и сборку ([правила доступа и приведение указателей](02-types-and-ub.md)).

Типичная ошибка: накладывать структуру на буфер со смещением.

```c
/* Ошибка: UB - struct наложена на буфер: невыровненный доступ, нарушение aliasing, лишний padding */
#include <stdint.h>
#include <stdio.h>

typedef struct { uint8_t kind; uint32_t len; } hdr;

int main(void) {
    alignas(8) unsigned char buf[16] = {0xEE, 1, 8, 0, 0, 0};
    const hdr *h = (const hdr *)(buf + 1);
    printf("%u\n", h->len);
}
```

`UBSan` сообщает о доступе по невыровненному адресу; на архитектурах, где такой доступ аппаратно запрещён, программа упадёт. Даже при выровненном адресе `len` лежал бы в структуре со смещением 4 (после padding), а в формате со смещением 1. Правильный разбор состоит из функций кодирования и декодирования с проверкой длины до чтения каждого поля:

```c
/* wire.h */
#ifndef WIRE_H
#define WIRE_H
#include <stddef.h>
#include <stdint.h>

enum { MSG_HDR = 5, MSG_MAX = 1024 }; /* формат: kind (1 байт), len (4 байта, little-endian), тело */

typedef struct { uint8_t kind; uint32_t len; const unsigned char *body; } msg;

/* msg_encode - записывает сообщение в out. Возврат: число байт, -EMSGSIZE (len > MSG_MAX) или -ENOSPC. */
int msg_encode(unsigned char *out, size_t cap, uint8_t kind, const void *body, uint32_t len);

/* msg_decode - разбирает одно сообщение из начала buf; buf заимствуется, m->body указывает внутрь buf.
 * Возврат: число использованных байт, 0 - сообщение неполное (нужно больше данных), -EBADMSG - len > MSG_MAX. */
int msg_decode(const unsigned char *buf, size_t n, msg *m);
#endif
```

```c
/* wire.c */
#include "wire.h"
#include <errno.h>
#include <string.h>

static uint32_t get_u32le(const unsigned char *p) {
    return (uint32_t)p[0] | (uint32_t)p[1] << 8 | (uint32_t)p[2] << 16 | (uint32_t)p[3] << 24;
}

static void put_u32le(unsigned char *p, uint32_t v) {
    for (int i = 0; i < 4; i++) p[i] = (unsigned char)(v >> (8 * i));
}

int msg_encode(unsigned char *out, size_t cap, uint8_t kind, const void *body, uint32_t len) {
    if (len > MSG_MAX) return -EMSGSIZE;
    if (cap < MSG_HDR + len) return -ENOSPC;
    out[0] = kind;
    put_u32le(out + 1, len);
    if (len) memcpy(out + MSG_HDR, body, len);
    return (int)(MSG_HDR + len);
}

int msg_decode(const unsigned char *buf, size_t n, msg *m) {
    if (n < MSG_HDR) return 0; /* длина проверяется до чтения поля */
    uint32_t len = get_u32le(buf + 1);
    if (len > MSG_MAX) return -EBADMSG;
    if (n - MSG_HDR < len) return 0;
    *m = (msg){.kind = buf[0], .len = len, .body = buf + MSG_HDR};
    return (int)(MSG_HDR + len);
}
```

Сообщение намеренно лежит с нечётного смещения: побайтовое чтение этого не замечает.

```c
/* main.c */
#include "wire.h"
#include <errno.h>
#include <stdio.h>

int main(void) {
    unsigned char buf[64] = {0xEE}; /* байт перед сообщением: адрес сообщения невыровнен */
    int n = msg_encode(buf + 1, sizeof buf - 1, 7, "hi!", 3);
    msg m;
    int used = msg_decode(buf + 1, (size_t)n, &m);
    printf("%d %d %u %.*s\n", n, used, (unsigned)m.kind, (int)m.len, (const char *)m.body); /* вывод: 8 8 7 hi! */
    printf("%d\n", msg_decode(buf + 1, (size_t)n - 1, &m)); /* вывод: 0 */
    buf[5] = 0xFF; /* старший байт длины: заявленный размер больше MSG_MAX */
    printf("%d\n", msg_decode(buf + 1, (size_t)n, &m) == -EBADMSG); /* вывод: 1 */
}
```

#### Подводные камни

- Порядок байт выбирают один раз и записывают в спецификацию формата. `memcpy` целого записывает порядок хоста, поэтому для проводов и файлов обмена используют побайтовую сборку; сетевой порядок байт (big-endian) собирается тем же способом со сдвигами в другую сторону.
- `fwrite(&s, sizeof s, 1, f)` записывает padding и порядок хоста, и файл не читается после смены компилятора, флагов или платформы. Структуру сериализуют по полям теми же функциями.
- Длина, пришедшая из данных, недоверенная: её сравнивают с пределом до выделения памяти и до копирования, а сложение длин считают с проверкой переполнения (`ckd_add`, [целые типы](02-types-and-ub.md)).
- Текст хранят в `char`, а байты формата читают и хранят как `unsigned char` (`uint8_t`): значение `char` со знаком расширяется и ломает сдвиги и сравнения.
- Битовые поля `struct` для форматов не годятся: их раскладка определяется реализацией. Флаги хранят в целом типе фиксированной ширины (`uint32_t`) и выделяют масками.
- Значение перечисления из данных читают как целое и проверяют на допустимость до приведения к `enum`.

## Платформенный слой

### Прячь системные вызовы за собственным интерфейсом

Код, который напрямую вызывает `clock_gettime`, `open` и `read`, нельзя проверить без реального времени и реальных файлов, а переносить его на другую платформу приходится целиком, поэтому все обращения к ОС собирают в одном файле за узким интерфейсом. Интерфейс описывает потребности приложения, а не зеркалит API ОС: здесь это монотонные часы и чтение файла целиком. Устроен он как таблица операций из [главы про интерфейсы](06-api-and-modules.md); подмена реализации в тестах (поддельные часы и файлы) и внедрение зависимостей разобраны в [главе про паттерны](14-patterns.md).

```c
/* sys.h */
#ifndef SYS_H
#define SYS_H
#include <stddef.h>
#include <stdint.h>

typedef struct sys_ops {
    int64_t (*now_ms)(void *ctx);
    int (*read_file)(void *ctx, const char *path, void *buf, size_t cap);
} sys_ops;
typedef struct sys { const sys_ops *vt; void *ctx; } sys;

/* sys_now_ms - монотонное время в миллисекундах; точка отсчёта произвольна, годится только для интервалов. */
static inline int64_t sys_now_ms(sys s) { return s.vt->now_ms(s.ctx); }

/* sys_read_file - читает до cap байт файла в buf. Возврат: число байт или -E.... */
static inline int sys_read_file(sys s, const char *path, void *buf, size_t cap) {
    return s.vt->read_file(s.ctx, path, buf, cap);
}

extern const sys_ops posix_ops; /* настоящая реализация: sys_posix.c */
#endif
```

Все заголовки ОС подключает только файл реализации; ошибки переводятся в `-E...` внутри него.

```c
/* sys_posix.c */
#define _GNU_SOURCE
#include "sys.h"
#include <errno.h>
#include <fcntl.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

static int64_t posix_now_ms(void *ctx) {
    (void)ctx;
    struct timespec ts;
    if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0) abort();
    return (int64_t)ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}

static int posix_read_file(void *ctx, const char *path, void *buf, size_t cap) {
    (void)ctx;
    int fd = open(path, O_RDONLY | O_CLOEXEC);
    if (fd < 0) return -errno;
    size_t got = 0;
    int rc = 0;
    while (got < cap && rc == 0) {
        ssize_t r = read(fd, (char *)buf + got, cap - got);
        if (r > 0) got += (size_t)r;
        else if (r == 0) break;
        else if (errno != EINTR) rc = -errno;
    }
    close(fd);
    return rc ? rc : (int)got;
}

const sys_ops posix_ops = {posix_now_ms, posix_read_file};
```

Прикладная функция получает `sys` и видит только `sys.h`: ни `unistd.h`, ни `time.h` в ней нет, а настоящую реализацию подставляет `main`.

```c
/* main.c */
#define _GNU_SOURCE
#include "sys.h"
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static int load(sys s, const char *path, char *buf, size_t cap, int64_t *ms) {
    int64_t t0 = sys_now_ms(s);
    int n = sys_read_file(s, path, buf, cap);
    *ms = sys_now_ms(s) - t0;
    return n;
}

int main(void) {
    char path[] = "/tmp/sysXXXXXX";
    int fd = mkstemp(path);
    if (fd < 0) return 1;
    if (write(fd, "real", 4) != 4) return 1;
    close(fd);
    char buf[16];
    int64_t ms;
    int n = load((sys){&posix_ops, nullptr}, path, buf, sizeof buf, &ms);
    unlink(path);
    printf("%d %d\n", n, ms >= 0); /* вывод: 4 1 */
}
```

#### Подводные камни

- Интерфейс повторяет API ОС (`open`, `read`, `close`, `fstat`). Тестовую реализацию придётся писать почти целиком; описывай то, что нужно приложению («прочитай файл», «который час»).
- Ошибки ОС выходят наружу как `errno` или код платформы. Слой переводит их в `-E...` ([ошибки](04-errors-and-cleanup.md)), иначе вызывающий код снова зависит от платформы.
- `#ifdef` платформы размазан по вызывающему коду. Реализацию выбирает сборка (`sys_posix.c` или файл другой платформы), а условная компиляция остаётся внутри файла реализации.
- Таблица операций нужна, когда подмена происходит во время работы; если реализация одна на сборку, достаточно функций с одним именем в разных `.c` файлах.

## События

### Строй цикл событий на неблокирующем вводе-выводе и poll

Поток, блокирующийся в `read` на одном соединении, не обслужит остальные, поэтому дескрипторы переводят в неблокирующий режим (`O_NONBLOCK`), а ждёт готовности один вызов `poll` на все сразу. Неблокирующий `read` и `write` возвращают `-1` с `EAGAIN`, когда делать нечего: это не ошибка, а сигнал вернуться в `poll`. Состояние каждого соединения хранится в структуре и меняется конечным автоматом ([автомат](14-patterns.md)): читает, пока не получит данные, затем пишет, пока не отправит всё, затем снова читает; срок простоя считается на монотонных часах, а ожидание в `poll` ограничено ближайшим сроком. `poll` переносим между POSIX-системами; когда соединений тысячи, его заменяют бэкендами `epoll` (Linux), `kqueue` (BSD, macOS) и `IOCP` (Windows), а автомат соединения остаётся прежним. В примере `SOCK_NONBLOCK` и `SOCK_CLOEXEC` в `socketpair` и `CLOCK_MONOTONIC` расширения Linux.

Пример: эхо-сервер на трёх парах `socketpair`.

```c
/* echo.h */
#ifndef ECHO_H
#define ECHO_H
#include <stddef.h>
#include <stdint.h>

typedef enum { CONN_READ, CONN_WRITE, CONN_DONE } conn_state;

typedef struct {
    int fd;
    conn_state st;
    int64_t deadline; /* монотонное время, мс */
    char buf[64];
    size_t len, off; /* buf[off..len) ещё не отправлено */
} conn;

int64_t echo_now_ms(void);

/* conn_step - двигает автомат после готовности дескриптора. Возврат: 0 или -E...;
 * конец данных переводит соединение в CONN_DONE. */
int conn_step(conn *c);

/* echo_run - обслуживает n <= 16 соединений до закрытия всех; закрытие по ошибке и по простою idle_ms.
 * Возврат: 0 или -E.... */
int echo_run(conn *cs, size_t n, int idle_ms);
#endif
```

```c
/* conn.c */
#define _GNU_SOURCE
#include "echo.h"
#include <errno.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

int64_t echo_now_ms(void) {
    struct timespec ts;
    if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0) abort();
    return (int64_t)ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}

int conn_step(conn *c) {
    if (c->st == CONN_READ) {
        ssize_t r = read(c->fd, c->buf, sizeof c->buf);
        if (r < 0) return errno == EAGAIN || errno == EINTR ? 0 : -errno;
        if (r == 0) { c->st = CONN_DONE; return 0; }
        c->len = (size_t)r;
        c->off = 0;
        c->st = CONN_WRITE;
    }
    while (c->st == CONN_WRITE && c->off < c->len) {
        ssize_t w = write(c->fd, c->buf + c->off, c->len - c->off);
        if (w < 0) return errno == EAGAIN || errno == EINTR ? 0 : -errno;
        c->off += (size_t)w;
    }
    if (c->st == CONN_WRITE) c->st = CONN_READ; /* всё отправлено */
    return 0;
}
```

Цикл собирает набор `pollfd`, считает время ожидания по ближайшему сроку и после возврата из `poll` двигает автомат либо закрывает простаивающее соединение. Закрытые записи получают `fd = -1`: `poll` такие игнорирует.

```c
/* loop.c */
#define _GNU_SOURCE
#include "echo.h"
#include <errno.h>
#include <poll.h>
#include <unistd.h>

int echo_run(conn *cs, size_t n, int idle_ms) {
    struct pollfd pf[16];
    if (n > 16) return -EINVAL;
    for (size_t i = 0; i < n; i++) cs[i].deadline = echo_now_ms() + idle_ms;
    for (;;) {
        int64_t next = INT64_MAX;
        for (size_t i = 0; i < n; i++) {
            bool on = cs[i].st != CONN_DONE;
            pf[i] = (struct pollfd){.fd = on ? cs[i].fd : -1, .events = cs[i].st == CONN_READ ? POLLIN : POLLOUT};
            if (on && cs[i].deadline < next) next = cs[i].deadline;
        }
        if (next == INT64_MAX) return 0;
        int64_t wait = next - echo_now_ms();
        if (poll(pf, n, wait < 0 ? 0 : (int)wait) < 0 && errno != EINTR) return -errno;
        int64_t now = echo_now_ms();
        for (size_t i = 0; i < n; i++) {
            conn *c = &cs[i];
            if (c->st == CONN_DONE) continue;
            if (pf[i].revents && conn_step(c) == 0) c->deadline = now + idle_ms;
            else if (pf[i].revents || now >= c->deadline) c->st = CONN_DONE;
            if (c->st == CONN_DONE) { close(c->fd); c->fd = -1; }
        }
    }
}
```

Два клиента отправляют данные и закрывают запись, третий молчит и отключается по простою. Считаются принятые каждым эхо-байты, поэтому вывод не зависит от порядка обработки.

```c
/* main.c */
#define _GNU_SOURCE
#include "echo.h"
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

int main(void) {
    enum { N = 3 };
    const char *msg[N] = {"alpha", "beta", nullptr}; /* третий клиент молчит */
    conn cs[N];
    int cli[N];
    for (int i = 0; i < N; i++) {
        int sv[2];
        if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, sv) != 0) return 1;
        cs[i] = (conn){.fd = sv[0], .st = CONN_READ};
        cli[i] = sv[1];
        if (msg[i] && (write(cli[i], msg[i], strlen(msg[i])) < 0 || shutdown(cli[i], SHUT_WR) != 0)) return 1;
    }
    if (echo_run(cs, N, 50) != 0) return 1;
    long got[N];
    for (int i = 0; i < N; i++) {
        char buf[16];
        got[i] = (long)read(cli[i], buf, sizeof buf);
        close(cli[i]);
    }
    printf("%ld %ld %ld\n", got[0], got[1], got[2]); /* вывод: 5 4 0 */
}
```

#### Подводные камни

- `POLLHUP`, `POLLERR` и `POLLNVAL` `poll` выставляет без запроса, и они не исчезают, пока дескриптор не закрыт. Цикл, который на них не реагирует, получает мгновенный возврат из `poll` на каждом обороте и занимает процессор целиком; здесь реакция заложена в автомат: после любого `revents` он пробует `read` или `write`, получает `0` или ошибку и закрывает соединение.
- Неотправленный остаток после `EAGAIN` хранят в самом соединении (`off`, `len`), а не повторяют вызов в цикле: это активное ожидание.
- Автомат не читает, пока не отправил ответ: медленный клиент замедляет только себя, а память соединения ограничена размером `buf`. Чтение «про запас» без предела превращается в неограниченный рост памяти.
- Срок простоя считают на монотонных часах (см. «Коротко»), иначе перевод системных часов ломает таймауты; соединение без срока удерживает дескриптор вечно.
- `poll` не годится для обычных файлов: файл на диске всегда «готов». Для него используют потоки ([потоки](10-concurrency.md)) или `io_uring`.

## Коротко

### В обработчике сигнала выставляй флаг и ничего больше

Обработчик прерывает код в произвольной точке, в том числе внутри `malloc` или `printf`, поэтому он только записывает значение в `volatile sig_atomic_t` (или вызывает `write`, `_exit`), а работу делает основной цикл. Обработчик ставят `sigaction`, а не `signal`: поведение `signal` различается между системами (перезапуск вызовов, сброс обработчика). Чтобы `poll` проснулся от сигнала, пришедшего между проверкой флага и входом в него, обработчик пишет байт в неблокирующий self-pipe, который `poll` слушает вместе с остальными дескрипторами; `write` меняет `errno` прерванного кода, поэтому обработчик сохраняет и восстанавливает его. Типичная ошибка: `printf` и `malloc` в обработчике (сигнал во время другого `printf` зависает или портит вывод) и счётчик `count++` вместо флага (два шага чтения и записи теряют приращение). Из обработчика `SIGSEGV`, `SIGFPE`, `SIGBUS` возвращаться нельзя.

```c
#define _GNU_SOURCE
#include <signal.h>

static volatile sig_atomic_t stop;

static void on_term(int sig) { (void)sig; stop = 1; }

int install(void) {
    struct sigaction sa = {.sa_handler = on_term};
    sigemptyset(&sa.sa_mask);
    return sigaction(SIGTERM, &sa, nullptr);
}

bool should_stop(void) { return stop; }
```

### Ставь O_CLOEXEC на каждый дескриптор и закрывай его сам

По умолчанию дескриптор наследуется при `exec`, и запущенная программа получает сокеты и файлы, которые ей не предназначены, а заодно удерживает соединения и блокировки. Флаг указывают при создании: `O_CLOEXEC` в `open` и `pipe2`, `SOCK_CLOEXEC` в `socket`, `socketpair` и `accept4` (`pipe2` и `accept4` расширения Linux и BSD); отдельный `fcntl(FD_CLOEXEC)` после создания оставляет окно гонки с `fork` в другом потоке. Закрытый дескриптор сразу заменяют на `-1`: номер освободился и достанется следующему `open`, и запись по старому номеру попадёт в чужой файл. Закрытие дескрипторов входит в [очистку на всех путях выхода](04-errors-and-cleanup.md).

### Не запускай процессы через system

`system` и `popen` передают строку оболочке, поэтому любой фрагмент недоверенного ввода в ней (пробел, `;`, `$(...)`, кавычка) превращается в команду: это инъекция ([безопасность](12-security.md)).

```c
/* Ошибка: имя из недоверенного ввода попадает в командную строку оболочки */
#include <stdio.h>
#include <stdlib.h>

int pack(const char *name) {
    char cmd[256];
    snprintf(cmd, sizeof cmd, "tar cf out.tar %s", name); /* name = "x; rm -rf ~" */
    return system(cmd);
}
```

Вместо этого вызывают `posix_spawn` (или `fork` и `execve`) с массивом аргументов: каждый аргумент передаётся программе как есть, оболочка не участвует. Путь к программе указывают абсолютным, а окружение передают явным списком `envp`, а не унаследованным `environ` с недоверенным `PATH` ([безопасность](12-security.md)). `posix_spawn` возвращает код ошибки, а не `-1` с `errno`, а завершение порождённого процесса нужно принять через `waitpid`, иначе останется зомби.

```c
#define _GNU_SOURCE
#include <errno.h>
#include <spawn.h>
#include <sys/wait.h>

/* run - запускает prog (абсолютный путь) без оболочки и с пустым окружением. Возврат: код выхода или -1. */
int run(const char *prog, char *const argv[]) {
    pid_t pid;
    char *const envp[] = {nullptr};
    if (posix_spawn(&pid, prog, nullptr, nullptr, argv, envp) != 0) return -1;
    int st;
    while (waitpid(pid, &st, 0) < 0)
        if (errno != EINTR) return -1;
    return WIFEXITED(st) ? WEXITSTATUS(st) : -1;
}
```

Если нужен именно `fork`, перед ним вызывают `fflush(nullptr)`: непустой буфер `stdio` копируется в дочерний процесс, и данные выйдут дважды. Дочерний процесс до `exec` вызывает только функции, безопасные для асинхронного вызова, и завершается через `_exit`, а не `exit`, который ещё раз сбросит унаследованные буферы.

### Для интервалов используй монотонные часы

`clock_gettime(CLOCK_MONOTONIC, ...)` не скачет при коррекции времени и ручном переводе часов, поэтому им меряют таймауты, сроки годности и длительности; точка отсчёта произвольна, значения вычитают друг из друга и не сохраняют между запусками. `CLOCK_REALTIME` и `time` показывают календарное время и годятся только для меток («когда это произошло»): перевод часов назад даёт отрицательные интервалы и таймауты, которые не срабатывают. Стандартный `timespec_get` в glibc 2.39 принимает только базу `TIME_UTC` (календарное время), поэтому часы прячут за платформенным интерфейсом, как в разделе «Платформенный слой».

### Создавай временные файлы через mkstemp и работай по дескриптору

Схема «проверить имя, потом открыть по имени» (`access` или `stat` перед `open`, `tmpnam`, `mktemp`) оставляет окно, в которое другой процесс подставит символьную ссылку на чужой файл: это TOCTOU. `mkstemp` создаёт файл атомарно (`O_EXCL`, права `0600`) и возвращает уже открытый дескриптор; дальнейшие проверки идут по дескриптору (`fstat`, `fchmod`), а не по имени, и файл можно сразу удалить `unlink`: пока дескриптор открыт, данные доступны. Для файлов в каталогах, которые контролирует кто-то другой, открывают через `openat` от дескриптора каталога с `O_NOFOLLOW` и `O_EXCL` и смотрят на результат открытия, а не проверяют заранее.

### Проверяй fflush, fclose и ferror

`fwrite` и `fprintf` пишут в буфер, поэтому ошибка диска или закрытого канала часто проявляется только при `fflush` или `fclose`: файл, записанный без проверки `fclose`, может оказаться усечённым при «успешном» завершении. После записи проверяют `fflush(f) != 0 || ferror(f)`, затем `fclose(f) != 0`; когда `fread` вернул меньше запрошенного, различают `feof` и `ferror`. Стандартный вывод к терминалу буферизуется построчно, в канал или файл целиком, поэтому диагностику перед аварийным завершением пишут в `stderr` (он без буфера) или вызывают `fflush`.

### Сохраняй файл атомарно: временный файл, fsync, rename

Запись прямо в целевой файл при сбое посреди записи оставляет обрезанный файл и теряет старое содержимое. Данные пишут во временный файл в том же каталоге (`rename` атомарен только внутри одной файловой системы), сбрасывают на диск `fsync`, переименовывают поверх целевого и сбрасывают `fsync` каталога, чтобы переименование пережило отключение питания: читатель видит либо старую версию целиком, либо новую целиком.

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int save_atomic(const char *dir, const char *name, const char *data, size_t n) {
    char tmp[256], dst[256];
    snprintf(tmp, sizeof tmp, "%s/.%s.XXXXXX", dir, name);
    snprintf(dst, sizeof dst, "%s/%s", dir, name);
    int fd = mkstemp(tmp);
    if (fd < 0) return -1;
    for (size_t off = 0; off < n;) {
        ssize_t w = write(fd, data + off, n - off);
        if (w < 0) { close(fd); unlink(tmp); return -1; }
        off += (size_t)w;
    }
    if (fsync(fd) != 0 || close(fd) != 0 || rename(tmp, dst) != 0) { unlink(tmp); return -1; }
    int dfd = open(dir, O_RDONLY | O_DIRECTORY);
    if (dfd < 0) return -1;
    int rc = fsync(dfd);
    close(dfd);
    return rc;
}

int main(void) {
    char dir[] = "/tmp/saveXXXXXX";
    if (!mkdtemp(dir)) return 1;
    if (save_atomic(dir, "a.txt", "v1", 2) != 0 || save_atomic(dir, "a.txt", "version 2", 9) != 0) return 1;
    char path[256], buf[16] = {0};
    snprintf(path, sizeof path, "%s/a.txt", dir);
    FILE *f = fopen(path, "r");
    if (!f) return 1;
    if (!fgets(buf, sizeof buf, f)) { fclose(f); return 1; }
    fclose(f);
    printf("%s\n", buf); /* вывод: version 2 */
    unlink(path);
    rmdir(dir);
}
```

Сбой записи (`write` вернул `-1`, закончилось место) оставляет целевой файл нетронутым, поэтому функция удаляет временный и сообщает об ошибке. Права и владельца нового файла `mkstemp` выставляет сам (`0600`), для исходных прав вызывают `fchmod` перед `rename`. Временный файл с точкой в имени не мешает обходу каталога, а после аварийного завершения остаётся мусор: при старте приложения такие файлы удаляют. Ссылки и другие имена того же файла после `rename` указывают на старое содержимое.

### Бинарные файлы: fread и fwrite только как нижний слой

Файл открывают в режиме `"rb"` или `"wb"`, `fread` и `fwrite` переносят блоки байт, а поля собирают и разбирают функциями из раздела про бинарные данные: запись структуры целиком сохраняет padding и порядок байт хоста, и такой файл не прочитается другой сборкой или платформой. `fread` возвращает число элементов, а не байт.

### cJSON: JSON одной небольшой библиотекой

`cJSON` (`#include <cjson/cJSON.h>`, линковка `-lcjson`) разбирает и строит JSON-деревья. `cJSON_Parse` возвращает `nullptr` при ошибке; `cJSON_AddItemToArray` и `cJSON_AddItemToObject` передают элемент во владение контейнеру, а при неудаче он остаётся у вызывающего; строку из `cJSON_PrintUnformatted` освобождает вызывающий через `free`, дерево освобождается одним `cJSON_Delete` корня. Элементы читают с проверкой типа (`cJSON_IsNumber`, `cJSON_IsString`), размер и глубину входного документа ограничивают сами; общее дерево не потокобезопасно.

```c
#include <cjson/cJSON.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    cJSON *doc = cJSON_Parse("{\"port\":8080}");
    if (!doc) return 1;
    const cJSON *port = cJSON_GetObjectItemCaseSensitive(doc, "port");
    if (cJSON_IsNumber(port)) printf("%d\n", port->valueint); /* вывод: 8080 */
    char *text = cJSON_PrintUnformatted(doc);
    if (text) puts(text); /* вывод: {"port":8080} */
    free(text);
    cJSON_Delete(doc);
}
```

### libcurl: easy API для HTTP-клиента

`curl_easy_init` создаёт дескриптор запроса (`nullptr` при нехватке памяти), `curl_easy_setopt` задаёт параметры (обязателен `CURLOPT_URL`, срок запроса ограничивают `CURLOPT_TIMEOUT_MS`), блокирующий `curl_easy_perform` выполняет запрос и возвращает `CURLcode`, `curl_easy_cleanup` освобождает дескриптор. Тело ответа приходит в функцию из `CURLOPT_WRITEFUNCTION`: она возвращает число принятых байт, любое другое значение прерывает передачу. `curl_global_init` вызывают один раз при старте, до создания потоков. Функция ниже возвращает `true`, если запрос выполнен, и кладёт размер тела в `*out`.

```c
#include <curl/curl.h>
#include <stddef.h>

static size_t on_body(char *d, size_t s, size_t n, void *u) { (void)d; *(size_t *)u += s * n; return s * n; }

bool fetch_size(const char *url, size_t *out) {
    CURL *h = curl_easy_init();
    if (!h) return false;
    *out = 0;
    curl_easy_setopt(h, CURLOPT_URL, url);
    curl_easy_setopt(h, CURLOPT_WRITEFUNCTION, on_body);
    curl_easy_setopt(h, CURLOPT_WRITEDATA, out);
    curl_easy_setopt(h, CURLOPT_TIMEOUT_MS, 5000L);
    CURLcode rc = curl_easy_perform(h);
    curl_easy_cleanup(h);
    return rc == CURLE_OK;
}
```

### libuv: готовый цикл событий

`libuv` (`#include <uv.h>`, линковка `-luv`; под `-std=c23` перед включением заголовка нужен `#define _GNU_SOURCE`) даёт единый цикл событий над `epoll`, `kqueue` и `IOCP` с TCP, UDP, таймерами, DNS, процессами, файлами и пулом потоков, поэтому писать цикл самому нужно только для учебных и узких случаев. Функции возвращают `0` или отрицательный `UV_E...`; `uv_run(loop, UV_RUN_DEFAULT)` работает, пока есть активные дескрипторы и запросы; все обратные вызовы выполняются в потоке цикла; память дескриптора освобождают только в обратном вызове `uv_close`; из другого потока цикл будит `uv_async_send`, остальные вызовы делают из потока цикла.

### Не храни указатель из getenv

Строку из окружения могут перезаписать `setenv` и `putenv`, поэтому значение сразу копируют (`strdup`) и считают недоверенным вводом ([безопасность](12-security.md)).

### Обработчик atexit возвращается нормально

Функция из `atexit` очищает и возвращается, не выходя через `longjmp` или повторный `exit` ([ошибки и очистка](04-errors-and-cleanup.md)).

## Что ещё стоит знать

`io_uring` (Linux, обёртка `liburing`) принимает пакеты операций ввода-вывода через кольцевые очереди и заменяет цикл `poll` там, где важна пропускная способность и асинхронный файловый ввод-вывод. `mmap` отображает файл в память; если другой процесс усечёт файл, обращение к исчезнувшей части даёт `SIGBUS`, поэтому недоверенные и изменяемые файлы читают обычным вводом. `getopt` (POSIX) и `getopt_long` (glibc, BSD) разбирают аргументы командной строки. `libevent` и `libev` предлагают готовые циклы событий наравне с `libuv`; `libevent` дополнительно содержит буферизованные соединения, HTTP и DNS. Случайные байты для секретов дают `getentropy` и `getrandom` ([безопасность](12-security.md)).
