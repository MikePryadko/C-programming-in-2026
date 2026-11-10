# Многопоточность

## Синхронизация

### Защищай общие данные мьютексом

Мьютекс защищает не переменную, а инвариант: связь между полями, которую другие потоки обязаны видеть целой, поэтому все поля инварианта меняют и читают под одним мьютексом, а лежат они вместе с ним в одной структуре. Одновременное обращение двух потоков к одной памяти, если хотя бы один пишет, называется гонкой данных и даёт неопределённое поведение ([о неопределённом поведении](03-types-and-ub.md)): компилятор вправе держать значение в регистре, а запись может оказаться неатомарной. Читать без блокировки нельзя так же, как писать.

```c
/* Ошибка: пишут под мьютексом, а читают без него (гонка данных) */
#include <pthread.h>
#include <stdio.h>

typedef struct { pthread_mutex_t mu; long count, total; } stats;
static stats g = {PTHREAD_MUTEX_INITIALIZER, 0, 0};

static void stats_add(stats *s, long v) {
    pthread_mutex_lock(&s->mu);
    s->count++;
    s->total += v;
    pthread_mutex_unlock(&s->mu);
}

static long stats_mean(stats *s) { return s->count ? s->total / s->count : 0; }

static void *worker(void *arg) {
    (void)arg;
    for (int i = 0; i < 1000; i++) stats_add(&g, 10);
    return nullptr;
}

int main(void) {
    pthread_t t;
    pthread_create(&t, nullptr, worker, nullptr);
    long m = stats_mean(&g);
    pthread_join(t, nullptr);
    printf("%ld\n", m);
}
```

Эту ошибку молча пропускают и компилятор, и ASan; находит её TSan (сборка и запуск описаны в [главе про флаги и санитайзеры](01-toolchain.md)). Отчёт называет обе стороны гонки, и видно, что писатель держал мьютекс, а читатель нет.

```text
WARNING: ThreadSanitizer: data race (pid=1)
  Write of size 8 at 0x5555569e8740 by thread T1 (mutexes: write M0):
    #0 stats_add
    #1 worker

  Previous read of size 8 at 0x5555569e8740 by main thread:
    #0 stats_mean
    #1 main
SUMMARY: ThreadSanitizer: data race in stats_add
```

Исправление: читатель берёт тот же мьютекс и читает оба поля одной секцией, иначе пара `count` и `total` будет несогласованной даже без формальной гонки. Структура следует канону `_init`/`_clear` ([пара создания и уничтожения](04-memory-ownership.md)).

```c
#include <pthread.h>
#include <stdio.h>

typedef struct { pthread_mutex_t mu; long count, total; } stats;

static void stats_init(stats *s) {
    pthread_mutex_init(&s->mu, nullptr);
    s->count = s->total = 0;
}

static void stats_clear(stats *s) { pthread_mutex_destroy(&s->mu); }

static void stats_add(stats *s, long v) {
    pthread_mutex_lock(&s->mu);
    s->count++;
    s->total += v;
    pthread_mutex_unlock(&s->mu);
}

static long stats_mean(stats *s) {
    pthread_mutex_lock(&s->mu);
    long m = s->count ? s->total / s->count : 0;
    pthread_mutex_unlock(&s->mu);
    return m;
}

static void *worker(void *arg) {
    for (int i = 0; i < 1000; i++) stats_add(arg, 10);
    return nullptr;
}

int main(void) {
    stats s;
    pthread_t t[4];
    stats_init(&s);
    for (int i = 0; i < 4; i++) pthread_create(&t[i], nullptr, worker, &s);
    for (int i = 0; i < 4; i++) pthread_join(t[i], nullptr);
    printf("%ld %ld\n", stats_mean(&s), s.count); /* вывод: 10 4000 */
    stats_clear(&s);
}
```

После `pthread_join` потоки завершены, и читать `s.count` напрямую можно: `join` сам синхронизирует память. Результат `lock` и `unlock` исправного мьютекса в примерах не проверяют: они не отказывают, а ошибки использования (повторный `lock`, `unlock` чужого мьютекса) не диагностируются и ловятся тестами под TSan.

#### Подводные камни

- Вызов чужого кода под блокировкой (callback, функция, берущая другой мьютекс). Он может блокироваться или взять ту же блокировку и повиснуть; собери нужные данные под замком, отпусти его, затем вызывай.
- Ввод-вывод и сон под блокировкой. Все потоки, которым нужен мьютекс, ждут медленную операцию, и зависший сокет останавливает всю программу, а не один поток; под замком делай только работу с памятью.
- Критическая секция с `return` или `goto` посередине. Мьютекс остаётся захваченным; пиши один выход с `unlock` ([единая точка очистки](05-errors-and-cleanup.md)).
- Повторный `lock` обычного мьютекса тем же потоком. Результат не определён, на практике зависание; рекурсивный мьютекс этого не лечит, а прячет плохое разделение кода. Лучше выделить функцию с суффиксом `_locked`, которая замок не берёт, а требует взятым, и написать это в контракте.
- Соседние данные. Мьютекс защищает только то, что ты им защитил; поле, добавленное рядом и забытое в списке инварианта, остаётся открытым.
- Соседние битовые поля. Битовые поля одного слова составляют одну ячейку памяти: запись в одно поле читает и перезаписывает всё слово, поэтому два потока, пишущие в соседние поля под разными мьютексами (или без них), портят друг друга, и TSan сообщает гонку. Поля, которые защищают разные мьютексы, не делай битовыми.
- Одна блокировка на всю структуру почти всегда лучше нескольких мелких: меньше порядков захвата и проще рассуждать.

### Захватывай несколько мьютексов в одном порядке

Если один поток берёт мьютексы A и B, а другой B и A, они рано или поздно оказываются каждый с одним замком в руках и ждут друг друга вечно, и программа зависает без единого сообщения. Лечится единым порядком захвата, общим для всего кода: по уровню (сначала замок реестра, потом замок элемента) или, когда объекты равноправны, по адресу.

```c
/* Ошибка: порядок захвата зависит от аргументов (взаимоблокировка) */
#include <pthread.h>

typedef struct { pthread_mutex_t mu; long balance; } account;

void transfer(account *from, account *to, long v) {
    pthread_mutex_lock(&from->mu);
    pthread_mutex_lock(&to->mu);
    from->balance -= v;
    to->balance += v;
    pthread_mutex_unlock(&to->mu);
    pthread_mutex_unlock(&from->mu);
}
```

Вызов `transfer(a, b)` в одном потоке и `transfer(b, a)` в другом зависает. TSan замечает это без самого зависания, достаточно того, что оба порядка когда-либо встретились:

```text
WARNING: ThreadSanitizer: lock-order-inversion (potential deadlock) (pid=1)
  Cycle in lock order graph: M0 (0x...) => M1 (0x...) => M0
```

Порядок по адресу: сравнивать указатели на разные объекты операторами `<` нельзя (результат не определён), поэтому приводи к `uintptr_t`.

```c
#include <pthread.h>
#include <stdint.h>
#include <stdio.h>

typedef struct { pthread_mutex_t mu; long balance; } account;
typedef struct { account *from, *to; } leg;

static void transfer(account *from, account *to, long v) {
    if (from == to) return;
    account *first = (uintptr_t)from < (uintptr_t)to ? from : to;
    account *second = first == from ? to : from;
    pthread_mutex_lock(&first->mu);
    pthread_mutex_lock(&second->mu);
    from->balance -= v;
    to->balance += v;
    pthread_mutex_unlock(&second->mu);
    pthread_mutex_unlock(&first->mu);
}

static void *shuffle(void *arg) {
    leg *l = arg;
    for (int i = 0; i < 500; i++) transfer(l->from, l->to, 1);
    return nullptr;
}

int main(void) {
    account a = {PTHREAD_MUTEX_INITIALIZER, 1000}, b = {PTHREAD_MUTEX_INITIALIZER, 1000};
    leg ab = {&a, &b}, ba = {&b, &a};
    pthread_t t1, t2;
    pthread_create(&t1, nullptr, shuffle, &ab);
    pthread_create(&t2, nullptr, shuffle, &ba);
    pthread_join(t1, nullptr);
    pthread_join(t2, nullptr);
    printf("%ld %ld\n", a.balance, b.balance); /* вывод: 1000 1000 */
}
```

#### Подводные камни

- Порядок знают не все. Правило записывают комментарием у типа («сначала реестр, потом элемент») и проверяют ревью; один нарушитель возвращает взаимоблокировку.
- Скрытый второй замок. Функция, вызванная под вашим мьютексом, может взять свой; поэтому чужой код под замком не вызывают, а в контракте функции указывают, какие замки она берёт.
- Откат через `pthread_mutex_trylock`: взять первый замок, попробовать второй, при неудаче отпустить оба и повторить. Работает без общего порядка, но требует паузы между попытками, иначе потоки вечно уступают друг другу.
- Для набора из N замков сортируй их по адресу и бери по возрастанию.

### Ждать условие нужно в цикле по предикату

Условная переменная не хранит состояние: она только будит ждущих, а то, чего ждут, лежит в обычном поле под тем же мьютексом (предикат). Ожидающий может проснуться без сигнала (ложное пробуждение), сигнал может прийти раньше, чем поток начал ждать, а к моменту, когда поток снова получит мьютекс, предикат может уже измениться другим потоком, поэтому условие перепроверяют в цикле `while`, а не в `if`.

```c
/* Ошибка: ожидание под if, ложное пробуждение пропустит проверку предиката */
#include <pthread.h>

typedef struct { pthread_mutex_t mu; pthread_cond_t cv; bool ready; } event;

void event_wait_bad(event *e) {
    pthread_mutex_lock(&e->mu);
    if (!e->ready) pthread_cond_wait(&e->cv, &e->mu);
    pthread_mutex_unlock(&e->mu);
}
```

Вариант хуже: вызывать `pthread_cond_wait` без проверки предиката вообще; если сигнал уже был, поток уснёт навсегда. Ниже готовый примитив «событие» с ожиданием до срока. `pthread_cond_timedwait` принимает абсолютное время по часам, заданным при создании переменной: по умолчанию это `CLOCK_REALTIME`, который прыгает при переводе системных часов, поэтому ставят `CLOCK_MONOTONIC`. Срок считают один раз до цикла: иначе каждое ложное пробуждение продлевает ожидание.

```c
/* event.h */
#ifndef EVENT_H
#define EVENT_H
#include <pthread.h>

typedef struct { pthread_mutex_t mu; pthread_cond_t cv; bool ready; } event;

int event_init(event *e);
void event_clear(event *e);
void event_set(event *e);
/* event_wait - ждёт установки события не дольше ms миллисекунд.
 * Возврат: 0 или -ETIMEDOUT. Потоки: потокобезопасна. */
int event_wait(event *e, long ms);
#endif
```

```c
/* event.c */
#define _GNU_SOURCE
#include "event.h"

int event_init(event *e) {
    pthread_condattr_t at;
    pthread_condattr_init(&at);
    pthread_condattr_setclock(&at, CLOCK_MONOTONIC);
    int rc = pthread_cond_init(&e->cv, &at);
    pthread_condattr_destroy(&at);
    pthread_mutex_init(&e->mu, nullptr);
    e->ready = false;
    return -rc;
}

void event_clear(event *e) {
    pthread_cond_destroy(&e->cv);
    pthread_mutex_destroy(&e->mu);
}

void event_set(event *e) {
    pthread_mutex_lock(&e->mu);
    e->ready = true;
    pthread_cond_broadcast(&e->cv);
    pthread_mutex_unlock(&e->mu);
}
```

```c
/* event_wait.c */
#define _GNU_SOURCE
#include "event.h"
#include <time.h>

int event_wait(event *e, long ms) {
    struct timespec dl;
    clock_gettime(CLOCK_MONOTONIC, &dl);
    dl.tv_sec += ms / 1000;
    dl.tv_nsec += (ms % 1000) * 1000000L;
    if (dl.tv_nsec >= 1000000000L) { dl.tv_sec++; dl.tv_nsec -= 1000000000L; }
    int rc = 0;
    pthread_mutex_lock(&e->mu);
    while (!e->ready && rc == 0) rc = pthread_cond_timedwait(&e->cv, &e->mu, &dl);
    bool ok = e->ready;
    pthread_mutex_unlock(&e->mu);
    return ok ? 0 : -rc;
}
```

```c
/* main.c */
#include "event.h"
#include <errno.h>
#include <stdio.h>

static void *setter(void *arg) {
    event_set(arg);
    return nullptr;
}

int main(void) {
    event e;
    if (event_init(&e) != 0) return 1;
    printf("%d\n", event_wait(&e, 20) == -ETIMEDOUT); /* вывод: 1 */
    pthread_t t;
    pthread_create(&t, nullptr, setter, &e);
    printf("%d\n", event_wait(&e, 5000)); /* вывод: 0 */
    pthread_join(t, nullptr);
    event_clear(&e);
}
```

Предикат меняют и сигналят в одной критической секции того же мьютекса: так ожидающий не пропустит изменение между проверкой и сном.

#### Подводные камни

- Предикат изменён без мьютекса. Ожидающий проверил условие, и до его засыпания другой поток поменял поле и просигналил: пробуждение потеряно, поток спит вечно.
- Один condvar с разными мьютексами. Одновременные ожидания на одной переменной с разными мьютексами запрещены, поведение не определено: одна переменная связана с одним мьютексом и одним предикатом (для второго условия заведи вторую переменную).
- `pthread_cond_signal` вместо `broadcast`. `signal` будит одного и годится, когда подойдёт любой ждущий; когда состояние меняется для всех (закрытие, остановка), нужен `broadcast`, иначе часть потоков останется спать.
- Уничтожение переменной с ожидающими. Поведение не определено: сначала разбуди и дождись (`join`) всех ждущих.

### Счётчик ссылок управляет временем жизни, блокировка нет

Блокировка держит структуру согласованной, но не удерживает объект живым: поток, нашедший объект в общей таблице, отпускает замок таблицы и дальше работает с указателем, а другой поток тем временем удаляет объект и освобождает память. Если объект находят другие потоки, нужен счётчик ссылок: тот, кто нашёл, берёт ссылку под замком таблицы (пока объект гарантированно там), а освобождает объект тот, кто снимает последнюю ссылку. Правила пар `ref`/`unref` описаны в [главе о владении](04-memory-ownership.md).

```c
/* Ошибка: указатель пережил замок, объект освобождён (UB: чтение освобождённой памяти) */
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct { int id; } conn;
typedef struct { pthread_mutex_t mu; conn *cur; } slot;

static conn *slot_get(slot *s) {
    pthread_mutex_lock(&s->mu);
    conn *c = s->cur;
    pthread_mutex_unlock(&s->mu);
    return c;
}

static void slot_replace(slot *s, conn *c) {
    pthread_mutex_lock(&s->mu);
    free(s->cur);
    s->cur = c;
    pthread_mutex_unlock(&s->mu);
}

static void *replacer(void *arg) {
    slot_replace(arg, calloc(1, sizeof(conn)));
    return nullptr;
}

int main(void) {
    slot s = {PTHREAD_MUTEX_INITIALIZER, calloc(1, sizeof(conn))};
    conn *c = slot_get(&s);
    pthread_t t;
    pthread_create(&t, nullptr, replacer, &s);
    pthread_join(t, nullptr);
    printf("%d\n", c->id);
    free(s.cur);
}
```

Замок здесь честный, но результатом `slot_get` вызывающий пользуется уже без него, и `slot_replace` в другом потоке успевает освободить тот же объект (в примере поток дожидается `join`, чтобы ошибка проявлялась всегда, а ASan показывает `heap-use-after-free`). Исправление: `slot_get` берёт ссылку под замком, `slot_replace` снимает ссылку слота, а не освобождает.

```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>
#include <stdlib.h>
typedef struct { atomic_int refs; int id; } conn;
typedef struct { pthread_mutex_t mu; conn *cur; } slot;
static conn *conn_create(int id) {
    conn *c = calloc(1, sizeof *c);
    if (!c) abort();
    *c = (conn){1, id};
    return c;
}
static void conn_unref(conn *c) {
    if (atomic_fetch_sub_explicit(&c->refs, 1, memory_order_acq_rel) == 1) free(c);
}

static conn *slot_get(slot *s) {
    pthread_mutex_lock(&s->mu);
    conn *c = s->cur;
    if (c) atomic_fetch_add_explicit(&c->refs, 1, memory_order_relaxed);
    pthread_mutex_unlock(&s->mu);
    return c;
}

static void slot_set(slot *s, conn *c) {
    pthread_mutex_lock(&s->mu);
    conn *old = s->cur;
    s->cur = c;
    pthread_mutex_unlock(&s->mu);
    if (old) conn_unref(old);
}

int main(void) {
    slot s = {PTHREAD_MUTEX_INITIALIZER, conn_create(1)};
    conn *mine = slot_get(&s);
    slot_set(&s, conn_create(2));
    printf("%d %d\n", mine->id, s.cur->id); /* вывод: 1 2 */
    conn_unref(mine);
    slot_set(&s, nullptr);
}
```

Прежний объект пережил замену, потому что у `main` осталась своя ссылка; освободит его последний `conn_unref`. Счётчик атомарный, потому что `unref` вызывают без блокировки слота. Взять ссылку можно с `relaxed`: объект уже виден по замку. Снимать ссылку нужно с `acq_rel`: освобождающий поток обязан увидеть все записи в объект, сделанные другими потоками до их `unref` (это даёт `release`), и свой `free` не может выполниться раньше проверки счётчика (это даёт `acquire`); с `relaxed` освобождающий поток мог бы не увидеть чужие записи и освободить объект, пока они ещё не завершены.

#### Подводные камни

- Ссылка берётся после отпускания замка таблицы. Между поиском и `ref` объект успевают удалить; `ref` всегда внутри той же критической секции, что и поиск.
- Снятие последней ссылки под замком таблицы, когда деструктор объекта долгий или сам берёт замки. Вынеси `unref` из критической секции, как `slot_set` выше.
- Счётчик без ссылки «на себя» в таблице: если таблица владеет объектом, она тоже держит одну ссылку и снимает её при удалении.
- Обычное целое вместо атомарного в `unref`. Два потока читают одно значение и оба уменьшают на единицу: двойное освобождение или утечка.

## Передача работы

### Передавай работу через очередь и пул потоков

Потоки, которые обмениваются сообщениями через очередь, делят меньше состояния, чем потоки, работающие над общими структурами: данные принадлежат одному потоку в каждый момент, а синхронизация сосредоточена в очереди. Потокобезопасная очередь состоит из списка, мьютекса и condvar; на ней строится пул: N рабочих потоков в цикле берут задания. Задание это функция и `user_data` ([callback с `user_data`](07-api-and-modules.md)), владение `user_data` и его время жизни описываются в контракте.

Ниже очередь на мьютексе и condvar с рабочими потоками. Закрытие очереди это флаг под тем же мьютексом плюс `broadcast`: рабочий выходит, только когда очередь закрыта и пуста, поэтому ни одно принятое задание не теряется, а `join` каждого потока гарантирует, что к моменту чтения результата все задания выполнены. `user_data` каждого задания должен пережить его выполнение.

```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
typedef struct job { void (*fn)(void *user_data); void *user_data; struct job *next; } job;
typedef struct { pthread_mutex_t mu; pthread_cond_t cv; job *head, **tail; bool closed; } queue;

static void push(queue *q, void (*fn)(void *), void *user_data) {
    job *j = malloc(sizeof *j);
    if (!j) abort();
    *j = (job){fn, user_data, nullptr};
    pthread_mutex_lock(&q->mu);
    *q->tail = j;
    q->tail = &j->next;
    pthread_cond_signal(&q->cv);
    pthread_mutex_unlock(&q->mu);
}

static void *worker(void *arg) {
    queue *q = arg;
    for (;;) {
        pthread_mutex_lock(&q->mu);
        while (!q->head && !q->closed) pthread_cond_wait(&q->cv, &q->mu);
        job *j = q->head;
        if (j && !(q->head = j->next)) q->tail = &q->head;
        pthread_mutex_unlock(&q->mu);
        if (!j) return nullptr;
        j->fn(j->user_data);
        free(j);
    }
}

static atomic_long sum;
static void add(void *user_data) { atomic_fetch_add(&sum, (long)(intptr_t)user_data); }

int main(void) {
    queue q = {PTHREAD_MUTEX_INITIALIZER, PTHREAD_COND_INITIALIZER, nullptr, &q.head, false};
    pthread_t th[4];
    for (int i = 0; i < 4; i++) pthread_create(&th[i], nullptr, worker, &q);
    for (int i = 0; i < 100; i++) push(&q, add, (void *)(intptr_t)i);
    pthread_mutex_lock(&q.mu);
    q.closed = true;
    pthread_cond_broadcast(&q.cv);
    pthread_mutex_unlock(&q.mu);
    for (int i = 0; i < 4; i++) pthread_join(th[i], nullptr);
    printf("%ld\n", atomic_load(&sum)); /* вывод: 4950 */
}
```

В библиотечном виде очередь и пул оформляют парами `_create`/`_destroy` по канону [главы о владении](04-memory-ownership.md); `push` тогда возвращает `-ENOMEM` или `-EPIPE` (очередь закрыта), а пул не уничтожают из задания этого же пула (рабочий дожидался бы сам себя).

#### Подводные камни

- Задание ждёт результата другого задания того же пула. Когда все рабочие заняты такими заданиями, никто не выполнит нужное: пул встал; ожидающие задания не ставят в тот же пул либо дробят работу иначе.
- Неограниченная очередь. Быстрый производитель съедает всю память; при нагрузке задай предел длины и возвращай `-EAGAIN` или блокируй `push` на втором condvar.
- `user_data` на локальной переменной вызывающего, который уже вышел из функции. Задание читает мёртвую память; владение передают явно (куча, задание освобождает) или гарантируют `join` рабочих до выхода.
- Закрытие очереди через `signal` вместо `broadcast`. Просыпается один рабочий, остальные спят до конца программы и `join` зависает.
- Задание, которое вызывает `exit` или вечно ждёт ввода. Остановку пула блокирует один такой рабочий; блокирующие операции делай с таймаутом.
- Ошибка внутри задания. Задание ничего не возвращает: результат и код ошибки кладут в структуру `user_data`.

## Атомики

### Используй атомики для флагов, счётчиков и статистики

Одиночную переменную, не связанную инвариантом с другими, можно сделать атомарной вместо мьютекса: операции над ней неделимы и не создают гонки. Хватает атомиков для флагов остановки, счётчиков, статистики и счётчика ссылок. Как только значение связано с другими полями (два числа должны меняться вместе), нужен мьютекс. Операции с суффиксом `_explicit` берут порядок памяти; по умолчанию и без `_explicit` действует `memory_order_seq_cst`, самый строгий и безопасный, и менять его без измерений и без понимания не нужно.

```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

static atomic_bool stop;
static atomic_long ticks;

static void *spinner(void *arg) {
    (void)arg;
    while (!atomic_load(&stop)) atomic_fetch_add(&ticks, 1);
    return nullptr;
}

int main(void) {
    pthread_t t;
    pthread_create(&t, nullptr, spinner, nullptr);
    while (atomic_load(&ticks) < 1000) {}
    atomic_store(&stop, true);
    pthread_join(t, nullptr);
    printf("%d\n", atomic_load(&ticks) >= 1000); /* вывод: 1 */
}
```

Статические атомарные объекты без инициализатора обнулены: `stop` равен `false`. Для динамических пользуйся `atomic_init` до публикации объекта другим потокам. Спецификатор `_Atomic(T)` даёт атомарный вариант произвольного (допустимого) типа, а `atomic_bool`, `atomic_long` и прочие это короткие имена для самых частых. `atomic_flag` единственный тип, чья свобода от блокировок гарантирована стандартом, но у него только `test_and_set` и `clear`; для флагов остановки удобнее `atomic_bool`. Разовую инициализацию флагом делать нельзя: потоки, не выигравшие `test_and_set`, идут дальше, пока победитель ещё не закончил `init()`; для этого служит `pthread_once` (раздел «Коротко»).

Каждая атомарная операция атомарна сама по себе, а последовательность двух операций нет: между проверкой и действием другой поток успевает вмешаться.

```c
/* Ошибка: проверка и уменьшение это две операции, между ними вмешивается другой поток */
#include <stdatomic.h>

bool take_one(atomic_int *pending) {
    if (atomic_load(pending) > 0) {
        atomic_fetch_sub(pending, 1);
        return true;
    }
    return false;
}

void bump(atomic_int *c) { *c = *c + 1; }
```

Обе функции проверяют и меняют значение раздельными операциями. В `take_one` два потока видят `1`, оба уменьшают, получается `-1`. `*c = *c + 1` читает и пишет отдельными операциями, поэтому параллельные приращения теряются: два потока по 200000 приращений дают заметно меньше 400000. Любая запись, где атомарная переменная встречается дважды (`a = a + 1`, `if (a > 0) a--`), это две независимые операции, между которыми вмешивается другой поток; `(*c)++` и `atomic_fetch_add` атомарны целиком. Проверка и изменение в одну операцию делается циклом `compare_exchange`.

```c
#include <stdatomic.h>

bool take_one(atomic_int *pending) {
    int n = atomic_load(pending);
    while (n > 0)
        if (atomic_compare_exchange_weak(pending, &n, n - 1)) return true;
    return false;
}
```

`atomic_compare_exchange_weak` при неудаче записывает актуальное значение в `n`, поэтому цикл читает заново сам; `weak` может отказать без причины и потому всегда стоит в цикле.

## Состояние и жизненный цикл

### Не полагайся на скрытое общее состояние

Функция со статическим буфером или собственным невидимым состоянием безопасна в однопоточной программе и портит результаты при двух потоках. К такому относятся статические локальные переменные в своём коде (правило «контекст вместо глобального состояния» в [главе об API](07-api-and-modules.md)) и функции libc, не гарантированные потокобезопасными: `strtok`, `localtime`, `gmtime`, `asctime`, `ctime`, `readdir`, `getenv` при параллельных `setenv`, `rand`, `strerror` и другие. Часть из них glibc сделала безопасными (`strerror`, `rand`), но разделяемая последовательность `rand` делает результат непредсказуемым, а переносимый код не должен на это рассчитывать. Для каждой есть `_r`-вариант с состоянием у вызывающего: `strtok_r`, `localtime_r`, `gmtime_r`, `strerror_r`, `rand_r`.

```c
/* Ошибка: strtok и localtime держат состояние в статической памяти, потоки портят друг другу результат */
#include <stdio.h>
#include <string.h>
#include <time.h>

int count_fields(char *line) {
    int n = 0;
    for (char *t = strtok(line, ","); t; t = strtok(nullptr, ",")) n++;
    return n;
}

void print_hour(time_t ts) {
    struct tm *tm = localtime(&ts);
    printf("%d\n", tm->tm_hour);
}
```

Исправление: состояние передаёт вызывающий. `strtok_r` хранит позицию в `save`, а `localtime_r` пишет в структуру вызывающего, поэтому вызовы из разных потоков не пересекаются. `strtok_r` по-прежнему портит исходную строку; для неизменяемых данных берут `str_token` из [главы о строках](06-strings-and-buffers.md).

```c
#define _GNU_SOURCE
#include <string.h>
#include <time.h>

int count_fields(char *line) {
    char *save;
    int n = 0;
    for (char *t = strtok_r(line, ",", &save); t; t = strtok_r(nullptr, ",", &save)) n++;
    return n;
}

int hour_of(time_t ts) {
    struct tm tm;
    localtime_r(&ts, &tm);
    return tm.tm_hour;
}
```

Отдельный случай: запись лога из нескольких потоков. `FILE` потокобезопасен, каждый вызов `fprintf` блокирует поток вывода на время вызова, но записи двух потоков из нескольких вызовов чередуются между собой. Формируй строку целиком и пиши одним вызовом (устройство такого логгера: [логирование](16-logging.md)).

```c
/* Ошибка: одна запись лога разбита на три вызова, строки двух потоков перемешиваются */
#include <stdio.h>

void log_bad(FILE *f, const char *who, int code) {
    fprintf(f, "[%s] ", who);
    fprintf(f, "code=%d", code);
    fputc('\n', f);
}
```

```c
#include <stdio.h>

int log_line(FILE *f, const char *who, int code) {
    return fprintf(f, "[%s] code=%d\n", who, code);
}
```

Если запись всё-таки состоит из нескольких вызовов, оберни её в `flockfile(f)` и `funlockfile(f)`. Для лога из многих потоков годится и сборка строки в локальном буфере через `snprintf` с последующим одним `fwrite`. Порядок строк разных потоков между собой не определён.

Своя функция, возвращающая указатель на `static`-массив (`const char *name(int id)`), затирает результат при вызовах из разных потоков: возвращай результат через буфер вызывающего. Общую последовательность `rand` в тестах и симуляциях делят все потоки, и она невоспроизводима: держи генератор в контексте потока.

### Передавай потоку собственную копию аргумента и присоединяй его ровно один раз

Поток это ресурс с одним владельцем: ему нужен ровно один `pthread_join` или ровно один `pthread_detach`, а всё, что поток использует, должно пережить его работу. Аргумент потока передают так, чтобы его не затёрли и не освободили раньше времени.

```c
/* Ошибка: потоку передан адрес переменной цикла, она меняется и перестаёт существовать */
#include <pthread.h>

void *worker(void *arg);

void start_all(pthread_t *t, int n) {
    for (int i = 0; i < n; i++) pthread_create(&t[i], nullptr, worker, &i);
}
```

Цикл меняет `i`, пока поток читает его по адресу, а после выхода из `start_all` адрес и вовсе указывает на мёртвый кадр. TSan отмечает первое:

```text
WARNING: ThreadSanitizer: data race (pid=20)
  Write of size 4 at 0x7fffffffebec by main thread:
SUMMARY: ThreadSanitizer: data race idx.c:5 in start_all
```

Исправление: у каждого потока своя ячейка в массиве, который переживает `join`, а число запущенных потоков учитывается: `pthread_create` может вернуть ошибку (код ошибки приходит возвращаемым значением, а не через `errno`).

```c
#include <pthread.h>
#include <stdio.h>

typedef struct { int id; long out; } arg;

static void *square(void *p) {
    arg *a = p;
    a->out = (long)a->id * a->id;
    return nullptr;
}

int main(void) {
    enum { N = 4 };
    pthread_t t[N];
    arg a[N];
    int started = 0;
    while (started < N) {
        a[started] = (arg){started, 0};
        if (pthread_create(&t[started], nullptr, square, &a[started]) != 0) break;
        started++;
    }
    long sum = 0;
    for (int i = 0; i < started; i++) {
        pthread_join(t[i], nullptr);
        sum += a[i].out;
    }
    printf("%ld\n", sum); /* вывод: 14 */
}
```

`thread_local` (переменная со своей копией в каждом потоке) не первый выбор для передачи данных в поток: он прячет зависимость в глобальном имени, функция перестаёт определяться своими аргументами, а в пуле значение остаётся от предыдущего задания. Сначала передай состояние аргументом или контекстом ([контекст вместо глобального состояния](07-api-and-modules.md)); `thread_local` оставь для состояния, которое нельзя протащить через чужой интерфейс. Деструкторов у него нет: память, на которую указывает `thread_local`-указатель, при завершении потока утекает. Если у потока должны быть собственные выделенные данные, заводят ключ `pthread_key_create(&key, free)` и кладут значение через `pthread_setspecific`: при завершении потока `free` вызывается для его значения автоматически.

#### Подводные камни

- Двойной `join`, `join` после `detach` или `join` потока, которого уже нет. Поведение не определено; решение принято один раз при создании: либо поток присоединяют, либо отсоединяют.
- Потока не присоединили и не отсоединили. Ресурсы потока остаются занятыми до конца процесса.
- Отсоединённый поток владеет своим аргументом: аргумент выделяется в куче, и поток освобождает его сам; адрес локальной переменной тут недопустим. `main` не должна завершиться раньше такого потока, иначе процесс уничтожит его на полуслове.
- Выход из потока с захваченным мьютексом (`return`, `pthread_exit`). Мьютекс остаётся занятым навсегда, остальные потоки зависают на нём.
- Уничтожение мьютекса или condvar, пока другой поток ждёт на нём или держит его. Поведение не определено; сначала `join` всех потоков или счётчик ссылок.
- Асинхронная отмена (`pthread_setcanceltype` с `PTHREAD_CANCEL_ASYNCHRONOUS`). Поток может умереть в любой точке, в том числе внутри `malloc` или под мьютексом; при асинхронной отмене безопасны только `pthread_cancel`, `pthread_setcancelstate` и `pthread_setcanceltype`. Останавливай потоки флагом.

## Коротко

### Размечай блокировки для статической проверки Clang

Clang проверяет на этапе компиляции (`-Wthread-safety`), что защищённые данные трогают только с захваченным мьютексом. Атрибуты ставят на тип-обёртку мьютекса (`capability`), на данные (`guarded_by`) и на функции (`requires_capability`, `acquire_capability`, `release_capability`); обёртки над `pthread_mutex_lock` и `unlock` помечают ещё `no_thread_safety_analysis`, потому что функции libc не размечены. В C `guarded_by` привязывается к мьютексу, который указан глобально или статически в файле; ссылка на поле-мьютекс той же структуры с объектом не связывается и даёт ложные предупреждения, поэтому так защищают данные модуля с одним мьютексом. Для GCC атрибуты закрывают макросом, раскрывающимся в ничто.

```c
/* Ошибка: чтение защищённых данных без мьютекса, предупреждение даёт только Clang */
/* Clang 19+ */
#include <pthread.h>
typedef struct __attribute__((capability("mutex"))) { pthread_mutex_t m; } lock;
static lock g_mu;
static long g_total __attribute__((guarded_by(g_mu)));
long counter_get(void) { return g_total; }
```

Сборка с `-Wthread-safety` сообщает `reading variable 'g_total' requires holding mutex 'g_mu'`. Тот же анализ ловит вызов функции с `requires_capability` без мьютекса и выход из функции с захваченным мьютексом.

### Публикуй данные парой release и acquire

Запись с `memory_order_release` и чтение с `memory_order_acquire` одной атомарной переменной образуют пару: всё, что поток записал до `release`, виден потоку, увидевшему это значение через `acquire`. Это самый дешёвый способ передать готовые данные через флаг; мьютекс даёт то же самое по границам критической секции.

```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

static int payload;
static atomic_bool ready;

static void *producer(void *arg) {
    (void)arg;
    payload = 42;
    atomic_store_explicit(&ready, true, memory_order_release);
    return nullptr;
}

int main(void) {
    pthread_t t;
    pthread_create(&t, nullptr, producer, nullptr);
    while (!atomic_load_explicit(&ready, memory_order_acquire)) {}
    printf("%d\n", payload); /* вывод: 42 */
    pthread_join(t, nullptr);
}
```

`payload` обычная переменная, и читать её безопасно именно благодаря паре. Если заменить оба порядка на `memory_order_relaxed`, TSan сообщает гонку на `payload`: `relaxed` гарантирует атомарность самой операции, но не видимость остальной памяти. Он годится для независимых счётчиков и статистики. Остальные порядки (`consume`, `acq_rel` для чтения-изменения-записи, как в счётчике ссылок) нужны редко; без измеренной необходимости оставляй `seq_cst`.

### Для редко меняемых данных и разовой инициализации есть rwlock и pthread_once

`pthread_rwlock` пускает много читателей разом и одного писателя; он окупается, когда чтений много, критическая секция читателя заметно длиннее обычной и записи редки; иначе обычный мьютекс быстрее. `pthread_once` выполняет функцию инициализации один раз, сколько бы потоков ни вызвали её одновременно, и остальные ждут завершения. Применение: `pthread_once(&once, init)` с `once = PTHREAD_ONCE_INIT` в начале функции, которая использует общую таблицу; читатели берут `pthread_rwlock_rdlock`, писатель `pthread_rwlock_wrlock`, и оба снимают `pthread_rwlock_unlock`. Под `-std=c23` типы `pthread_rwlock_t` и `PTHREAD_RWLOCK_INITIALIZER` видны только при `#define _GNU_SOURCE` (первой строкой файла).

### volatile не синхронизация

`volatile` запрещает компилятору выбрасывать и склеивать обращения к переменной, но не делает их атомарными и не упорядочивает остальную память, поэтому флаг между потоками на `volatile` остаётся гонкой данных. Для межпоточных флагов используй `atomic_bool`; `volatile` остаётся для регистров устройств и `volatile sig_atomic_t` в обработчиках сигналов ([сигналы](13-system-io.md)).

```c
/* Ошибка: volatile не даёт ни атомарности, ни порядка (гонка данных) */
static volatile int ready;
static int data;
void publish(void) { data = 42; ready = 1; }
int consume(void) { while (!ready) {} return data; }
```

TSan на такой паре потоков сообщает гонку данных.

### Прерывание и таймауты

Поток останавливают сообщением, а не отменой: атомарный флаг остановки, который рабочий проверяет между порциями работы, либо закрытие очереди (рабочий выходит, когда очередь закрыта и пуста). Блокирующие вызовы должны иметь срок: `pthread_cond_timedwait` с абсолютным временем `CLOCK_MONOTONIC` (практика про condvar), `poll` с таймаутом и дополнительным дескриптором для побудки, чтобы поток, ждущий ввода, можно было разбудить записью в него ([событийный цикл](13-system-io.md)). `pthread_cancel` не используй: отменённый поток освобождает ресурсы только через `pthread_cleanup_push`, которые легко забыть, а асинхронная отмена опасна вдвойне.

## Что ещё стоит знать

Примеры главы написаны на pthread. Lock-free структуры (стек и очередь без блокировок) требуют решения проблемы ABA и безопасного освобождения узлов (hazard pointers, RCU), поэтому их берут готовыми, а не пишут сами. Заголовок `threads.h` (потоки C11: `thrd_create`, `mtx_lock`, `cnd_wait`, `call_once`) есть в glibc 2.39 и работает, но программа на нём под TSan на Clang падает с SEGV. `strerror_r` в glibc с `_GNU_SOURCE` возвращает `char *`, а с `_POSIX_C_SOURCE=200809L` без него `int`, поэтому проверяй, какая версия у тебя. Для счётного ограничения ресурса служит семафор `sem_t` из `semaphore.h`, для одновременного старта группы потоков барьер `pthread_barrier_t`. Готовый пул потоков вместе с циклом событий есть в `libuv` ([готовый цикл событий](13-system-io.md)). Контейнеры и их потокобезопасность описаны в [главе о контейнерах](11-containers-and-generics.md).
