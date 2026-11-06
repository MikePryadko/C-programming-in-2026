# Паттерны

## Зависимости

### Передавай внешний мир в логику интерфейсом, а не вызывай его напрямую

Функция, которая сама читает часы, сокет или диск, проверяется только на настоящих часах, сокете и диске, поэтому всё, что делает логику недетерминированной, передают ей через таблицу операций из [главы про API](06-api-and-modules.md). Боевая реализация подставляется в `main`, тестовая в тесте; фреймворк для этого не нужен. Ниже ограничитель частоты: логика зависит только от интерфейса часов `clk`, а не от `clock_gettime`.

```c
/* clk.h */
#ifndef CLK_H
#define CLK_H
#include <stdint.h>

typedef struct clk_ops { int64_t (*now)(void *ctx); } clk_ops;
typedef struct clk { const clk_ops *vt; void *ctx; } clk;

/* clk_now - монотонное время в миллисекундах. */
static inline int64_t clk_now(clk c) { return c.vt->now(c.ctx); }

/* clk_system - боевые часы (CLOCK_MONOTONIC). */
clk clk_system(void);
#endif
```

Боевая реализация лежит в отдельном файле и единственная знает про ОС. Таблица статическая и константная, контекст ей не нужен.

```c
/* clk_sys.c */
#define _GNU_SOURCE
#include "clk.h"
#include <time.h>

static int64_t sys_now(void *ctx) {
    (void)ctx;
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (int64_t)ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}

static const clk_ops sys_ops = {.now = sys_now};

clk clk_system(void) { return (clk){&sys_ops, nullptr}; }
```

Логика получает зависимость при создании и хранит в своей структуре (контекст); функцию без состояния передают интерфейс параметром.

```c
/* limiter.h */
#ifndef LIMITER_H
#define LIMITER_H
#include "clk.h"
#include <stdbool.h>

typedef struct limiter { clk clock; int limit, used; int64_t window, start; } limiter;

/* limiter_init - не более limit событий за окно window миллисекунд. */
void limiter_init(limiter *l, clk c, int limit, int64_t window);

/* limiter_allow - true, если событие укладывается в лимит (и учитывает его). */
bool limiter_allow(limiter *l);
#endif
```

```c
/* limiter.c */
#include "limiter.h"

void limiter_init(limiter *l, clk c, int limit, int64_t window) {
    *l = (limiter){.clock = c, .limit = limit, .window = window, .start = clk_now(c)};
}

bool limiter_allow(limiter *l) {
    int64_t t = clk_now(l->clock);
    if (t - l->start >= l->window) { l->start = t; l->used = 0; }
    if (l->used >= l->limit) return false;
    l->used++;
    return true;
}
```

Тестовая реализация задаёт время руками: тест двигает часы и проверяет границы окна мгновенно и без `sleep`.

```c
/* main.c */
#include "limiter.h"
#include <stdio.h>

typedef struct { int64_t now; } fake_time;

static int64_t fake_now(void *ctx) { return ((fake_time *)ctx)->now; }
static const clk_ops fake_ops = {.now = fake_now};

int main(void) {
    fake_time ft = {1000};
    limiter l;
    limiter_init(&l, (clk){&fake_ops, &ft}, 2, 100);
    int a = limiter_allow(&l), b = limiter_allow(&l), c = limiter_allow(&l);
    printf("%d%d%d\n", a, b, c); /* вывод: 110 */
    ft.now += 100; /* окно истекло */
    printf("%d\n", limiter_allow(&l)); /* вывод: 1 */

    limiter prod; /* то же, но на боевых часах */
    limiter_init(&prod, clk_system(), 1, 60000);
    printf("%d\n", limiter_allow(&prod)); /* вывод: 1 */
}
```

Как собирать такие тесты с моками и проверками, показано в [главе про тестирование](07-testing-and-debugging.md); приём тот же: тест подставляет свою таблицу операций.

#### Подводные камни

- Один большой интерфейс на всё (часы, диск, сеть, лог в одной структуре `env`) превращает внедрение в объект бога: любая функция зависит от всего и тест обязан подделать всё. Передавай каждой функции только то, что она использует, и дели интерфейсы по назначению.
- Тестовая реализация должна подчиняться тому же контракту, что боевая (коды ошибок, смысл `0` из `read`); иначе тест проходит на заглушке, которая ведёт себя не так, как настоящая зависимость.
- Подмена через `#ifdef TEST` внутри логики не заменяет внедрение: в тесте проверяется другой код, чем в продукте. Подмену на этапе линковки (другой `.c` в тестовой сборке) допускай для зависимостей с одной реализацией на сборку.
- Внедрение не нужно для чистых функций, для стандартных `memcpy` и `strlen` и для зависимостей, у которых нет ни второй реализации, ни побочных эффектов, мешающих тесту. Интерфейс на каждый вызов запутывает код и убирает инлайнинг.
- Владение контекстом остаётся за создателем: интерфейс не освобождает `ctx`, если иное не оговорено в контракте ([владение](03-memory-ownership.md)).

## Данные и состояния

### Моделируй варианты размеченным объединением и обрабатывай все теги

Структура с тегом и объединением хранит ровно один из вариантов и не позволяет забыть, какой именно, поэтому сообщения протокола, узлы выражений и результаты разбора описывают так, а не набором необязательных полей. Тег читают всегда первым, а `switch` по нему пишут без `default`: тогда добавление варианта вызывает предупреждение `-Wswitch` (входит в `-Wall`) во всех местах, где вариант не обработан.

```c
#include <errno.h>
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>

typedef struct { const char *ptr; size_t len; } str;
typedef enum { MSG_PING, MSG_DATA, MSG_ERROR } msg_kind;

typedef struct {
    msg_kind kind;
    union {
        uint32_t ping_id;
        str data;
        int error_code;
    };
} msg;

static msg msg_ping(uint32_t id) { return (msg){.kind = MSG_PING, .ping_id = id}; }
static msg msg_data(str s) { return (msg){.kind = MSG_DATA, .data = s}; }
static msg msg_error(int code) { return (msg){.kind = MSG_ERROR, .error_code = code}; }

static int msg_describe(msg m, char *buf, size_t n) {
    switch (m.kind) {
    case MSG_PING:  return snprintf(buf, n, "ping %u", (unsigned)m.ping_id);
    case MSG_DATA:  return snprintf(buf, n, "data %zu bytes", m.data.len);
    case MSG_ERROR: return snprintf(buf, n, "error %d", m.error_code);
    }
    abort(); /* тег повреждён: сюда попасть нельзя */
}

int main(void) {
    char buf[32];
    msg_describe(msg_ping(7), buf, sizeof buf);
    puts(buf); /* вывод: ping 7 */
    msg_describe(msg_data((str){"hello", 5}), buf, sizeof buf);
    puts(buf); /* вывод: data 5 bytes */
    msg_describe(msg_error(EINVAL), buf, sizeof buf);
    puts(buf); /* вывод: error 22 */
}
```

Конструкторы вариантов (`msg_ping`, `msg_data`) единственное место, где пишут одновременно тег и поле, а доступ к варианту идёт только внутри `case` своего тега или после проверки тега. Код после `switch` нужен, потому что тег может содержать мусор; `abort()` честно это выражает; `unreachable()` здесь не годится: при мусорном теге он даёт неопределённое поведение, а годится он только там, где недостижимость доказана построением ([типы и UB](02-types-and-ub.md)).

Ловушка, которую закрывает отсутствие `default`:

```c
/* Ошибка: default скрывает необработанный вариант, компилятор молчит */
#include <stdio.h>

typedef enum { MSG_PING, MSG_DATA, MSG_ERROR, MSG_CLOSE } msg_kind; /* MSG_CLOSE добавлен позже */

int describe(msg_kind k) {
    switch (k) {
    case MSG_PING:  return 1;
    case MSG_DATA:  return 2;
    case MSG_ERROR: return 3;
    default:        return 0; /* MSG_CLOSE попадает сюда без предупреждения */
    }
}
```

#### Подводные камни

- Чтение не того члена объединения по правилам языка допустимо только как переинтерпретация представления и даёт неопределённое значение для невалидного представления; обращайся к члену, соответствующему тегу, и больше ни к какому.
- `default` убирает предупреждение для всех будущих вариантов. Нужна общая ветка для «прочих», перечисляй варианты явно или добавь проверку после `switch`; флаг `-Wswitch-enum` требует перечисления всех констант даже при наличии `default`, а у Clang `-Wcovered-switch-default` ругается на `default`, когда все константы уже перечислены.
- Нулевая инициализация `msg m = {}` даёт тег 0, то есть первый вариант: сделай его осмысленным «пустым» значением или введи вариант `MSG_NONE = 0`.
- Объединение занимает размер самого большого варианта; крупные варианты (массивы, буферы) держи по указателю, иначе каждое сообщение платит за самый большой.
- Вариант, владеющий памятью (строка, буфер), требует функции освобождения с `switch` по тегу; без неё утечки в одних вариантах незаметны.

### Описывай автомат таблицей переходов

Таблица `состояние x событие -> следующее состояние` делает допустимые переходы данными, которые видно целиком и можно проверить перебором, а не логикой, размазанной по вложенным `if`. Такой автомат подходит для соединения, протокольного разбора и жизненного цикла запроса; использование для событийного ввода-вывода показано в [главе про системный уровень](11-system-io.md).

```c
#include <errno.h>
#include <stdio.h>

typedef enum { S_IDLE, S_CONNECTING, S_OPEN, S_CLOSING, S_COUNT } state;
typedef enum { E_CONNECT, E_CONNECTED, E_FAILED, E_CLOSE, E_CLOSED, E_COUNT } event;

constexpr int NO = -1; /* переход запрещён */

static const int next_state[S_COUNT][E_COUNT] = {
    /*                 CONNECT        CONNECTED  FAILED  CLOSE      CLOSED */
    [S_IDLE]       = { S_CONNECTING,  NO,        NO,     NO,        NO     },
    [S_CONNECTING] = { NO,            S_OPEN,    S_IDLE, S_CLOSING, NO     },
    [S_OPEN]       = { NO,            NO,        S_IDLE, S_CLOSING, S_IDLE },
    [S_CLOSING]    = { NO,            NO,        NO,     NO,        S_IDLE },
};

static const char *const state_name[S_COUNT] = {
    [S_IDLE] = "idle", [S_CONNECTING] = "connecting", [S_OPEN] = "open", [S_CLOSING] = "closing",
};

typedef struct { state st; } conn;

/* conn_fire - применяет событие. Возврат: 0 или -EPROTO, состояние при отказе не меняется. */
static int conn_fire(conn *c, event e) {
    int to = next_state[c->st][e];
    if (to == NO) return -EPROTO;
    c->st = (state)to;
    return 0;
}

int main(void) {
    conn c = {S_IDLE};
    event seq[] = {E_CONNECT, E_CONNECT, E_CONNECTED, E_CLOSE, E_CLOSED};
    for (size_t i = 0; i < sizeof seq / sizeof seq[0]; i++) {
        int rc = conn_fire(&c, seq[i]);
        printf("%s%s\n", state_name[c.st], rc ? " (отказ)" : "");
    }
}
```

Вывод: `connecting`, `connecting (отказ)`, `open`, `closing`, `idle`. Каждая клетка таблицы выписана, а запрет стоит явно как `NO`, поэтому ревью видит, что про клетку подумали; повторное `E_CONNECT` отвергнуто и состояние не изменилось. Действия при входе в состояние добавляют вторым `switch` по новому состоянию без `default` либо вторым массивом указателей на функции, индексированным по состоянию.

#### Подводные камни

- Пропущенная клетка в таблице с неполной инициализацией молча равна нулю, то есть переходу в состояние с номером 0. Выписывай все клетки или делай нулевое значение запрещённым.
- Новое состояние или событие в enum должно менять размеры таблицы: бери размер из `S_COUNT` и `E_COUNT` (последний элемент enum), а не из числа в массиве, и добавь `static_assert` для связанных таблиц одного размера.
- Недопустимое событие не игнорируй молча: возвращай ошибку и логируй, иначе рассинхронизация с партнёром всплывёт далеко от причины.
- Состояние, зависящее от данных (счётчик, накопленный префикс), не раздувай в enum: держи данные в структуре рядом с состоянием, а в таблице оставляй только значимые для переходов классы.

## Ссылки и обход

### Выдавай хэндлы вместо указателей на переиспользуемые слоты

Указатель на элемент пула остаётся «валидным» после освобождения слота и указывает на чужой объект, а хэндл из индекса и поколения после освобождения просто перестаёт опознаваться. Это защита от висячих ссылок без счётчиков ссылок: слоты лежат в массиве (в пуле или в блоке арены из [главы про память](03-memory-ownership.md)), индекс даёт доступ за одно обращение, поколение растёт при каждом освобождении слота и старый хэндл отвергается.

```c
#include <stdint.h>
#include <stdio.h>
#define POOL_CAP 4u

typedef struct { uint32_t idx, gen; } handle; /* gen == 0: недействительный */
typedef struct { int hp; uint32_t gen, next; } slot;
typedef struct { slot s[POOL_CAP]; uint32_t free; } pool;

static void pool_init(pool *p) { /* POOL_CAP - конец списка свободных */
    for (uint32_t i = 0; i < POOL_CAP; i++) p->s[i] = (slot){.gen = 1, .next = i + 1};
    p->free = 0;
}

static handle pool_add(pool *p, int hp) {
    if (p->free == POOL_CAP) return (handle){};
    uint32_t i = p->free;
    p->free = p->s[i].next, p->s[i].hp = hp;
    return (handle){i, p->s[i].gen};
}

static slot *pool_get(pool *p, handle h) {
    return h.idx < POOL_CAP && p->s[h.idx].gen == h.gen ? &p->s[h.idx] : nullptr;
}

static bool pool_remove(pool *p, handle h) {
    slot *s = pool_get(p, h);
    if (!s) return false;
    if (++s->gen == 0) s->gen = 1; /* при переполнении 0 пропускаем */
    s->next = p->free, p->free = h.idx;
    return true;
}

int main(void) {
    pool p;
    pool_init(&p);
    handle a = pool_add(&p, 10); pool_remove(&p, a);
    handle c = pool_add(&p, 30); /* займёт слот a с новым поколением */
    printf("%u %u %u\n", a.idx, c.idx, c.gen); /* вывод: 0 0 2 */
    printf("%d %d\n", pool_get(&p, a) == nullptr, pool_get(&p, c)->hp); /* вывод: 1 30 */
}
```

Старый хэндл `a` указывает на тот же индекс, что и `c`, но поколение отличается, поэтому `pool_get` возвращает `nullptr`: вместо чтения чужого объекта вызывающий получает явный отказ. Хэндл передают и хранят по значению, а указатель из `pool_get` не сохраняют дольше одной операции.

#### Подводные камни

- Указатель из `pool_get` инвалидируется любой операцией, которая может переставить слоты (рост пула с `realloc`, перенос); хэндл переживает такой рост, указатель нет. Это и есть главная причина выбирать хэндлы для растущих коллекций.
- Счётчик поколения конечен: после переполнения старый хэндл может совпасть с новым. Выбирай достаточную разрядность (32 бита хватает практически всегда) и пропускай значение 0, зарезервированное для нулевого хэндла.
- Хэндл не знает, из какого пула он: хэндл одного пула, предъявленный другому, пройдёт проверку по индексу и поколению. При нескольких пулях добавь в хэндл номер пула или используй для каждого пула свой тип структуры.
- `{}` для хэндла даёт недействительное значение: ошибку выдачи и неинициализированное поле возвращай так же.

### Обходи коллекцию курсором без выделений

Курсор-структура на стеке хранит позицию, а `next` возвращает `true` и отдаёт очередной элемент через out-параметр, поэтому обход не выделяет память и работает с любой структурой данных одним и тем же циклом `while`. Пример разбивает срез по разделителю: части остаются срезами исходной строки.

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdio.h>
#include <string.h>

typedef struct { const char *ptr; size_t len; } str;
typedef struct { str rest; char sep; bool done; } split_it;

static split_it split_begin(str s, char sep) { return (split_it){s, sep, false}; }

/* split_next - следующая часть; false, когда части кончились. Пустые части отдаются. */
static bool split_next(split_it *it, str *out) {
    if (it->done) return false;
    const char *p = it->rest.len ? memchr(it->rest.ptr, it->sep, it->rest.len) : nullptr;
    size_t n = p ? (size_t)(p - it->rest.ptr) : it->rest.len;
    *out = (str){it->rest.ptr, n};
    if (p) { it->rest.ptr += n + 1; it->rest.len -= n + 1; }
    else it->done = true;
    return true;
}

int main(void) {
    split_it it = split_begin((str){"a,bc,,d", 7}, ',');
    str part;
    while (split_next(&it, &part)) printf("[%.*s]\n", (int)part.len, part.ptr);
}
```

Вывод: `[a]`, `[bc]`, `[]`, `[d]`.

Когда элементы уже лежат в чужом списке, звено встраивают в структуру элемента и возвращаются к элементу сдвигом на `offsetof` (макрос `container_of`, [контейнеры и макросы](09-containers-and-generics.md)); такой список не выделяет память для звеньев. Изменение коллекции во время обхода делает курсор недействительным: для удаления в цикле читай `next` до удаления. Срез-результат живёт, пока жив исходный буфер, а не курсор.

## Расширение

### Загружай плагины через интерфейс с версией ABI

Плагин собирается отдельным загружаемым модулем и не знает хоста на этапе компиляции, поэтому весь обмен идёт через одну экспортируемую структуру с таблицей операций, а первым полем стоит версия интерфейса: хост проверяет её раньше всего остального. Хост ищет через `dlsym` не функцию, а данные: указатель на объект приводится к указателю на структуру без нестандартных преобразований указателей на функции.

```c
/* plugin.h */
#ifndef PLUGIN_H
#define PLUGIN_H
#include <stddef.h>

#define PLUGIN_ABI 1u

/* Первое поле abi обязано оставаться на месте во всех версиях. */
typedef struct plugin_ops {
    unsigned abi;
    int (*run)(const char *in, char *out, size_t cap); /* 0 или -E... */
} plugin_ops;
#endif
```

```c
/* plugin_greet.c */
#include "plugin.h"
#include <errno.h>
#include <stdio.h>

static int greet_run(const char *in, char *out, size_t cap) {
    int n = snprintf(out, cap, "hello, %s", in);
    return n >= 0 && (size_t)n < cap ? 0 : -ENOSPC;
}

[[gnu::visibility("default")]] const plugin_ops plugin_v1 = {PLUGIN_ABI, greet_run};
```

Хост проверяет по порядку: открылся ли модуль, найден ли символ, совпадает ли версия; модуль закрывает на каждом пути выхода и после закрытия ничего из него не вызывает.

```c
/* main.c */
#include "plugin.h"
#include <dlfcn.h>
#include <stdio.h>

int main(int argc, char **argv) {
    if (argc < 2) { puts("usage: host plugin.so"); return 0; }
    void *h = dlopen(argv[1], RTLD_NOW | RTLD_LOCAL);
    if (!h) { fprintf(stderr, "dlopen: %s\n", dlerror()); return 1; }
    const plugin_ops *p = dlsym(h, "plugin_v1");
    char out[64];
    int rc = p && p->abi == PLUGIN_ABI && p->run("world", out, sizeof out) == 0 ? (puts(out), 0) : 1;
    dlclose(h);
    return rc;
}
```

Плагин собирается как разделяемая библиотека (`cc -std=c23 -fPIC -shared -o greet.so plugin_greet.c`), хост линкуется как обычная программа; на glibc 2.34 и новее `-ldl` не нужен.

#### Подводные камни

- Загрузка плагина выполняет его код (в том числе конструкторы) с правами хоста: плагины из недоверенных каталогов не загружай ([безопасность](12-security.md)). Имя без `/` в `dlopen` ищется по путям загрузчика, а не в текущем каталоге.
- После `dlclose` все указатели в модуль (таблица операций, функции) недействительны; плагин не оставляет потоков и обработчиков, исполняющих его код.
- Изменение раскладки `plugin_ops` ломает собранные плагины: добавляй поля только в конец, а несовместимые изменения оформляй новой версией `abi` и новым именем символа (`plugin_v2`). `[[gnu::visibility("default")]]` (расширение GCC/Clang) нужен при `-fvisibility=hidden`; флаг `-Wl,-z,nodlopen` из [главы про сборку](01-toolchain.md) модулю плагина ставить нельзя.

## Коротко

### Делай логирующий макрос с файлом, строкой и уровнем, отсекаемым на этапе компиляции

Уровень ниже порога превращается в ветку с константным условием, которую компилятор выбрасывает целиком, а проверка формата и аргументов при этом сохраняется. `__VA_OPT__` убирает лишнюю запятую, когда после формата нет аргументов; аргументы вычисляются только при включённом уровне, так что побочные эффекты в них не должны быть нужны программе. Номер `__LINE__` приводят к `long`, чтобы не зависеть от её типа при печати через `%ld`.

```c
#include <stdio.h>
#include <stdarg.h>

enum { LOG_DEBUG, LOG_INFO, LOG_WARN, LOG_ERROR };

#ifndef LOG_MIN
#define LOG_MIN LOG_INFO
#endif

#define LOG(lvl, fmt, ...) \
    do { if ((lvl) >= LOG_MIN) log_write(lvl, __FILE__, (long)__LINE__, fmt __VA_OPT__(,) __VA_ARGS__); } while (0)

/* gnu::format: расширение GCC/Clang, проверка формата как у printf */
[[gnu::format(printf, 4, 5)]]
static void log_write(int lvl, const char *file, long line, const char *fmt, ...) {
    static const char *const names[] = {"DEBUG", "INFO", "WARN", "ERROR"};
    va_list ap;
    va_start(ap, fmt);
    printf("%s %s:%ld: ", names[lvl], file, line);
    vprintf(fmt, ap);
    putchar('\n');
    va_end(ap);
}

int main(void) {
    LOG(LOG_DEBUG, "скрыто"); /* отсечено при LOG_MIN == LOG_INFO */
    LOG(LOG_INFO, "готово");
    LOG(LOG_WARN, "код %d", 7);
}
```

Строка лога выглядит как `INFO main.c:НОМЕР: готово`: файл и строка берутся из места вызова.

#### Подводные камни

- Чтобы превратить число в строку (`__FILE__ ":" STR(__LINE__)`), нужна двухуровневая обёртка: `#define STR_(x) #x` и `#define STR(x) STR_(x)`. Прямое `#__LINE__` даст текст `"__LINE__"`, потому что оператор `#` не раскрывает макрос в аргументе; то же верно для склейки `##`.
- Аргумент макроса подставляется в тело столько раз, сколько он там встречается: `LOG(lvl, ...)` вычисляет `lvl` дважды, поэтому передавай константу, а не выражение с побочным эффектом.
- Порог `LOG_MIN` задаётся флагом сборки (`-DLOG_MIN=2`); энумератор нельзя использовать в `#if`, поэтому условие пишется как обычный `if`.

### Диспетчеризуй по enum через таблицу указателей на функции

Таблица, индексируемая значением enum, заменяет длинную цепочку `if`/`switch` и позволяет проверить покрытие в одном месте. Индекс из внешнего источника проверяют до обращения, а отсутствующая запись (`nullptr` при designated-инициализации) трактуется как неподдерживаемая команда.

```c
#include <errno.h>

typedef enum { CMD_PING, CMD_ECHO, CMD_COUNT } cmd;

static int do_ping(int arg) { (void)arg; return 0; }
static int do_echo(int arg) { return arg; }

static int (*const handlers[CMD_COUNT])(int) = {[CMD_PING] = do_ping, [CMD_ECHO] = do_echo};

int dispatch(unsigned c, int arg) {
    return c < CMD_COUNT && handlers[c] ? handlers[c](arg) : -ENOSYS;
}
```

## Что ещё стоит знать

Объектные системы с наследованием и динамической типизацией (GObject и подобные) строятся из тех же приёмов: таблицы операций с указателем на родительскую таблицу, теги типов и счётчики ссылок. Использовать их стоит, когда нужна интроспекция и привязки к другим средам; для обычного кода хватает таблицы операций из этой главы. «Исключения» через `setjmp`/`longjmp` описаны в [главе про ошибки](04-errors-and-cleanup.md) и в паттернах не нужны.
