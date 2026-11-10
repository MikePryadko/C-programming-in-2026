# Паттерны

## Зависимости

### Передавай внешний мир в логику интерфейсом, а не вызывай его напрямую

Функция, которая сама читает часы, сокет или диск, проверяется только на настоящих часах, сокете и диске, поэтому всё, что делает логику недетерминированной, передают ей через таблицу операций из [главы про API](07-api-and-modules.md). Боевая реализация подставляется в `main`, тестовая в тесте; фреймворк для этого не нужен. Ниже ограничитель частоты: логика зависит только от интерфейса часов `clk`, а не от `clock_gettime`.

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

Логика получает зависимость при создании и хранит в своей структуре (контекст); функции без состояния интерфейс передают параметром.

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

Как собирать такие тесты с моками и проверками, показано в [главе про тестирование](09-testing.md); приём тот же: тест подставляет свою таблицу операций.

#### Подводные камни

- Один большой интерфейс на всё (часы, диск, сеть, лог в одной структуре `env`) превращает внедрение в объект бога: любая функция зависит от всего и тест обязан подделать всё. Передавай каждой функции только то, что она использует, и дели интерфейсы по назначению.
- Тестовая реализация должна подчиняться тому же контракту, что боевая (коды ошибок, смысл `0` из `read`); иначе тест проходит на заглушке, которая ведёт себя не так, как настоящая зависимость.
- Подмена через `#ifdef TEST` внутри логики не заменяет внедрение: в тесте проверяется другой код, чем в продукте. Подмену на этапе линковки (другой `.c` в тестовой сборке) допускай для зависимостей с одной реализацией на сборку.
- Внедрение не нужно для чистых функций, для стандартных `memcpy` и `strlen` и для зависимостей, у которых нет ни второй реализации, ни побочных эффектов, мешающих тесту. Интерфейс на каждый вызов запутывает код и убирает инлайнинг.
- Владение контекстом остаётся за создателем: интерфейс не освобождает `ctx`, если иное не оговорено в контракте ([владение](04-memory-ownership.md)).

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

Конструкторы вариантов (`msg_ping`, `msg_data`) единственное место, где пишут одновременно тег и поле, а доступ к варианту идёт только внутри `case` своего тега или после проверки тега. Код после `switch` нужен, потому что тег может содержать мусор; `abort()` честно это выражает; `unreachable()` здесь не годится: при мусорном теге он даёт неопределённое поведение, а годится он только там, где недостижимость доказана построением ([типы и UB](03-types-and-ub.md)).

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

Таблица `состояние x событие -> следующее состояние` делает допустимые переходы данными, которые видно целиком и можно проверить перебором, а не логикой, размазанной по вложенным `if`. Такой автомат подходит для соединения, протокольного разбора и жизненного цикла запроса; использование для событийного ввода-вывода показано в [главе про системный уровень](13-system-io.md).

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

Указатель на элемент пула остаётся «валидным» после освобождения слота и указывает на чужой объект, а хэндл из индекса и поколения после освобождения просто перестаёт опознаваться. Это защита от висячих ссылок без счётчиков ссылок: слоты лежат в массиве (в пуле или в блоке арены из [главы про память](04-memory-ownership.md)), индекс даёт доступ за одно обращение, поколение растёт при каждом освобождении слота и старый хэндл отвергается.

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

Курсор-структура на стеке хранит позицию, а `next` возвращает `true` и отдаёт очередной элемент через out-параметр, поэтому обход не выделяет память и работает с любой структурой данных одним и тем же циклом `while`. Пример разбивает срез по разделителю: части остаются срезами исходной строки. Готовый вариант для срезов, `str_token`, разобран в [строках и буферах](06-strings-and-buffers.md); здесь та же идея показана как общий приём.

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

Когда элементы уже лежат в чужом списке, звено встраивают в структуру элемента и возвращаются к элементу сдвигом на `offsetof` (макрос `container_of`, [контейнеры и макросы](11-containers-and-generics.md)); такой список не выделяет память для звеньев. Изменение коллекции во время обхода делает курсор недействительным: для удаления в цикле читай `next` до удаления. Срез-результат живёт, пока жив исходный буфер, а не курсор.

## Расширение

Механизм расширения выбирают по тому, насколько доверяют коду расширения и сколько готовы платить за изоляцию.

| Механизм | Изоляция | Цена вызова | Язык расширения | ABI | Горячая перезагрузка | Безопасность |
| --- | --- | --- | --- | --- | --- | --- |
| `dlopen` в процессе | нет, общее адресное пространство | самая низкая: вызов по указателю | C и всё, что отдаёт C ABI | бинарный: таблицы с версией и размером | ограниченная: новый файл через `rename`, затем новый `dlopen` | код с правами хоста: только доверенные плагины |
| Подпроцесс и IPC | процесс: падение и зависание плагина хост переживает | высокая: копирование данных и переключения контекста | любой | протокол сообщений | перезапуск процесса | права процесса ограничиваются отдельно |
| Встроенный Lua | виртуальная машина; лимиты времени и памяти ставит хост | средняя: значения идут через стек интерпретатора | Lua | C API библиотеки | загрузка нового сценария без перезапуска | сценарию доступно только то, что открыл хост |
| WebAssembly | линейная память модуля | средняя | любой язык, собираемый в wasm | импорты и экспорты модуля | замена модуля | наружу только импортированные функции |
| Статическая регистрация | нет | прямой вызов | C | нет, одна сборка | нет, нужна пересборка | код своего репозитория |

Выбор по умолчанию такой. Расширение C-приложения без пересборки: `dlopen` с таблицей `host_api`. Плагины недоверенные или могут падать: подпроцесс. Пользовательские сценарии и настройка логики: встроенный Lua. Модули одного репозитория, которым не нужна загрузка во время работы: статическая регистрация. WebAssembly остаётся перспективным способом изоляции, но готовых сред исполнения в пакетах Ubuntu 24.04 нет, поэтому рекомендации ему пока нет (см. «Коротко»).

### Загружай плагины через таблицу host_api и версию ABI

Плагин собирается отдельным модулем и не знает хоста на этапе компиляции, поэтому хост передаёт ему всё нужное таблицей функций `host_api` при инициализации, а не заставляет искать свои символы в процессе (для этого хост пришлось бы линковать с `-rdynamic`, а плагин привязался бы к именам внутри конкретной сборки хоста). Плагин в ответ отдаёт одну экспортируемую структуру `plugin_ops` с версией ABI первым полем. Размер `host_api` лежит в первом поле `size`: приём расширения структуры тот же, что в [главе про API](07-api-and-modules.md), плагин проверяет, что нужные ему поля хост заполнил.

```c
/* hostapi.h */
#ifndef HOSTAPI_H
#define HOSTAPI_H
#include <stddef.h>

#define PLUGIN_ABI 1u

/* Таблица хоста: size - sizeof(host_api) из заголовка хоста; новые поля только в конец. */
typedef struct host_api { size_t size; void (*log)(int level, const char *msg); void *(*alloc)(size_t n); void (*release)(void *p); } host_api;

/* Первое поле abi обязано оставаться на месте во всех версиях. Функции возвращают 0 или -E...,
 * ошибку не сообщают через exit или longjmp. */
typedef struct plugin_ops { unsigned abi; int (*init)(const host_api *host); int (*run)(const char *in, char *out, size_t cap); void (*shutdown)(void); } plugin_ops;
#endif
```

Плагин получает таблицу в `init` и пользуется только ею; память, которой обмениваются стороны, он берёт и возвращает через `host->alloc` и `host->release`.

```c
/* plug_greet.c */
#include "hostapi.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>

static const host_api *host;

static int greet_init(const host_api *h) {
    if (h->size < offsetof(host_api, release) + sizeof h->release) return -EPROTO;
    host = h;
    host->log(1, "greet: init");
    return 0;
}

static int greet_run(const char *in, char *out, size_t cap) {
    char *tmp = host->alloc(cap);
    if (!tmp) return -ENOMEM;
    int n = snprintf(tmp, cap, "hello, %s", in);
    int rc = n >= 0 && (size_t)n < cap ? 0 : -ENOSPC;
    if (rc == 0) memcpy(out, tmp, (size_t)n + 1);
    host->release(tmp);
    return rc;
}

static void greet_shutdown(void) { host->log(1, "greet: shutdown"); host = nullptr; }

[[gnu::visibility("default")]] const plugin_ops plugin_v1 = {PLUGIN_ABI, greet_init, greet_run, greet_shutdown};
```

Хост проверяет по порядку: открылся ли модуль, найден ли символ, совпадает ли версия, успешен ли `init`; `shutdown` зовёт только после успешного `init`, модуль закрывает на каждом пути выхода и после закрытия ничего из него не вызывает. Через `dlsym` хост ищет не функцию, а объект `plugin_v1`: `void *` в указатель на структуру приводится без оговорок, а в указатель на функцию GCC под `-Wpedantic` отвергает (Clang молчит).

```c
/* main.c */
#include "hostapi.h"
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>

static void host_log(int level, const char *msg) { printf("[%d] %s\n", level, msg); }
static void *host_alloc(size_t n) { return malloc(n); }
static void host_release(void *p) { free(p); }

int main(int argc, char **argv) {
    if (argc < 2) { puts("usage: host ./plugin.so"); return 0; }
    static const host_api api = {sizeof(host_api), host_log, host_alloc, host_release};
    void *h = dlopen(argv[1], RTLD_NOW | RTLD_LOCAL);
    if (!h) { fprintf(stderr, "dlopen: %s\n", dlerror()); return 1; }
    int rc = 1;
    const plugin_ops *p = dlsym(h, "plugin_v1");
    if (p && p->abi == PLUGIN_ABI && p->init(&api) == 0) { /* вывод: [1] greet: init */
        char out[64];
        if (p->run("world", out, sizeof out) == 0) { puts(out); rc = 0; } /* вывод: hello, world */
        p->shutdown(); /* вывод: [1] greet: shutdown */
    }
    dlclose(h);
    return rc;
}
```

Плагин собирается как разделяемая библиотека, хост линкуется как обычная программа; на glibc 2.34 и новее `-ldl` не нужен. Флаг `-Wl,-z,defs` на плагине превращает любую ссылку на символ хоста (прямой вызов вместо `host->log`) в ошибку линковки, а не в отказ `dlopen` у пользователя.

```sh
cc -std=c23 -fPIC -shared -Wl,-z,defs -o greet.so plug_greet.c
cc -std=c23 -o host main.c
./host ./greet.so
```

```text
[1] greet: init
hello, world
[1] greet: shutdown
```

Плагины находят в каталоге: хост читает его через `scandir` с `alphasort` (порядок `readdir` не определён, а порядок загрузки должен быть воспроизводимым) и открывает по пути с `/`. Рядом с каждым модулем кладут небольшой текстовый манифест (имя, версия ABI, зависимости): по нему плагин отвергают до `dlopen`, ведь сама загрузка уже исполняет код модуля. Манифест только предфильтр, истиной остаётся `abi` из `plugin_ops`.

#### Подводные камни

- Загрузка плагина выполняет его код (в том числе конструкторы) с правами хоста: плагины из недоверенных каталогов не загружай. Имя без `/` в `dlopen` ищется по путям загрузчика, а не в текущем каталоге.
- Выделять и освобождать общую память нужно в одном модуле: на Linux с общей libc `free` хоста освободит память плагина, но модули с разными копиями библиотеки C (типично для Windows) или хост со своим аллокатором (арена, mimalloc) этого не переживут. Поэтому обмен идёт парой `host->alloc` и `host->release`, а не прямыми `malloc` и `free`.
- Конструктор в модуле выполняется до `init`, когда таблицы хоста у плагина ещё нет: вызов функции хоста из него без `-Wl,-z,defs` собирается, а `dlopen` с `RTLD_NOW` падает с `undefined symbol` (хост без `-rdynamic` их не экспортирует). Всю работу с хостом делай в `init`.
- После `dlclose` все указатели в модуль (таблица операций, функции) недействительны; плагин не оставляет потоков и обработчиков, исполняющих его код. Повторный `dlopen` того же пути возвращает тот же дескриптор со счётчиком: конструктор выполняется один раз, деструктор после последнего `dlclose`.
- Загруженный файл нельзя перезаписывать на месте (`cp`, `>`): отображённые страницы меняются под работающим хостом, и он падает по `SIGSEGV`. Новую версию кладут рядом и подменяют через `rename`: загруженный экземпляр продолжает работать со старым содержимым.
- Утечка из плагина после `dlclose` попадает в отчёт LSan с кадром `<unknown module>` вместо имени функции. В тестовой сборке собирай плагин с `-Wl,-z,nodelete` (модуль остаётся в памяти до выхода) или не закрывай модули.
- Изменение раскладки `plugin_ops` ломает собранные плагины: добавляй поля только в конец, а несовместимые изменения оформляй новой версией `abi` и новым именем символа (`plugin_v2`); `host_api` растёт так же, а плагин сверяет `size`. `[[gnu::visibility("default")]]` (расширение GCC/Clang) нужен при `-fvisibility=hidden`; флаг `-Wl,-z,nodlopen` из [главы про флаги и санитайзеры](01-toolchain.md) модулю плагина ставить нельзя.
- Падение, `abort` или зависание плагина останавливает весь хост: границы процесса нет. Для недоверенного или нестабильного кода переходи к подпроцессу (следующая практика).

## Плагин в подпроцессе

### Запускай ненадёжный плагин отдельным процессом с таймаутом ответа

Плагин в подпроцессе может упасть, зависнуть или утечь, а хост остаётся целым: он видит закрытое соединение или истёкший таймаут и запускает плагин заново. Хост и плагин обмениваются сообщениями по `socketpair` типа `SOCK_SEQPACKET`: он сохраняет границы сообщений, поэтому кадры с длиной не нужны (для потока `SOCK_STREAM` или канала их строят, как описано в [главе про сериализацию](14-serialization.md)). Пример собирает хост из трёх файлов; в настоящем плагине `body` заменяет запуск отдельной программы через `posix_spawn` ([системный уровень](13-system-io.md)).

```c
/* plug.h */
#ifndef PLUG_H
#define PLUG_H
#include <stddef.h>
#include <sys/types.h>

typedef struct plug { pid_t pid; int fd; } plug;

/* plug_start - запускает body(fd) в подпроцессе. Возврат: 0 или -errno. */
int plug_start(plug *p, void (*body)(int fd));

/* plug_call - один запрос, один ответ. Возврат: 0, -ETIMEDOUT (плагин убит)
 * или -EPIPE (плагин умер); после ошибки плагин завершён и собран, p закрыт. */
int plug_call(plug *p, const char *in, char *out, size_t cap, int ms);
#endif
```

```c
/* plug.c */
#define _GNU_SOURCE
#include "plug.h"
#include <errno.h>
#include <poll.h>
#include <signal.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>

int plug_start(plug *p, void (*body)(int fd)) {
    int sv[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0, sv)) return -errno;
    pid_t pid = fork();
    if (pid < 0) { close(sv[0]); close(sv[1]); return -errno; }
    if (pid == 0) { close(sv[0]); body(sv[1]); _exit(0); }
    close(sv[1]);
    *p = (plug){pid, sv[0]};
    return 0;
}

int plug_call(plug *p, const char *in, char *out, size_t cap, int ms) {
    int rc = 0, st;
    if (send(p->fd, in, strlen(in) + 1, MSG_NOSIGNAL) < 0) rc = -EPIPE;
    else if (poll(&(struct pollfd){p->fd, POLLIN, 0}, 1, ms) == 0) rc = -ETIMEDOUT;
    else if (recv(p->fd, out, cap, 0) <= 0) rc = -EPIPE;
    if (rc == -ETIMEDOUT) kill(p->pid, SIGKILL);
    if (rc) { close(p->fd); waitpid(p->pid, &st, 0); }
    return rc;
}
```

Демонстрация подставляет тело плагина прямо в `main.c`: запрос `crash` роняет плагин через `abort`, запрос `hang` вешает его; хост в обоих случаях получает код ошибки и продолжает работу.

```c
/* main.c */
#define _GNU_SOURCE
#include "plug.h"
#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

static void upper_body(int fd) { /* код плагина: прописные буквы, "crash" роняет, "hang" висит */
    char buf[64];
    ssize_t n;
    while ((n = recv(fd, buf, sizeof buf, 0)) > 0) {
        if (strcmp(buf, "crash") == 0) abort();
        if (strcmp(buf, "hang") == 0) pause();
        for (ssize_t i = 0; i < n; i++) buf[i] = (char)toupper((unsigned char)buf[i]);
        send(fd, buf, (size_t)n, MSG_NOSIGNAL);
    }
}

int main(void) {
    char out[64];
    plug p;
    if (plug_start(&p, upper_body)) return 1;
    int rc = plug_call(&p, "hello", out, sizeof out, 1000);
    printf("%d %s\n", rc, out); /* вывод: 0 HELLO */
    printf("%d\n", plug_call(&p, "crash", out, sizeof out, 1000) == -EPIPE); /* вывод: 1 */
    if (plug_start(&p, upper_body)) return 1;
    printf("%d\n", plug_call(&p, "hang", out, sizeof out, 100) == -ETIMEDOUT); /* вывод: 1 */
}
```

Цена изоляции: каждый запрос копирует данные через системный вызов в обе стороны и переключает контекст, поэтому подпроцесс годится для крупных операций (преобразование файла, проверка документа), а не для вызова в горячем цикле. Ответ плагина остаётся недоверенными данными: длину и формат проверяй так же, как у внешнего ввода ([безопасность](19-security.md)).

#### Подводные камни

- Запись в сокет мёртвого плагина без `MSG_NOSIGNAL` присылает хосту `SIGPIPE` и убивает его; флаг превращает это в ошибку `EPIPE`.
- Убитый по таймауту плагин остаётся зомби, пока хост не вызвал `waitpid`; `plug_call` собирает его сам, а повторный вызов на закрытом `plug` запрещён: после ошибки запускай плагин заново.
- Потомок `fork` наследует все открытые дескрипторы хоста, в том числе сокеты других плагинов: пока хоть один экземпляр держит конец сокета, `recv` не получит конец потока. Запускай плагин через `exec` с `SOCK_CLOEXEC`; демонстрационный `fork` без `exec` годится только для однопоточного хоста (между `fork` и `exec` в многопоточной программе допустимы лишь функции, безопасные в обработчике сигнала).
- Запрос больше буфера плагина или ответ больше `cap` обрезается границей сообщения `SOCK_SEQPACKET`: размер проверяй на обеих сторонах.

## Статическая регистрация

### Регистрируй модули через секцию, а не центральный список

Новый модуль добавляется одним файлом без правки общей таблицы, поэтому забытая запись в списке невозможна: каждый модуль кладёт свою запись в именованную секцию, а компоновщик собирает секцию в массив и даёт его границы. Механизм секций разобран в [главе про возможности компилятора](08-compiler-features.md); это единственный пример границ `__start_` и `__stop_` в книге. Они работают для ELF с GNU ld и lld, а имя секции пишется без точки (иначе символов границ нет).

```c
/* reg.h */
#ifndef REG_H
#define REG_H
#include <stddef.h>
#include <stdint.h>
#include <string.h>

typedef struct reg_entry { const char *name; int (*run)(int); } reg_entry;

#if __has_attribute(retain)
#define REG_ATTR [[gnu::used, gnu::retain, gnu::section("reg_entries")]]
#else
#define REG_ATTR [[gnu::used, gnu::section("reg_entries")]]
#endif
/* REG_MODULE - кладёт запись в секцию reg_entries; все записи одного типа и const. */
#define REG_MODULE(id, fn) REG_ATTR static const reg_entry reg_##id = {#id, fn}

extern const reg_entry __start_reg_entries[], __stop_reg_entries[];

/* reg_count - число записей; разность берётся в байтах (не вычитай указатели на разные объекты). */
static inline size_t reg_count(void) {
    return ((uintptr_t)__stop_reg_entries - (uintptr_t)__start_reg_entries) / sizeof(reg_entry);
}
#endif
```

```c
/* mod_twice.c */
#include "reg.h"

static int twice(int x) { return 2 * x; }
REG_MODULE(twice, twice);
```

```c
/* mod_inc.c */
#include "reg.h"

static int inc(int x) { return x + 1; }
REG_MODULE(inc, inc);
```

Главный файл ничего не знает о модулях, кроме границ секции; порядок записей не определён, поэтому он ищет по имени, а не по номеру.

```c
/* main.c */
#include "reg.h"
#include <stdio.h>

int main(void) {
    printf("%zu\n", reg_count()); /* вывод: 2 */
    for (size_t i = 0; i < reg_count(); i++)
        if (strcmp(__start_reg_entries[i].name, "inc") == 0)
            printf("%d\n", __start_reg_entries[i].run(41)); /* вывод: 42 */
}
```

Ловушку архива показывает такая сборка: `mod_twice.c` и `mod_inc.c` скомпилированы в объекты и собраны в `libmods.a` командой `ar rcs libmods.a mod_twice.o mod_inc.o`, а `main.c` скомпилирован в `main.o`.

```sh
cc main.o libmods.a -o app
cc main.o -Wl,--whole-archive libmods.a -Wl,--no-whole-archive -o app
```

Первая команда не находит `__start_reg_entries`, вторая подключает все объекты архива.

#### Подводные камни

- Объект из статической библиотеки `.a` линкер берёт, только если на него есть ссылка. Модули, на которые ссылается лишь секция, из архива не попадают: без других ссылок границ секции вообще нет и компоновка падает с `undefined reference to __start_reg_entries`, а если один модуль архива нужен по другой причине, регистрация тихо получает только его. Подключай архив целиком или собирай модули как объектную библиотеку ([сборка](02-build.md)).
- Сборка с `--gc-sections` удаляет записи, на которые нет ссылок. GNU ld сохраняет секцию по `used`, а lld без `retain` останавливает компоновку с подсказкой про `-z nostart-stop-gc`; поэтому макрос ставит `retain` по `__has_attribute` и одинаково у всех записей ([возможности компилятора](08-compiler-features.md)).
- Разность `__stop_reg_entries - __start_reg_entries` вычитает указатели на разные объекты, а это неопределённое поведение: считай разность через `uintptr_t`, как в `reg_count`.
- Записи разных типов, размеров или выравниваний в одной секции делают обход массива неверным: все объекты секции одного типа.
- Регистрация живёт в одной сборке: добавить модуль без пересборки нельзя, а для форматов исполняемых файлов без `__start_` и `__stop_` нужен другой механизм (регистрация из `constructor`, см. «Коротко»).

## Коротко

### Встраивай Lua для пользовательских сценариев

Сценарии пользователя не должны быть собранным кодом: встроенный интерпретатор Lua (пакет `liblua5.4-dev`, флаги из `pkg-config lua5.4`) выполняет их без пересборки, а доступ к системе хост выдаёт сам: функции регистрирует он, стандартные библиотеки не открывает, поэтому `os` и `io` сценарию недоступны.

```c
/* lua_host.c */
#include <lauxlib.h>
#include <lua.h>
#include <stdio.h>

static int host_add(lua_State *L) { /* функция хоста для сценария */
    lua_pushinteger(L, luaL_checkinteger(L, 1) + luaL_checkinteger(L, 2));
    return 1;
}

int main(void) {
    lua_State *L = luaL_newstate();
    if (!L) return 1;
    lua_register(L, "host_add", host_add); /* стандартные библиотеки не открыты */
    if (luaL_dostring(L, "return host_add(40, 2)") == LUA_OK)
        printf("%lld\n", (long long)lua_tointeger(L, -1)); /* вывод: 42 */
    if (luaL_dostring(L, "return os.exit()") != LUA_OK)
        printf("%s\n", lua_tostring(L, -1)); /* вывод: [string "return os.exit()"]:1: attempt to index a nil value (global 'os') */
    lua_close(L);
}
```

Ошибка сценария возвращается кодом, а сообщение лежит на вершине стека. Функции хоста с `luaL_check*` при неверном аргументе не возвращаются (интерпретатор прерывает их долгим переходом), поэтому в них не держат ресурсы, требующие освобождения. Лимиты времени и памяти хост ставит сам: бесконечный цикл сценария без этого блокирует вызывающий поток.

### Регистрируй через constructor, когда секция недоступна

Функция с `[[gnu::constructor]]` выполняется до `main` и может добавить модуль в список регистрации; порядок между единицами трансляции без приоритетов не определён, а правила приоритетов описаны в [главе про возможности компилятора](08-compiler-features.md). Секция лучше: список строится компоновщиком, а не кодом, выполняемым до `main`.

### WebAssembly как изоляция пока без рекомендации

Модуль wasm изолирован линейной памятью, а наружу ему доступно только импортированное, но готовой среды исполнения (wasm3, WAMR, wasmtime) в пакетах Ubuntu 24.04 нет: её собирают из исходников, а это отдельная зависимость. Сам модуль собирается обычным Clang с lld без libc: проверена сборка и сигнатура `\0asm`, запуск не проверялся.

```sh
clang-19 --target=wasm32 -nostdlib -fuse-ld=lld -Wl,--no-entry -Wl,--export=add -O2 add.c -o add.wasm
```

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

Объектные системы с наследованием и динамической типизацией (GObject и подобные) строятся из тех же приёмов: таблицы операций с указателем на родительскую таблицу, теги типов и счётчики ссылок. Использовать их стоит, когда нужна интроспекция и привязки к другим средам; для обычного кода хватает таблицы операций из этой главы. «Исключения» через `setjmp`/`longjmp` описаны в [главе про ошибки](05-errors-and-cleanup.md) и в паттернах не нужны. Журнал событий, макрос с файлом и строкой и приёмники лога разобраны в [главе про логирование](16-logging.md).
