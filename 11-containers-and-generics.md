# Контейнеры и обобщённый код

## Свои контейнеры

### Строй динамический массив из {ptr, len, cap} с ростом удвоением

Три поля дают добавление за амортизированное O(1), доступ по индексу и совместимость с любой функцией, принимающей `(ptr, len)`, поэтому динамический массив служит контейнером по умолчанию: списки, очереди и небольшие множества тоже живут в нём. Рост удвоением делает число `realloc` логарифмическим; умножения размера проверяют `ckd_mul` (C23, `stdckdint.h`), иначе переполнение даёт маленький блок и запись за его границей. Нулевое значение `ivec` допустимо, поэтому `ivec_init` и `ivec_clear` не требуют ничего, кроме самого вектора ([жизненный цикл значения](04-memory-ownership.md)).

```c
/* ivec.h */
#ifndef IVEC_H
#define IVEC_H
#include <stddef.h>

typedef struct { int *ptr; size_t len, cap; } ivec;

void ivec_init(ivec *v);
void ivec_clear(ivec *v);

/* ivec_push - добавляет x в конец.
 * Возврат: 0 или -ENOMEM; при ошибке вектор не изменён.
 * Инвалидирует указатели на элементы: храни индексы. */
[[nodiscard]] int ivec_push(ivec *v, int x);

/* ivec_remove_swap - удаляет i-й элемент, ставя на его место последний.
 * Порядок элементов не сохраняется. Требуется i < len. */
void ivec_remove_swap(ivec *v, size_t i);
#endif
```

```c
/* ivec.c */
#include "ivec.h"
#include <assert.h>
#include <errno.h>
#include <stdckdint.h>
#include <stdlib.h>

void ivec_init(ivec *v) { *v = (ivec){}; }
void ivec_clear(ivec *v) { free(v->ptr); *v = (ivec){}; }

int ivec_push(ivec *v, int x) {
    if (v->len == v->cap) {
        size_t cap, bytes;
        if (ckd_mul(&cap, v->cap ? v->cap : 2, (size_t)2)) return -ENOMEM;
        if (ckd_mul(&bytes, cap, sizeof *v->ptr)) return -ENOMEM;
        int *p = realloc(v->ptr, bytes); /* старый указатель не затирай до проверки */
        if (!p) return -ENOMEM;
        v->ptr = p;
        v->cap = cap;
    }
    v->ptr[v->len++] = x;
    return 0;
}

void ivec_remove_swap(ivec *v, size_t i) {
    assert(i < v->len);
    v->ptr[i] = v->ptr[--v->len];
}
```

```c
/* main.c */
#include "ivec.h"
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    ivec v;
    ivec_init(&v);
    size_t third = 0;
    for (int i = 0; i < 10; i++) {
        if (ivec_push(&v, i * 10) != 0) abort();
        if (i == 3) third = v.len - 1; /* индекс переживёт рост, указатель нет */
    }
    printf("%d\n", v.ptr[third]); /* вывод: 30 */
    ivec_remove_swap(&v, third);
    printf("%d %zu\n", v.ptr[third], v.len); /* вывод: 90 9 */
    ivec_clear(&v);
}
```

Указатель на элемент действует только до следующего добавления: `realloc` вправе перенести блок. Ссылаться на элемент надёжно можно индексом, а вектор с отдельно выделенными элементами (массив указателей) даёт стабильные адреса ценой лишней косвенности.

```c
/* Ошибка: UB, указатель на элемент используется после роста массива */
#include <stdio.h>
#include <stdlib.h>

typedef struct { int *ptr; size_t len, cap; } ivec;

int main(void) {
    ivec v = {};
    int *first = nullptr;
    for (int i = 0; i < 100; i++) {
        if (v.len == v.cap) {
            v.cap = v.cap ? v.cap * 2 : 4;
            v.ptr = realloc(v.ptr, v.cap * sizeof *v.ptr); /* блок мог переехать */
            if (!v.ptr) abort();
        }
        v.ptr[v.len++] = i;
        if (i == 0) first = &v.ptr[0];
    }
    printf("%d\n", *first); /* first указывает в освобождённый блок */
    free(v.ptr);
}
```

#### Подводные камни

- Рост на константу (`cap += 16`). Число копирований квадратично зависит от размера; растить нужно умножением.
- `v->ptr = realloc(v->ptr, n)`. При отказе `nullptr` затирает единственный указатель на блок и память утекает; результат принимают во временную переменную.
- `ivec_remove_swap` меняет порядок. Если порядок важен, сдвигай хвост через `memmove` за O(n); при удалении в цикле индекс не увеличивают, ведь на его месте теперь другой элемент.
- Нулевая длина с `ptr == nullptr`. Передавать такой `ptr` в `memcpy` и подобные функции нельзя даже при длине 0 ([неопределённое поведение](03-types-and-ub.md)), поэтому перед вызовом проверяй `len`.

### Строй хеш-таблицу с единой политикой владения и статусом в lookup

Открытая адресация хранит записи в одном массиве, без выделения памяти на запись, поэтому проходит по кэшу быстрее цепочек; платой служит дисциплина: ёмкость степень двойки, заполнение не выше половины, удаление без «дыр» в цепочке. Таблица владеет копиями ключей и, если задана `free_val`, значениями: одна политика на всю таблицу. Lookup отвечает статусом, а значение отдаёт через out-параметр, чтобы отличить «нет ключа» от значения `nullptr`.

```c
/* map.h */
#ifndef MAP_H
#define MAP_H
#include <stdbool.h>
#include <stddef.h>

typedef struct map map;
typedef void (*map_free_fn)(void *);

/* map_create - таблица "строка -> void *"; ключи копирует и освобождает сама.
 * Значения освобождает free_val (nullptr: значения заимствованы).
 * Возврат: nullptr и -ENOMEM в *err (err допустим nullptr). */
map *map_create(map_free_fn free_val, int *err);
void map_destroy(map *m); /* nullptr допустим */

/* map_put - вставляет или заменяет значение (старое освобождает free_val).
 * Возврат: 0 или -ENOMEM; при ошибке таблица не изменена, val остаётся у вызывающего. */
[[nodiscard]] int map_put(map *m, const char *key, void *val);

/* map_get - true, если ключ есть; значение (возможно nullptr) кладёт в *out. */
bool map_get(const map *m, const char *key, void **out);

/* map_steal - изымает запись без free_val: значение уходит вызывающему. Возврат: 0 или -ENOENT.
 * map_remove - удаляет запись и освобождает значение. */
[[nodiscard]] int map_steal(map *m, const char *key, void **out);
[[nodiscard]] int map_remove(map *m, const char *key);

/* map_next - обход: *pos начинается с 0, порядок не определён.
 * Во время обхода put, remove и steal вызывать нельзя. */
bool map_next(const map *m, size_t *pos, const char **key, void **val);
#endif
```

```c
/* map_int.h */
#include "map.h"
#include <stdint.h>
#include <string.h>

typedef struct { char *key; void *val; } slot; /* key == nullptr: слот пуст */
struct map { slot *s; size_t cap, len; map_free_fn free_val; }; /* len <= cap / 2 */

static inline size_t map_hash(const char *k) {
    uint64_t h = 14695981039346656037u;
    for (; *k; k++) h = (h ^ (uint64_t)(unsigned char)*k) * 1099511628211u;
    return (size_t)h;
}

/* слот с ключом k либо первый пустой на пути пробирования */
static inline slot *map_find(slot *s, size_t cap, const char *k) {
    size_t i = map_hash(k) & (cap - 1);
    while (s[i].key && strcmp(s[i].key, k) != 0) i = (i + 1) & (cap - 1);
    return &s[i];
}
```

```c
/* map.c */
#include "map_int.h"
#include <errno.h>
#include <stdlib.h>

map *map_create(map_free_fn free_val, int *err) {
    map *m = calloc(1, sizeof *m);
    slot *s = calloc(8, sizeof *s);
    if (!m || !s) {
        free(m);
        free(s);
        if (err) *err = -ENOMEM;
        return nullptr;
    }
    *m = (map){.s = s, .cap = 8, .free_val = free_val};
    return m;
}

void map_destroy(map *m) {
    if (!m) return;
    for (size_t i = 0; i < m->cap; i++) {
        if (!m->s[i].key) continue;
        free(m->s[i].key);
        if (m->free_val) m->free_val(m->s[i].val);
    }
    free(m->s);
    free(m);
}

bool map_next(const map *m, size_t *pos, const char **key, void **val) {
    while (*pos < m->cap) {
        const slot *e = &m->s[(*pos)++];
        if (!e->key) continue;
        *key = e->key;
        *val = e->val;
        return true;
    }
    return false;
}
```

```c
/* map_put.c */
#include "map_int.h"
#include <errno.h>
#include <stdckdint.h>
#include <stdlib.h>

static int rehash(map *m, size_t cap) {
    slot *n = calloc(cap, sizeof *n);
    if (!n) return -ENOMEM;
    for (size_t i = 0; i < m->cap; i++)
        if (m->s[i].key) *map_find(n, cap, m->s[i].key) = m->s[i];
    free(m->s);
    m->s = n;
    m->cap = cap;
    return 0;
}

int map_put(map *m, const char *key, void *val) {
    slot *e = map_find(m->s, m->cap, key);
    if (e->key) { /* замена: освобождать нечего, кроме старого значения */
        if (m->free_val && e->val != val) m->free_val(e->val);
        e->val = val;
        return 0;
    }
    char *copy = strdup(key);
    if (!copy) return -ENOMEM;
    if ((m->len + 1) * 2 > m->cap) {
        size_t cap;
        if (ckd_mul(&cap, m->cap, (size_t)2) || rehash(m, cap) != 0) {
            free(copy);
            return -ENOMEM;
        }
        e = map_find(m->s, m->cap, key); /* слот переехал */
    }
    e->key = copy;
    e->val = val;
    m->len++;
    return 0;
}
```

```c
/* map_get.c */
#include "map_int.h"
#include <errno.h>
#include <stdlib.h>

bool map_get(const map *m, const char *key, void **out) {
    const slot *e = map_find(m->s, m->cap, key);
    if (!e->key) return false;
    *out = e->val;
    return true;
}

static void erase(map *m, size_t i) { /* сдвиг назад: дыра оборвала бы цепочку */
    size_t mask = m->cap - 1;
    for (size_t j = (i + 1) & mask; m->s[j].key; j = (j + 1) & mask) {
        size_t home = map_hash(m->s[j].key) & mask;
        if (((j - home) & mask) >= ((j - i) & mask)) {
            m->s[i] = m->s[j];
            i = j;
        }
    }
    m->s[i] = (slot){};
}

int map_steal(map *m, const char *key, void **out) {
    slot *e = map_find(m->s, m->cap, key);
    if (!e->key) return -ENOENT;
    *out = e->val;
    free(e->key);
    erase(m, (size_t)(e - m->s));
    m->len--;
    return 0;
}

int map_remove(map *m, const char *key) {
    void *v;
    int rc = map_steal(m, key, &v);
    if (rc == 0 && m->free_val) m->free_val(v);
    return rc;
}
```

```c
/* main.c */
#include "map.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    map *m = map_create(free, nullptr);
    if (!m) abort();
    char *alpha = strdup("alpha");
    if (!alpha || map_put(m, "a", alpha) != 0) abort();
    if (map_put(m, "none", nullptr) != 0) abort();
    void *v = (void *)1;
    bool has = map_get(m, "none", &v);
    bool found = map_get(m, "zzz", &v);
    printf("%d %d %d\n", has, v == nullptr, found); /* вывод: 1 1 0 */
    for (int i = 0; i < 100; i++) {
        char key[16];
        snprintf(key, sizeof key, "k%d", i);
        if (map_put(m, key, nullptr) != 0) abort();
    }
    if (map_steal(m, "a", &v) != 0 || map_remove(m, "k7") != 0) abort();
    printf("%s\n", (char *)v); /* вывод: alpha */
    free(v); /* после steal значение принадлежит нам */
    size_t pos = 0, n = 0;
    const char *k;
    void *val;
    while (map_next(m, &pos, &k, &val)) n++;
    printf("%zu\n", n); /* вывод: 100 */
    map_destroy(m);
}
```

Ключ копируется до роста таблицы: при `-ENOMEM` таблица остаётся прежней, и вызывающий по-прежнему владеет переданным значением. `map_next` не защищает от вызова `map_put` изнутри цикла: это контракт, а не проверка.

#### Подводные камни

- Заполнение выше половины. Линейное пробирование деградирует в длинные серии, а при полной таблице поиск отсутствующего ключа не завершается; рост по границе `len + 1 > cap / 2` это исключает.
- Удаление пустым слотом. Пустой слот посреди цепочки обрывает поиск, и последующие ключи «исчезают»; нужен сдвиг назад (`erase`) или метка «удалено» с очисткой при росте.
- Плохой хеш. Индекс берётся маской, поэтому хеш с различием только в старших битах складывает ключи в одну цепочку; на недоверенном вводе без случайного зерна атакующий подбирает ключи с одним хешем и вырождает таблицу в линейный поиск.
- Повторная вставка того же указателя значения. Без проверки `e->val != val` замена освободила бы значение, которое сразу же снова сохраняется.

## Обобщённый код и макросы

### Выбирай способ обобщения по числу типов и цене ошибки

Контейнер для `int`, `double` и структур в C пишут тремя способами: `void *` с размером элемента даёт один экземпляр кода, но не проверяет типы; макро-инстанцирование даёт проверку типов и скорость ценой копии кода на каждый тип; `_Generic` выбирает реализацию по типу аргумента из заранее известного набора.

| Способ | Проверка типов | Цена | Когда выбирать |
|---|---|---|---|
| `void *` и размер элемента | нет, приведения на границе | `memcpy` на элемент, вызов компаратора через указатель | много типов, одна готовая реализация, скорость вторична |
| Макро-инстанцирование | полная | код на каждый тип, всё инлайнится | один-два типа в горячем пути |
| `_Generic` | по списку типов | нет, выбор при компиляции | конечный набор встроенных типов; новый тип правит список |

Первый путь знаком по `qsort` и `bsearch`: функция получает `(ptr, len, size)` и работает с байтами.

```c
#include <errno.h>
#include <stdckdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct { void *ptr; size_t len, cap, esz; } gvec;

/* gvec_push - копирует esz байт из elem в конец. Возврат: 0 или -ENOMEM. */
[[nodiscard]] static int gvec_push(gvec *v, const void *elem) {
    if (v->len == v->cap) {
        size_t cap, bytes;
        if (ckd_mul(&cap, v->cap ? v->cap : 2, (size_t)2)) return -ENOMEM;
        if (ckd_mul(&bytes, cap, v->esz)) return -ENOMEM;
        void *p = realloc(v->ptr, bytes);
        if (!p) return -ENOMEM;
        v->ptr = p;
        v->cap = cap;
    }
    memcpy((char *)v->ptr + v->len * v->esz, elem, v->esz);
    v->len++;
    return 0;
}

int main(void) {
    gvec v = {.esz = sizeof(double)};
    for (int i = 0; i < 5; i++) {
        double d = i * 0.5;
        if (gvec_push(&v, &d) != 0) abort();
    }
    printf("%g %zu\n", ((double *)v.ptr)[3], v.len); /* вывод: 1.5 5 */
    free(v.ptr);
}
```

Второй путь: макрос порождает структуру и функции для типа `T`; каждый тип получает собственные имена, а компилятор проверяет их как обычный код и инлайнит.

```c
#include <errno.h>
#include <stdckdint.h>
#include <stdio.h>
#include <stdlib.h>

#define VEC_DEFINE(T, name)                                          \
    typedef struct { T *ptr; size_t len, cap; } name;                \
    [[nodiscard]] static inline int name##_push(name *v, T x) {      \
        if (v->len == v->cap) {                                      \
            size_t cap, bytes;                                       \
            if (ckd_mul(&cap, v->cap ? v->cap : 2, (size_t)2) ||     \
                ckd_mul(&bytes, cap, sizeof(T))) return -ENOMEM;     \
            T *p = realloc(v->ptr, bytes);                           \
            if (!p) return -ENOMEM;                                  \
            v->ptr = p;                                              \
            v->cap = cap;                                            \
        }                                                            \
        v->ptr[v->len++] = x;                                        \
        return 0;                                                    \
    }                                                                \
    static inline void name##_clear(name *v) { free(v->ptr); *v = (name){}; }

VEC_DEFINE(int, intvec)
VEC_DEFINE(const char *, namevec)

int main(void) {
    intvec a = {};
    namevec s = {};
    for (int i = 0; i < 5; i++) if (intvec_push(&a, i * i) != 0) abort();
    if (namevec_push(&s, "x") != 0) abort();
    printf("%zu %d %s\n", a.len, a.ptr[4], s.ptr[0]); /* вывод: 5 16 x */
    intvec_clear(&a);
    namevec_clear(&s);
}
```

Третий путь выбирает функцию по типу аргумента. Ветки `_Generic` перечисляют указатели на функции, а вызов делается после выбора: так в каждой ветке стоит корректное выражение при любом типе аргумента.

```c
#include <stdio.h>

static void put_int(int x) { printf("int %d\n", x); }
static void put_dbl(double x) { printf("double %g\n", x); }
static void put_str(const char *s) { printf("str %s\n", s); }

#define put(x) _Generic((x),    \
    int: put_int,               \
    double: put_dbl,            \
    char *: put_str,            \
    const char *: put_str)(x)

int main(void) {
    const int n = 42;
    put(n);       /* вывод: int 42 */
    put(2.5);     /* вывод: double 2.5 */
    put("text");  /* вывод: str text */
}
```

Выбор делает компилятор, поэтому для типа вне списка получится ошибка компиляции (без ветки `default`) вместо тихого преобразования.

#### Подводные камни

- `void *`: размер и тип не связаны. `int *p = v.ptr` на векторе `double` компилируется молча и читает мусор; на границе модуля оборачивай доступ в типизированные `static inline` функции.
- Макро-инстанцирование: `T` должен быть именем типа, пригодным в `T *` (указатели на функции оформляй через `typedef`), а `name` одним идентификатором. Ошибки компилятора указывают на строку вызова макроса, не на его тело; в заголовочном файле функции делают `static inline`, иначе в каждой единице трансляции появится внешний символ.
- `_Generic`: каждая ветка должна быть корректным выражением независимо от выбранной. Запись `_Generic((x), char *: strlen(x), int: x)` не компилируется для `int` (`strlen(x)` с целым аргументом отвергается и в невыбранной ветке).
- `_Generic`: управляющее выражение теряет квалификаторы, а массив превращается в указатель. `const int` попадает в ветку `int`, строковый литерал в ветку `char *`, массив `int[3]` даёт `int *`, а ветка `const int` никогда не выбирается. Типы `char`, `signed char` и `unsigned char` различны.
- `_Generic`: две ветки с совместимыми типами не компилируются. На Linux x86-64 `int64_t` это `long`, поэтому `long: ..., int64_t: ...` в одном списке отвергается; перечисляй либо встроенные типы, либо псевдонимы фиксированной ширины.
- `_Generic`: `default` скрывает неподдерживаемые типы: функция для `int` молча принимает `float` и `long` через преобразование; ветку `default` пишут, только когда это задумано.

### Пиши макрос как функцию: скобки, один раз, do-while, а лучше static inline

Макрос подставляет текст, не значения: выражение из аргумента раскрывается в каждое место параметра, поэтому без защиты результат зависит от соседних операторов и от числа вычислений. Каждый параметр и всё тело выражения оборачивают в скобки; аргумент с побочным эффектом используют в теле один раз; составной макрос-оператор пишут в `do { ... } while (0)`, чтобы он вёл себя как один оператор рядом с `if`/`else`. Если макрос не нужен для типа, имени или `__LINE__`, пиши `static inline` функцию: она вычисляет аргументы один раз, проверяет типы и видна отладчику.

```c
/* Ошибка: нет скобок, двойное вычисление аргумента, составной макрос без do-while */
#include <stdio.h>

#define SQ(x) x * x
#define ADD(a, b) (a) + (b)
#define MAX(a, b) ((a) > (b) ? (a) : (b))
#define INC2(a, b) (a)++; (b)++

int main(void) {
    int i = 1, x = 0, y = 0;
    printf("%d\n", SQ(i + 1));   /* i + 1 * i + 1 даёт 3, не 4 */
    printf("%d\n", 2 * ADD(1, 2)); /* 2 * 1 + 2 даёт 4, не 6 */
    printf("%d\n", MAX(i++, 0)); /* i++ выполняется дважды: результат 2, i == 3 */
    for (int k = 0; k < 3; k++) INC2(x, y); /* в цикле только x++; y++ выполнится один раз */
    printf("%d %d %d\n", i, x, y);
}
```

Когда нужен именно макрос, временные значения берут в локальные переменные через `typeof` (сохраняет тип с квалификаторами) или `auto` (C23, берёт тип инициализатора без квалификаторов). Имена локальных переменных макроса оформляют по конвенции, например с суффиксом `_`, и в вызывающем коде такие имена не используют. Макрос-оператор с `auto` или `typeof` пишется в стандартном C; выражение, возвращающее значение и вычисляющее аргумент один раз, требует statement expression `({ ... })` (расширение GCC/Clang).

`SWAP` вычисляет каждый аргумент один раз, если брать их адреса: тело работает с указателями `pa_` и `pb_`, а не с текстом аргументов; не годится он только для битовых полей, у которых нет адреса. Имена `pa_`, `pb_` и `t_` зарезервированы макросом.

```c
/* расширение GCC/Clang: statement expression в MAX */
#include <stdio.h>

#define MAX(a, b) ({ auto a_ = (a); auto b_ = (b); a_ > b_ ? a_ : b_; })
#define SWAP(a, b) do { auto pa_ = &(a); auto pb_ = &(b); auto t_ = *pa_; *pa_ = *pb_; *pb_ = t_; } while (0)

int main(void) {
    int v[3] = {1, 2, 3};
    int i = 0;
    if (i == 0) SWAP(v[i++], v[2]); else puts("no");
    printf("%d %d %d %d\n", v[0], v[1], v[2], i); /* вывод: 3 2 1 1 */
    int m = MAX(i++, 0);
    printf("%d %d\n", m, i); /* вывод: 1 2 */
}
```

#### Подводные камни

- Вложенный вызов макроса с локальными переменными. `MAX(x, MAX(y, z))` на statement expression объявляет второй `a_` внутри инициализатора внешнего `b_`, и он затеняет внешний `a_`; `-Wshadow` с `-Werror` отвергает код, поэтому для вычисления значения лучше `static inline` функция.
- Магическая переменная макроса совпала с именем аргумента. `SWAP(x, pa_)` не компилируется, потому что внутренняя `pa_` закрывает переменную вызывающего, а `SWAP(x, t_)` работает, но `-Wshadow` с `-Werror` отвергает код; конвенция имён должна быть написана рядом с макросом.
- Макрос, меняющий поток управления (`return`, `goto`, `break` внутри). Читатель вызова этого не видит; такие макросы пишут заглавными буквами и применяют редко.
- Запятая вне круглых скобок в аргументе. Compound literal `(int[]){1, 2}[0]` внутри вызова макроса делится на два аргумента, и препроцессор сообщает о лишних аргументах; обёрнутый в скобки `((int[]){1, 2})[0]` проходит, как и вызов функции `f(a, b)`.

### Генерируй параллельные таблицы одним X-макросом

Если перечисление и таблица строк (или команд) описывают одно и то же, их правят вместе, а забытая правка тихо ломает соответствие. X-макрос задаёт список один раз как вызовы макроса `X(...)`, а каждое место использования определяет `X` по-своему, раскрывает список и убирает определение. Приём применяют для кодов ошибок модуля ([ошибки](05-errors-and-cleanup.md)), для имён и описаний команд, для перечислений с отладочными именами.

```c
#include <stdio.h>

#define APP_ERRORS(X)                      \
    X(APP_E_PARSE, 1001, "parse error")    \
    X(APP_E_LIMIT, 1002, "limit exceeded") \
    X(APP_E_STATE, 1003, "invalid state")

enum app_err {
#define X(name, value, text) name = value,
    APP_ERRORS(X)
#undef X
};

static const struct { int code; const char *text; } app_errs[] = {
#define X(name, value, text) {name, text},
    APP_ERRORS(X)
#undef X
};

static const char *app_strerror(int code) {
    for (size_t i = 0; i < sizeof app_errs / sizeof *app_errs; i++)
        if (app_errs[i].code == code) return app_errs[i].text;
    return "unknown error";
}

int main(void) {
    puts(app_strerror(APP_E_LIMIT)); /* вывод: limit exceeded */
    puts(app_strerror(7)); /* вывод: unknown error */
    printf("%d\n", APP_E_STATE); /* вывод: 1003 */
}
```

Тот же список может породить и функции: `#define X(n, help) static int cmd_##n(void) { ... }` для каждой записи `COMMANDS(X)`, а вторым раскрытием собирается таблица `{#n, help, cmd_##n}` для диспетчера.

#### Подводные камни

- Забытый `#undef X`. Следующее раскрытие получит чужое определение, а имя `X` может столкнуться с другим макросом; для библиотечных списков берут уникальное имя параметра.
- Имена, собранные через `##`. `cmd_add` нигде не написан целиком, и поиск по тексту его не находит; приём оправдан, когда таблиц две и больше.
- Ошибки раскрытия. Компилятор указывает на строку со списком, а не на запись; читай результат препроцессора (`-E`). Директивы `#if` внутри определения макроса недопустимы: условные записи выносят в отдельный список.
- Дубликаты значений в перечислении. Две записи с одним числом компилируются, а поиск по таблице вернёт первую из них; уникальность кодов проверяют в тесте ([тесты](09-testing.md)).

## Готовые контейнеры

### Выбирай готовую библиотеку по контракту: память, стабильность указателей, владение

Свой массив и хеш-таблица берутся там, где нужен один контейнер и своя политика ошибок; в остальных случаях готовая библиотека экономит дни работы. Четыре варианта различаются не набором функций, а контрактом.

| | stb_ds | klib | uthash | GLib |
|---|---|---|---|---|
| Зависимость | один заголовок | заголовки копируются в проект | один заголовок | системная библиотека |
| Нехватка памяти | не проверяется | `kh_put` возвращает код `ret`; `kvec` не проверяет | `exit` по умолчанию, `HASH_NONFATAL_OOM` | процесс завершается |
| Указатели на элементы | нестабильны при росте | нестабильны при росте | стабильны (интрузивная) | элементы `GArray` нестабильны; ключи и значения `GHashTable` остаются на своих адресах |
| Владение | значения внутри; ключи через `sh_new_strdup` | значения внутри; строковые ключи ваши | элементы ваши, таблица не освобождает | `new_full` с destroy-функциями |
| Лицензия | public domain или MIT | MIT | BSD | LGPL |

Ни у одной из четырёх библиотек контейнеры не защищены от потоков: общий контейнер закрывают снаружи блокировкой ([потоки](12-concurrency.md)).

#### stb_ds

Указатель на первый элемент и есть массив: `int *a = nullptr` готов к `arrput`, длину хранит заголовок перед данными. Реализацию включает `#define STB_DS_IMPLEMENTATION` ровно в одном `.c`. Рост перемещает блок, поэтому функция, добавляющая элементы, получает `int **`; память не проверяется. Для строковых ключей `sh_new_strdup` заставляет таблицу владеть копиями.

```c
#define STB_DS_IMPLEMENTATION
#include "stb_ds.h"
#include <stdio.h>

int main(void) {
    int *a = nullptr;
    for (int i = 0; i < 5; i++) arrput(a, i * i);
    printf("%td %d\n", arrlen(a), a[4]); /* вывод: 5 16 */
    arrdelswap(a, 0); /* удаление с перестановкой */
    printf("%td %d\n", arrlen(a), a[0]); /* вывод: 4 16 */
    struct { char *key; int value; } *m = nullptr;
    sh_new_strdup(m);
    shput(m, "two", 2);
    printf("%d\n", shget(m, "two")); /* вывод: 2 */
    shfree(m);
    arrfree(a);
}
```

#### klib

`khash` и `kvec` порождаются макросами для нужных типов. `kh_put` возвращает итератор и в `ret` код: отрицательный при ошибке, 0 если ключ уже был, положительный для нового. В новом бакете значение не инициализировано, `kh_get` при отсутствии ключа возвращает `kh_end`. Строковые ключи хранятся как указатели: ключом владеет вызывающий. Есть ещё `kvec_t` (`kv_push`, `kv_A`), `kbtree` и `ksort`.

```c
#include "khash.h"
#include <stdio.h>
#include <stdlib.h>

KHASH_MAP_INIT_STR(counts, int)

int main(void) {
    khash_t(counts) *h = kh_init(counts);
    if (!h) abort();
    int ret;
    khint_t k = kh_put(counts, h, "a", &ret); /* таблица хранит указатель на литерал */
    if (ret < 0) abort();
    if (ret > 0) kh_value(h, k) = 0; /* новый бакет: значение не определено */
    kh_value(h, k)++;
    k = kh_get(counts, h, "a");
    printf("%d\n", k != kh_end(h) ? kh_value(h, k) : -1); /* вывод: 1 */
    printf("%d\n", kh_get(counts, h, "z") == kh_end(h)); /* вывод: 1 */
    kh_destroy(counts, h);
}
```

#### uthash

Интрузивная таблица: структура элемента содержит поле `UT_hash_handle hh`, голова таблицы это указатель, равный `nullptr` для пустой. Элементы принадлежат вам и остаются на месте при любых вставках, поэтому на них можно держать указатели. `HASH_ADD_INT` не проверяет уникальность ключа: сначала `HASH_FIND_INT`, либо `HASH_REPLACE_INT`. `HASH_ITER` безопасен при удалении текущего элемента. При нехватке памяти по умолчанию вызывается `exit`; с `#define HASH_NONFATAL_OOM 1` элемент не вставляется, а вызывается макрос `uthash_nonfatal_oom`, в котором ошибку фиксируют.

```c
#include "uthash.h"
#include <stdio.h>
#include <stdlib.h>

typedef struct { int id; int score; UT_hash_handle hh; } user;

int main(void) {
    user *users = nullptr, *u, *tmp, *first = nullptr;
    for (int id = 1; id < 1000; id++) {
        u = calloc(1, sizeof *u);
        if (!u) abort();
        u->id = id;
        u->score = id * 10;
        HASH_ADD_INT(users, id, u); /* уникальность ключа не проверяется */
        if (id == 1) first = u; /* указатель переживёт любые вставки */
    }
    printf("%d %u\n", first->score, HASH_COUNT(users)); /* вывод: 10 999 */
    HASH_ITER(hh, users, u, tmp) { HASH_DEL(users, u); free(u); }
}
```

#### GLib

`GHashTable` не копирует ключи и значения; `g_hash_table_new_full` задаёт функции освобождения для ключа и значения, а без них всё остаётся на вашей совести. `g_hash_table_lookup` не различает «нет ключа» и значение `nullptr`: для различия есть `g_hash_table_lookup_extended`. `g_hash_table_steal_extended` изымает запись без вызова деструкторов и отдаёт ключ и значение. `g_autoptr` и `g_autofree` опираются на атрибут `cleanup` (расширение GCC/Clang) и освобождают объект при выходе из области видимости. `g_malloc` при нехватке памяти завершает процесс.

```c
#include <glib.h>
#include <stdio.h>

int main(void) {
    g_autoptr(GHashTable) t = g_hash_table_new_full(g_str_hash, g_str_equal, g_free, g_free);
    g_hash_table_insert(t, g_strdup("a"), g_strdup("x"));
    g_hash_table_insert(t, g_strdup("n"), nullptr);
    gpointer v = nullptr, k = nullptr;
    printf("%d\n", g_hash_table_lookup(t, "zzz") == nullptr); /* вывод: 1 */
    printf("%d\n", g_hash_table_lookup_extended(t, "n", nullptr, &v)); /* вывод: 1 */
    printf("%d\n", g_hash_table_lookup_extended(t, "zzz", nullptr, &v)); /* вывод: 0 */
    if (g_hash_table_steal_extended(t, "a", &k, &v)) {
        printf("%s\n", (char *)v); /* вывод: x */
        g_free(k);
        g_free(v);
    }
}
```

Остальное в GLib устроено так же: `GArray` хранит элементы по значению, `GPtrArray` указатели (компаратор сортировки получает указатели на элементы, то есть на указатели), `GString` растёт при добавлении, а `g_string_free(s, FALSE)` отдаёт буфер вызывающему.

#### Подводные камни

- stb_ds: `shget(m, k)` и `hmget` присваивают переменную таблицы, поэтому `printf` с `shget(m, k)` и `shlen(m)` в одном вызове даёт предупреждение о неупорядоченном изменении (с `-Werror` это ошибка); пиши отдельными операторами.
- khash: итераторы и указатели на значения невалидны после `kh_put`. `kh_del` не освобождает строковый ключ: освободи его до удаления, иначе утечка.
- uthash: ключ менять нельзя, пока элемент в таблице; `HASH_DEL` не освобождает элемент, а `HASH_CLEAR` не освобождает элементы вообще; составной ключ перед вставкой обнуляют `memset`, чтобы не хешировать заполнение.
- GLib: `g_hash_table_insert` при совпадении ключа освобождает переданный новый ключ и сохраняет старый, а `g_hash_table_replace` заменяет ключ; путаница даёт либо утечку, либо двойное освобождение.

### Проверь пять общих ловушек любого контейнера до того, как положишься на него

Эти вопросы одинаковы для массива, хеш-таблицы и списка из любой библиотеки; ответы записывают в контракт. Свои контейнеры выше отвечают на все пять.

- Кто владеет элементами. Политика единая на весь контейнер и задаётся при создании ([владение](04-memory-ownership.md)): в GLib пары destroy-функций у `g_hash_table_new_full`, в stb_ds `sh_new_strdup`, в `uthash` элементы всегда ваши.
- Как отличить отсутствие ключа от значения `nullptr`. `g_hash_table_lookup_extended` и `map_get` решают это отдельным статусом, а `kh_get` и `hmgeti` возвращают позицию или признак «не найдено».
- Что нельзя менять во время обхода. Вставка и удаление в цикле допустимы только средствами самого контейнера: `HASH_ITER` с запасным указателем, `g_hash_table_foreach_remove`, `g_hash_table_iter_remove`.
- Когда указатели на элементы становятся недействительными. Любой рост массива или таблицы переносит данные; храни индексы и ключи либо используй стабильные структуры (`uthash`, интрузивный список, массив указателей на отдельные элементы).
- Что делает steal. Изъятие записи без деструктора передаёт владение вызывающему: `map_steal`, `g_hash_table_steal_extended`. В klib и uthash деструкторов нет, поэтому `kh_del` и `HASH_DEL` всегда steal: ключ и элемент освобождаете вы.

### Применяй интрузивный список, когда нужны O(1) удаление по элементу и несколько списков

Интрузивный список вкладывает узел `{prev, next}` прямо в объект, поэтому не выделяет память под узлы, удаляет объект за O(1) по одному указателю и позволяет объекту состоять в нескольких списках сразу; по узлу объект восстанавливает `container_of`. Связный список оправдан редко (массив почти всегда выигрывает по кэшу и простоте): его берут для очередей таймеров, LRU, списков ожидания, где объекты живут отдельно и перемещаются между коллекциями.

```c
#include <stddef.h>
#include <stdio.h>

typedef struct node { struct node *prev, *next; } node;
#define container_of(p, T, m) ((T *)((char *)(p) - offsetof(T, m)))

static void list_init(node *h) { h->prev = h->next = h; }
static void list_add_tail(node *h, node *n) {
    n->prev = h->prev; n->next = h;
    h->prev->next = n; h->prev = n;
}
static void list_del(node *n) {
    n->prev->next = n->next; n->next->prev = n->prev;
    list_init(n); /* после удаления узел снова допустим */
}

typedef struct { int id; node all; node ready; } task;

int main(void) {
    node all, ready;
    list_init(&all);
    list_init(&ready);
    task t[3] = {{.id = 1}, {.id = 2}, {.id = 3}};
    for (int i = 0; i < 3; i++) {
        list_add_tail(&all, &t[i].all);
        if (i != 1) list_add_tail(&ready, &t[i].ready);
    }
    list_del(&t[2].all); /* из одного списка, второй не тронут */
    int sum = 0;
    for (node *n = ready.next; n != &ready; n = n->next) sum += container_of(n, task, ready)->id;
    printf("%d\n", sum); /* вывод: 4 */
    for (node *n = all.next, *nx; n != &all; n = nx) { /* nx сохранён до удаления */
        nx = n->next;
        list_del(n);
    }
}
```

#### Подводные камни

- Объект уничтожен, не будучи изъят из списка: остаётся висячий узел; перед освобождением вызывай `list_del` для каждого его узла.
- Объекты лежат в растущем массиве: рост переносит их, и ссылки соседних узлов указывают в старую память.
- Удаление в цикле: после `list_del(n)` читать `n->next` нельзя, следующий узел сохраняют заранее.
- `container_of` с неверным типом или полем даёт указатель мимо объекта; компилятор это не проверяет.

## Коротко

### Считай размеры в size_t и проверяй произведения

Длина, ёмкость и индекс это `size_t`; произведение числа элементов на размер проверяют `ckd_mul` либо берут `reallocarray` (glibc, требует `_GNU_SOURCE`), который сам возвращает `nullptr` при переполнении.

### Сравнивай в компараторе без вычитания и ищи через bsearch

Компаратор для `qsort` и `bsearch` получает указатели на элементы; возвращать `a - b` нельзя, потому что разность переполняется для далёких значений `int`, а разность `double` при приведении к `int` обрезается до нуля. Надёжная запись: `(x > y) - (x < y)`. Массив обязан быть отсортирован тем же компаратором, иначе результат `bsearch` непредсказуем; он возвращает указатель на элемент или `nullptr`.

```c
#include <stdio.h>
#include <stdlib.h>

static int cmp_int(const void *pa, const void *pb) {
    int a = *(const int *)pa, b = *(const int *)pb;
    return (a > b) - (a < b);
}

int main(void) {
    int v[] = {5, -2, 9, 0, 3}, key = 3;
    qsort(v, 5, sizeof *v, cmp_int);
    int *p = bsearch(&key, v, 5, sizeof *v, cmp_int);
    printf("%d %td\n", v[0], p ? p - v : -1); /* вывод: -2 2 */
}
```

### Давай функции и макросу одно имя, а строки собирай через двухуровневый макрос

Функциональный макрос раскрывается только перед `(`, поэтому имя функции без скобок остаётся функцией (её можно взять по адресу и передать как callback), а `(sq)(5)` обходит макрос; `default: sq` указывает на функцию, потому что внутри своего раскрытия макрос не раскрывается. Оператор `#x` превращает аргумент в строку, `a##b` склеивает токены (так `name##_push` строит имена), но оба применяются до раскрытия аргумента: для значения макроса нужна обёртка `STR(x)` над `STR_(x)`, иначе `STR_(PORT)` даёт `"PORT"`, а `STR_(__LINE__)` строку `"__LINE__"`. `__VA_OPT__` (C23) подставляет текст только при непустом `...` и убирает лишнюю запятую; законченный логирующий макрос разобран в [паттернах](21-patterns.md).

```c
#include <stdio.h>

static inline int sq(int x) { return x * x; }
static inline double sq_d(double x) { return x * x; }
#define sq(x) _Generic((x), double: sq_d, default: sq)(x)
#define STR_(x) #x
#define STR(x) STR_(x)
#define PORT 8080

int main(void) {
    int (*f)(int) = sq; /* без скобок макрос не раскрывается: это функция */
    printf("%d %g %d\n", sq(3), sq(1.5), f(4)); /* вывод: 9 2.25 16 */
    puts(STR_(PORT)); /* вывод: PORT */
    puts(STR(PORT)); /* вывод: 8080 */
}
```

## Что ещё стоит знать

`GQueue` и `GTree` из GLib дают очередь и сбалансированное дерево; `GQueue` не потокобезопасна, для потоков есть `GAsyncQueue`. Рядом с `uthash` лежат `utlist` (списки), `utarray` (массив с копированием элементов внутрь) и `utstring`. Clang поддерживает `__attribute__((overloadable))`: перегрузку функций по типам аргументов, как альтернативу `_Generic` (расширение Clang).
