# Техники текстового редактора

Глава состоит из независимых техник: каждая стоит на своём примере и читается отдельно. Всё реализуется самостоятельно; примеры используют массивы фиксированной ёмкости и `abort()` при переполнении ради краткости, а в рабочем коде массивы растут ([контейнеры](09-containers-and-generics.md)), а ошибки возвращаются по схемам главы [ошибки](04-errors-and-cleanup.md).

## Хранение текста и позиции

### Выбирай представление буфера по шаблону правок

Массив байт с `memmove` на каждую вставку тормозит уже на мегабайтах, поэтому текст хранят в структуре, где правка не сдвигает весь хвост. Выбор определяют три вопроса: сколько правок идёт в разных местах одновременно, нужны ли дешёвые снимки для undo и загружается ли большой файл целиком.

Gap buffer - массив с «щелью» свободного места в позиции курсора: вставка и удаление у курсора стоят O(1), а прыжок курсора переносит щель одним `memmove`. Piece table хранит исходный текст неизменным (его можно отобразить в память, см. [системный ввод-вывод](11-system-io.md)), все вставки дописывает в отдельный буфер, а документом считает список кусков «буфер, смещение, длина»; правка меняет только список. Rope - сбалансированное дерево кусков с длинами в узлах: любая позиция находится за O(log n), а при неизменяемых узлах снимок равен указателю на корень.

| Структура | Вставка у курсора | Правка в произвольном месте | Снимок для undo | Сложность |
| --- | --- | --- | --- | --- |
| gap buffer | O(1) | O(n): перенос щели | копия текста или журнал правок | минимальная |
| piece table | O(число кусков) | O(число кусков) | копия списка кусков, текст не копируется | средняя |
| rope | O(log n) | O(log n) | O(1) при общих поддеревьях | высокая |

Для текста до десятков мегабайт с одним курсором хватает gap buffer. Piece table берут, когда нужны дешёвые снимки или большой файл с неизменяемым оригиналом; когда кусков станут десятки тысяч, их список превращают в дерево, то есть в rope. Ниже полная piece table: правка собирает новый список кусков, а undo возвращает предыдущий.

```c
/* ptab.h */
#ifndef PTAB_H
#define PTAB_H
#include <stddef.h>

typedef struct { int add; size_t off, len; } piece; /* add: 0 - исходный буфер, 1 - буфер добавлений */
typedef struct { piece *p; size_t n; } pieces;
typedef struct {
    const char *orig;
    char add[64]; /* только растёт, поэтому смещения в нём не устаревают */
    size_t addlen;
    pieces cur, hist[8]; /* hist: прежние списки кусков, по одному на правку */
    size_t nhist;
} ptab;

ptab pt_init(const char *orig, size_t len);
/* pt_edit - заменяет del байт с позиции pos на n байт s. Требуется pos + del <= длины текста. */
void pt_edit(ptab *t, size_t pos, size_t del, const char *s, size_t n);
void pt_undo(ptab *t);
size_t pt_text(const ptab *t, char *out); /* без завершающего нуля; возвращает длину */
void pt_free(ptab *t);
#endif
```

```c
/* ptab.c */
#include "ptab.h"
#include <stdlib.h>
#include <string.h>

static void keep(pieces *o, piece x, size_t from, size_t to) {
    if (to > from) o->p[o->n++] = (piece){x.add, x.off + from, to - from};
}

void pt_edit(ptab *t, size_t pos, size_t del, const char *s, size_t n) {
    if (t->nhist == 8 || t->addlen + n > sizeof t->add) abort();
    memcpy(t->add + t->addlen, s, n);
    piece ins = {1, t->addlen, n};
    t->addlen += n;
    pieces o = {malloc((t->cur.n + 2) * sizeof(piece)), 0};
    if (!o.p) abort();
    bool put = false;
    size_t at = 0; /* позиция начала текущего куска в тексте */
    for (size_t i = 0; i < t->cur.n; i++) {
        piece x = t->cur.p[i];
        size_t l = pos > at ? pos - at : 0, r = pos + del > at ? pos + del - at : 0;
        keep(&o, x, 0, l < x.len ? l : x.len); /* часть до правки */
        if (!put && pos <= at + x.len) { keep(&o, ins, 0, n); put = true; }
        keep(&o, x, r, x.len); /* часть после удалённого */
        at += x.len;
    }
    if (!put) keep(&o, ins, 0, n);
    t->hist[t->nhist++] = t->cur;
    t->cur = o;
}
```

```c
/* ptab_util.c */
#include "ptab.h"
#include <stdlib.h>
#include <string.h>

ptab pt_init(const char *orig, size_t len) {
    ptab t = {.orig = orig};
    t.cur.p = malloc(sizeof(piece));
    if (!t.cur.p) abort();
    if (len) t.cur.p[t.cur.n++] = (piece){0, 0, len};
    return t;
}

void pt_undo(ptab *t) {
    if (!t->nhist) return;
    free(t->cur.p);
    t->cur = t->hist[--t->nhist];
}

size_t pt_text(const ptab *t, char *out) {
    size_t n = 0;
    for (size_t i = 0; i < t->cur.n; i++) {
        piece x = t->cur.p[i];
        memcpy(out + n, (x.add ? t->add : t->orig) + x.off, x.len);
        n += x.len;
    }
    return n;
}

void pt_free(ptab *t) {
    free(t->cur.p);
    for (size_t i = 0; i < t->nhist; i++) free(t->hist[i].p);
}
```

```c
/* main.c */
#include "ptab.h"
#include <stdio.h>

static void show(const ptab *t) {
    char out[64];
    size_t n = pt_text(t, out);
    printf("%.*s | pieces: %zu\n", (int)n, out, t->cur.n);
}

int main(void) {
    ptab t = pt_init("hello world", 11);
    pt_edit(&t, 5, 0, ",", 1); show(&t); /* вывод: hello, world | pieces: 3 */
    pt_edit(&t, 7, 5, "C", 1); show(&t); /* вывод: hello, C | pieces: 4 */
    pt_edit(&t, 0, 7, "", 0); show(&t); /* вывод: C | pieces: 1 */
    pt_undo(&t); pt_undo(&t); show(&t); /* вывод: hello, world | pieces: 3 */
    pt_free(&t);
    return 0;
}
```

Куски неизменяемы, а буфер добавлений только растёт, поэтому старый список остаётся корректным снимком: undo не копирует и не пересчитывает текст. Правка, которая ничего не удаляет, добавляет не больше двух кусков, а редактирование внутри одного куска делит его пополам.

#### Подводные камни

- Число кусков растёт с каждой правкой, и поиск по позиции линеен. Склеивай соседние куски, идущие подряд в буфере добавлений (набор текста продлевает последний кусок), и при росте списка переходи на дерево.
- В буфере добавлений храни смещения, а не указатели: при росте массива указатели в него становятся висячими ([владение и аллокация](03-memory-ownership.md)).
- `pos` и `del` измеряются в байтах и должны лежать на границах последовательностей UTF-8 ([строки](05-strings-and-buffers.md)); правка посреди последовательности даёт невалидный текст.
- Каждый снимок хранит копию списка кусков: ограничивай глубину истории или храни не списки, а обратные правки (см. раздел про undo).
- Структуру проверяй модельным тестом ([тесты](07-testing-and-debugging.md)): случайные правки применяй и к piece table, и к обычному массиву, после каждой сравнивай текст и undo.

### Храни позицию в байтах, а строки и столбцы вычисляй

Байтовое смещение единственное, что сравнивается, складывается и передаётся в `memcpy` без обхода текста, поэтому оно служит канонической позицией: курсор, метки и правки хранят только его. Номер строки получают бинарным поиском по индексу начал строк, столбец отображения считают при отрисовке и только для видимых строк.

Индекс начал строк - отсортированный массив смещений, `start[0] == 0`. Строку позиции находят поиском последнего начала, не превышающего позицию. При правке индекс обновляют, а не строят заново: начала после точки правки сдвигаются на дельту, начала внутри удалённого диапазона исчезают, переводы строк во вставленном тексте добавляют новые начала.

```c
/* lines.h */
#ifndef LINES_H
#define LINES_H
#include <stddef.h>

typedef struct { size_t start[32], n; } lines; /* start[0] == 0; ёмкость фиксирована для краткости */

lines ln_build(const char *s, size_t len);
size_t ln_find(const lines *l, size_t pos); /* строка, содержащая байт pos */
void ln_insert(lines *l, size_t pos, const char *s, size_t n);
void ln_delete(lines *l, size_t pos, size_t n);
#endif
```

```c
/* lines.c */
#include "lines.h"
#include <stdlib.h>
#include <string.h>

lines ln_build(const char *s, size_t len) {
    lines l = {.n = 1};
    for (size_t i = 0; i < len; i++)
        if (s[i] == '\n') l.start[l.n++] = i + 1;
    return l;
}

size_t ln_find(const lines *l, size_t pos) {
    size_t lo = 0, hi = l->n; /* инвариант: start[lo] <= pos, start[hi] > pos */
    while (hi - lo > 1) {
        size_t mid = lo + (hi - lo) / 2;
        if (l->start[mid] <= pos) lo = mid; else hi = mid;
    }
    return lo;
}

void ln_insert(lines *l, size_t pos, const char *s, size_t n) {
    size_t i = ln_find(l, pos) + 1, k = 0;
    for (size_t j = i; j < l->n; j++) l->start[j] += n;
    for (size_t j = 0; j < n; j++) k += s[j] == '\n';
    if (l->n + k > 32) abort();
    memmove(l->start + i + k, l->start + i, (l->n - i) * sizeof l->start[0]);
    for (size_t j = 0, m = i; j < n; j++)
        if (s[j] == '\n') l->start[m++] = pos + j + 1;
    l->n += k;
}

void ln_delete(lines *l, size_t pos, size_t n) {
    size_t i = ln_find(l, pos) + 1, j = i;
    while (j < l->n && l->start[j] <= pos + n) j++; /* начала в (pos, pos + n] исчезают */
    memmove(l->start + i, l->start + j, (l->n - j) * sizeof l->start[0]);
    l->n -= j - i;
    for (j = i; j < l->n; j++) l->start[j] -= n;
}
```

```c
/* main.c */
#include "lines.h"
#include <stdio.h>
#include <string.h>

static void dump(const lines *l) {
    for (size_t i = 0; i < l->n; i++) printf("%zu%c", l->start[i], i + 1 < l->n ? ' ' : '\n');
}

int main(void) {
    lines l = ln_build("ab\ncd\nef", 8);
    dump(&l); /* вывод: 0 3 6 */
    ln_insert(&l, 4, "X\nY", 3); /* текст: ab\ncX\nYd\nef */
    dump(&l); /* вывод: 0 3 6 9 */
    printf("%zu\n", ln_find(&l, 7)); /* вывод: 2 */
    ln_delete(&l, 2, 4); /* текст: abYd\nef */
    dump(&l); /* вывод: 0 5 */
    lines w = ln_build("abYd\nef", 7);
    printf("%d\n", l.n == w.n && !memcmp(l.start, w.start, l.n * sizeof l.start[0])); /* вывод: 1 */
    return 0;
}
```

Последняя проверка сверяет инкрементальный результат с перестроенным с нуля индексом; в тестах это основной способ поймать ошибку сдвига. Обновление стоит O(число строк после правки) из-за сдвига; для сотен тысяч строк индекс делят на блоки или хранят длины строк в дереве.

Столбец отображения зависит от табуляций и ширины символов, поэтому его не хранят: считают обходом строки от начала до смещения. Табуляция доводит столбец до ближайшей кратной ширины позиции, ширину остальных символов даёт `wcwidth` (0 для комбинирующих знаков, 2 для иероглифов). Функция зависит от `LC_CTYPE`, поэтому `setlocale` вызывает только `main`, а библиотечный код локаль не трогает ([локаль](05-strings-and-buffers.md)).

```c
#define _GNU_SOURCE
#include <locale.h>
#include <stdio.h>
#include <string.h>
#include <wchar.h>

/* vcol - столбец отображения для байтового смещения off; битый байт занимает один столбец */
size_t vcol(const char *s, size_t off, size_t tab) {
    mbstate_t st = {};
    size_t col = 0;
    for (size_t i = 0, n; i < off; i += n) {
        wchar_t wc;
        n = mbrtowc(&wc, s + i, off - i, &st);
        if (n == (size_t)-1 || n == (size_t)-2) { n = 1; col++; st = (mbstate_t){}; continue; }
        if (n == 0) n = 1;
        if (wc == L'\t') col += tab - col % tab;
        else if (wcwidth(wc) > 0) col += (size_t)wcwidth(wc);
    }
    return col;
}

int main(void) {
    if (!setlocale(LC_CTYPE, "C.UTF-8")) return 1;
    const char *s = "a\t\xd0\xb1\xe4\xb8\x96" "e\xcc\x81x"; /* a, табуляция, б, иероглиф, e с акцентом, x */
    size_t offs[] = {1, 2, 4, 7, 8, 10, 11};
    for (size_t i = 0; i < 7; i++) printf("%zu ", vcol(s, offs[i], 4));
    printf("\n"); /* вывод: 1 4 5 7 8 8 9 */
    return 0;
}
```

Обратное отображение (клик мышью по столбцу) делают тем же обходом: идут по символам, пока столбец не превысит заданный, и берут границу последовательности.

#### Подводные камни

- `wcwidth` возвращает -1 для управляющих символов и зависит от версии таблиц Unicode в libc: при другом `LC_CTYPE` (не UTF-8) ширины иные, а вывод теста перестаёт быть детерминированным. Задавай локаль явно, как в примере.
- Кластеры графем (эмодзи с модификаторами, флаги, буква с несколькими акцентами) занимают одно место на экране, но состоят из нескольких кодовых точек: покодовое перемещение курсора остановится внутри кластера. Для границ кластеров нужны таблицы Unicode; без них курсор перемещают по кодовым точкам и документируют это ограничение.
- Перевод строки `\r\n` индекс считает по `\n`, а `\r` остаётся байтом конца строки; решение «показывать ли его» принимает отрисовка, не индекс.
- Хранить номер строки и столбец в метках нельзя: любая правка выше их обесценивает. Хранится смещение, строка и столбец выводятся из него.
- Смещение должно лежать на границе последовательности UTF-8: `mbrtowc` на середине даёт ошибку, и столбец становится приблизительным.

### Привязывай курсор и диагностики к правкам дельтой

Метка, хранящая смещение, после правки выше неё указывает не туда, поэтому каждая правка описывается дельтой «позиция, сколько удалено, сколько вставлено», и все живые метки пересчитываются одной функцией. Так живут курсор, выделение (две метки: якорь и голова), диапазоны диагностик и закладки.

Правило сдвига простое. Метка левее правки не меняется, правее удалённого диапазона смещается на разность вставленного и удалённого, внутри удалённого схлопывается в позицию правки. Неоднозначна только метка ровно в точке вставки: она либо остаётся перед вставленным текстом, либо уходит за него. Это притяжение задают параметром: голова курсора уходит за набираемый текст, начало диагностики тоже, конец диагностики остаётся.

Несколько курсоров правят в порядке убывания смещения: правка в конце документа не меняет позиций ранних курсоров, поэтому позиции каждой правки остаются верными в исходных координатах, а уже обработанные курсоры справа получают сдвиг от той же функции.

```c
#include <stdio.h>
#include <string.h>

typedef struct { size_t pos, del, ins; } edit; /* заменить del байт с pos на ins байт */
typedef struct { size_t start, end; } range;

/* after: метка в точке вставки уходит за вставленный текст */
static size_t mark_shift(size_t m, edit e, bool after) {
    if (m < e.pos || (m == e.pos && e.del == 0 && !after)) return m;
    if (m >= e.pos + e.del) return m - e.del + e.ins;
    return e.pos + (after ? e.ins : 0); /* метка внутри удалённого */
}

static void apply(char *b, size_t *len, edit e, const char *s) {
    memmove(b + e.pos + e.ins, b + e.pos + e.del, *len - e.pos - e.del);
    memcpy(b + e.pos, s, e.ins);
    *len = *len - e.del + e.ins;
}

int main(void) {
    char b[64] = "aaa bbb ccc";
    size_t len = strlen(b), cur[] = {3, 7, 11};
    range diag = {4, 7}; /* "bbb" */
    for (size_t i = 3; i-- > 0;) { /* по убыванию смещения */
        edit e = {cur[i], 0, 1};
        apply(b, &len, e, "!");
        for (size_t j = 0; j < 3; j++) cur[j] = mark_shift(cur[j], e, true);
        diag.start = mark_shift(diag.start, e, true);
        diag.end = mark_shift(diag.end, e, false);
    }
    printf("%.*s\n", (int)len, b); /* вывод: aaa! bbb! ccc! */
    printf("%zu %zu %zu\n", cur[0], cur[1], cur[2]); /* вывод: 4 9 14 */
    printf("[%.*s]\n", (int)(diag.end - diag.start), b + diag.start); /* вывод: [bbb] */
    return 0;
}
```

Диагностика осталась на слове `bbb`, хотя перед ним и после него вставлены символы; вставка ровно в конец диапазона в него не вошла.

#### Подводные камни

- Удаление, накрывающее диапазон целиком, схлопывает его в пустой: такую диагностику отбрасывай, пустое выделение превращай в курсор.
- Притяжение метки в точке вставки выбирай осознанно для каждого вида метки: курсор уходит за набранное, закладка остаётся на месте.
- После правок несколько курсоров могут совпасть: убирай дубликаты после каждой операции, иначе следующая вставка пройдёт дважды в одной точке.
- Пересчёт всех меток на каждую правку стоит O(число меток). Для тысяч диагностик хватает массива, для сотен тысяч метки держат в дереве интервалов или в самом буфере.
- Undo - тоже правка: его дельта обязана пройти через ту же функцию, иначе метки разойдутся с текстом.

## Подсветка исходного кода

### Храни лексическое состояние на начало строки и перелексивай до совпадения

Подсвечивать весь файл после каждой буквы дорого, а подсвечивать одну строку нельзя: многострочный комментарий в строке выше меняет смысл строк ниже. Решение: лексер, у которого всё влияющее на следующую строку сведено к небольшому состоянию, и кэш состояния на конец каждой строки. После правки лексят строки от изменённой, пока состояние на конце строки не совпадёт с закэшированным: дальше всё осталось прежним.

Подсветку для отрисовки не хранят: отрисовка берёт состояние на начало видимой строки из кэша, запускает лексер на ней одной и получает классы байт. Лексер не разбирает грамматику, он различает комментарии, строки и тому подобное, и не должен отказывать ни на каком входе: текст при правке почти всегда некорректен.

```c
/* hl.h */
#ifndef HL_H
#define HL_H
#include <stddef.h>

typedef struct { const char *ptr; size_t len; } str;
enum { HL_CODE, HL_COMMENT }; /* состояние на границе строк */

typedef struct { const str *line; size_t n; unsigned char *end; } hl; /* end[i]: состояние после строки i */

/* hl_line - лексит строку из состояния in; при cls != nullptr пишет класс каждого байта ('.', 'c', 's').
 * Возвращает состояние после строки. */
int hl_line(str s, int in, char *cls);
/* hl_relex - перелексивает строки first..last и дальше, пока состояние не совпадёт с кэшем.
 * Возвращает число перелексированных строк. */
size_t hl_relex(hl *h, size_t first, size_t last);
#endif
```

```c
/* hl_lex.c */
#include "hl.h"
#include <string.h>

static void paint(char *cls, size_t i, size_t n, char c) { if (cls) memset(cls + i, c, n); }

int hl_line(str s, int in, char *cls) {
    size_t i = 0;
    while (i < s.len) {
        bool two = i + 1 < s.len;
        if (in == HL_COMMENT) {
            bool end = two && s.ptr[i] == '*' && s.ptr[i + 1] == '/';
            paint(cls, i, end ? 2 : 1, 'c');
            i += end ? 2 : 1;
            if (end) in = HL_CODE;
        } else if (two && s.ptr[i] == '/' && s.ptr[i + 1] == '*') {
            paint(cls, i, 2, 'c'); i += 2; in = HL_COMMENT;
        } else if (two && s.ptr[i] == '/' && s.ptr[i + 1] == '/') {
            paint(cls, i, s.len - i, 'c'); break;
        } else if (s.ptr[i] == '"') {
            size_t j = i + 1;
            while (j < s.len && s.ptr[j] != '"') j += s.ptr[j] == '\\' ? 2 : 1;
            j = j < s.len ? j + 1 : s.len; /* строка без закрытия кончается с концом строки */
            paint(cls, i, j - i, 's'); i = j;
        } else paint(cls, i++, 1, '.');
    }
    return in;
}
```

```c
/* hl.c */
#include "hl.h"

size_t hl_relex(hl *h, size_t first, size_t last) {
    size_t cnt = 0;
    for (size_t i = first; i < h->n; i++) {
        int out = hl_line(h->line[i], i ? h->end[i - 1] : HL_CODE, nullptr);
        bool same = out == h->end[i];
        h->end[i] = (unsigned char)out;
        cnt++;
        if (i >= last && same) break; /* дальше состояние прежнее */
    }
    return cnt;
}
```

```c
/* main.c */
#include "hl.h"
#include <stdio.h>

#define S(x) {x, sizeof(x) - 1}

int main(void) {
    str src[] = {S("a();"), S("b();"), S("c();"), S("d(); */ e();"), S("f();"), S("g();")};
    unsigned char end[6] = {};
    hl h = {src, 6, end};
    char cls[16];
    printf("%zu\n", hl_relex(&h, 0, 5)); /* вывод: 6 */
    src[0] = (str)S("a(); /*"); /* открыли комментарий в первой строке */
    printf("%zu\n", hl_relex(&h, 0, 0)); /* вывод: 4 */
    hl_line(src[1], end[0], cls);
    printf("%.*s\n", (int)src[1].len, cls); /* вывод: cccc */
    src[0] = (str)S("a();"); /* убрали */
    printf("%zu\n", hl_relex(&h, 0, 0)); /* вывод: 4 */
    src[5] = (str)S("g(); h();");
    printf("%zu\n", hl_relex(&h, 5, 5)); /* вывод: 1 */
    return 0;
}
```

Открытие комментария в первой строке перелексило строки 0-3 из шести: в строке 3 комментарий закрывается, состояние после неё снова `HL_CODE`, как в кэше, и строки 4-5 не тронуты. Закрытие того же комментария потребовало те же четыре строки, а правка последней строки - одну. Если бы закрывающего `*/` не было, перелексирование дошло бы до конца файла: это честная цена, и ради неё лексер должен быть быстрым.

#### Подводные камни

- Состояние должно содержать всё, что влияет на следующую строку: многострочные строковые литералы, вложенные комментарии (счётчик глубины), встроенные языки (стек). Сравнивай состояние целиком, а не одно поле.
- Правка, меняющая число строк, сдвигает массив `end` вместе с массивом строк: вызывающий код вставляет и убирает элементы `end` до `hl_relex`, затем передаёт диапазон строк в новой нумерации.
- Первая правка в недокэшированной области (файл открыт, подсвечен только экран): считай состояние от ближайшей строки с известным состоянием, а неизвестные строки лексируй лениво при прокрутке.
- Лексер сам не различает типы, функции и переменные: для семантической подсветки нужен разбор грамматики (см. «Что ещё стоит знать»).
- Очень длинные строки (минифицированные файлы) лексируй с лимитом длины: иначе одна строка блокирует интерфейс.

## История правок

### Склеивай набор в группы и оформляй связанные правки транзакцией

Undo по одному символу не нужен пользователю, а undo всей сессии слишком грубо, поэтому записи журнала объединяют в группы: undo снимает группу целиком. Группу образуют подряд набранные символы без разрывов или явная транзакция, которой команда оборачивает несколько связанных правок (замену во всех местах, форматирование, правку во всех курсорах).

Запись хранит позицию, удалённый и вставленный текст: этого достаточно, чтобы выполнить её в обратную сторону. Склейка набора применяется, когда новая правка - вставка одного не-пробельного символа ровно за предыдущей вставкой; пробел начинает новую группу, поэтому undo откатывает по слову. Граница по времени (пауза в наборе) добавляется поверх, но таймер делает поведение недетерминированным, поэтому в тестах время подставляют ([тесты](07-testing-and-debugging.md)). Общие команды, их выполнение и стек отмены на уровне интерфейса описаны в [архитектуре интерфейса](15-ui-architecture.md); здесь только то, что специфично для текста.

```c
/* doc.h */
#ifndef DOC_H
#define DOC_H
#include <stddef.h>

typedef struct { size_t pos; char del[8], ins[8]; unsigned grp; } rec; /* тексты фиксированы для краткости */
typedef struct { char text[64]; size_t len; rec h[32]; size_t n; unsigned grp; bool open; } doc;

/* doc_edit - заменяет nd байт с pos на строку ins, записывает правку в журнал. */
void doc_edit(doc *d, size_t pos, size_t nd, const char *ins);
void doc_begin(doc *d); /* начало транзакции: все правки до doc_end образуют одну группу */
void doc_end(doc *d);
void doc_undo(doc *d); /* откатывает последнюю группу */
#endif
```

```c
/* doc.c */
#include "doc.h"
#include <stdlib.h>
#include <string.h>

static void splice(doc *d, size_t pos, size_t nd, const char *s, size_t ni) {
    if (d->len - nd + ni > sizeof d->text) abort();
    memmove(d->text + pos + ni, d->text + pos + nd, d->len - pos - nd);
    memcpy(d->text + pos, s, ni);
    d->len = d->len - nd + ni;
}

void doc_edit(doc *d, size_t pos, size_t nd, const char *ins) {
    size_t ni = strlen(ins);
    if (d->n == 32 || nd >= 8 || ni >= 8) abort();
    const rec *p = d->n ? &d->h[d->n - 1] : nullptr;
    bool typing = !d->open && p && p->ins[0] && !p->del[0] && nd == 0 && ni == 1 && ins[0] != ' '
                  && pos == p->pos + strlen(p->ins);
    rec *r = &d->h[d->n++];
    *r = (rec){.pos = pos, .grp = typing ? d->grp : d->open ? d->grp : ++d->grp};
    memcpy(r->del, d->text + pos, nd);
    memcpy(r->ins, ins, ni);
    splice(d, pos, nd, ins, ni);
}

void doc_begin(doc *d) { d->grp++; d->open = true; }
void doc_end(doc *d) { d->grp++; d->open = false; }

void doc_undo(doc *d) {
    unsigned g = d->n ? d->h[d->n - 1].grp : 0;
    while (d->n && d->h[d->n - 1].grp == g) {
        const rec *r = &d->h[--d->n];
        splice(d, r->pos, strlen(r->ins), r->del, strlen(r->del));
    }
}
```

```c
/* main.c */
#include "doc.h"
#include <stdio.h>

static void type(doc *d, const char *s) {
    for (; *s; s++) doc_edit(d, d->len, 0, (char[]){*s, 0});
}

static void show(const doc *d) { printf("[%.*s]\n", (int)d->len, d->text); }

int main(void) {
    doc d = {};
    type(&d, "hello");
    type(&d, " world");
    doc_undo(&d);
    show(&d); /* вывод: [hello] */
    doc_begin(&d);
    doc_edit(&d, 1, 1, "E");
    doc_edit(&d, 3, 1, "L");
    doc_end(&d);
    show(&d); /* вывод: [hElLo] */
    doc_undo(&d);
    show(&d); /* вывод: [hello] */
    doc_undo(&d);
    show(&d); /* вывод: [] */
    return 0;
}
```

Одиннадцать нажатий дали две группы, «hello» и « world», а две замены в транзакции откатились одним вызовом. Откат записей идёт от последней к первой, и только так позиции в записях остаются верными.

#### Подводные камни

- Запись должна хранить и положение курсора до правки и после: undo возвращает курсор туда, где пользователь был, а не в место последней правки.
- Склейка только соседних правок: после перемещения курсора, сохранения файла или паузы группу закрывают, иначе undo перепрыгнет через осознанное действие.
- Новая правка после undo обрезает стек повтора; если нужна вся история, строят дерево версий, а не стек.
- Запись на символ расходует память: для длинного набора склеивай вставки в одну запись (расширяй `ins`), а не только объединяй в группу.
- Undo и метки: откат - такая же правка, его дельта проходит через пересчёт меток (см. раздел про метки), иначе курсоры и диагностики разойдутся с текстом.
- Признак «документ изменён» связывай с номером группы на момент сохранения, а не с флагом: после undo до сохранённого состояния документ снова чистый.

## Структурированные документы

### Разбирай подмножество markdown блоками, затем инлайном, и перепарсивай затронутое

Блоки не зависят от контекста до них, а инлайн живёт внутри одной строки блока, поэтому формат разбирают в два прохода: сначала строки складываются в блоки, затем содержимое блока разбирается на инлайн-разметку. Отрисовка идёт из дерева блоков, а после правки заново разбирают только затронутые блоки. Реализуется подмножество формата, а не весь markdown: заголовки `#`, абзацы, списки `- `, огороженные блоки кода и инлайн `*выделение*` и `` `код` ``; вложенность, ссылки, таблицы и нумерованные списки опущены, а полный формат содержит много краевых правил.

Блок начинается с первой непустой строки. Огороженный код тянется до закрывающей ограды (без неё - до конца документа), заголовок занимает одну строку, список собирает подряд идущие пункты, абзац продолжается до пустой строки или начала другого блока. Поскольку блок разбирается только по строкам от своего начала, разбор самосинхронизируется: если новый блок начался там же, где начинался старый, оставшийся хвост дерева остался прежним. Это тот же приём, что и в подсветке.

```c
/* md.h */
#ifndef MD_H
#define MD_H
#include <stddef.h>

typedef struct { const char *ptr; size_t len; } str;
typedef enum { MD_HEAD, MD_PARA, MD_LIST, MD_CODE } md_kind;
/* arg: уровень заголовка, для кода 1 если ограда закрыта */
typedef struct { md_kind kind; size_t first, count; int arg; } md_block;
typedef struct { md_block b[32]; size_t n; } md_tree; /* ёмкость фиксирована для краткости */

md_block md_block_at(const str *ln, size_t nl, size_t i); /* блок, начинающийся с непустой строки i */
void md_parse(md_tree *t, const str *ln, size_t nl);
/* md_reparse - после правки одной строки line (число строк не менялось) возвращает число заново разобранных блоков. */
size_t md_reparse(md_tree *t, const str *ln, size_t nl, size_t line);
void md_render(const md_tree *t, const str *ln); /* HTML в stdout */
void md_inline(str s);  /* инлайн-разметка строки */
void md_escape(str s);  /* экранирование без разметки */
#endif
```

```c
/* md.c */
#include "md.h"
#include <stdlib.h>
#include <string.h>

static bool starts(str s, const char *p) { size_t n = strlen(p); return s.len >= n && !memcmp(s.ptr, p, n); }
static bool opens(str s) { return starts(s, "#") || starts(s, "```") || starts(s, "- "); }

md_block md_block_at(const str *ln, size_t nl, size_t i) {
    md_block b = {MD_PARA, i, 1, 0};
    if (starts(ln[i], "```")) {
        bool closed = false;
        b.kind = MD_CODE;
        while (i + b.count < nl && !closed) closed = starts(ln[i + b.count++], "```");
        b.arg = closed;
    } else if (starts(ln[i], "#")) {
        b.kind = MD_HEAD;
        while (b.arg < 6 && (size_t)b.arg < ln[i].len && ln[i].ptr[b.arg] == '#') b.arg++;
    } else if (starts(ln[i], "- ")) {
        b.kind = MD_LIST;
        while (i + b.count < nl && starts(ln[i + b.count], "- ")) b.count++;
    } else {
        while (i + b.count < nl && ln[i + b.count].len && !opens(ln[i + b.count])) b.count++;
    }
    return b;
}

void md_parse(md_tree *t, const str *ln, size_t nl) {
    t->n = 0;
    for (size_t i = 0; i < nl; i++) {
        if (!ln[i].len) continue;
        if (t->n == 32) abort();
        t->b[t->n] = md_block_at(ln, nl, i);
        i += t->b[t->n++].count - 1;
    }
}
```

```c
/* md_reparse.c */
#include "md.h"
#include <stdlib.h>
#include <string.h>

size_t md_reparse(md_tree *t, const str *ln, size_t nl, size_t line) {
    size_t k = 0, cnt = 0;
    if (!t->n) { md_parse(t, ln, nl); return t->n; }
    while (k + 1 < t->n && t->b[k + 1].first <= line) k++;
    size_t s = k ? k - 1 : 0; /* предыдущий блок мог слиться с правленым */
    md_tree nt = {.n = s};
    memcpy(nt.b, t->b, s * sizeof t->b[0]);
    for (size_t i = t->b[s].first; i < nl;) {
        if (!ln[i].len) { i++; continue; }
        size_t j = 0;
        while (j < t->n && !(i > line && t->b[j].first == i)) j++;
        if (j < t->n) { /* совпало с началом старого блока: хвост прежний */
            if (nt.n + (t->n - j) > 32) abort();
            memcpy(nt.b + nt.n, t->b + j, (t->n - j) * sizeof t->b[0]);
            nt.n += t->n - j;
            break;
        }
        if (nt.n == 32) abort();
        nt.b[nt.n] = md_block_at(ln, nl, i);
        i += nt.b[nt.n++].count;
        cnt++;
    }
    *t = nt;
    return cnt;
}
```

```c
/* inline.c */
#include "md.h"
#include <stdio.h>
#include <string.h>

void md_escape(str s) {
    for (size_t i = 0; i < s.len; i++) {
        char c = s.ptr[i];
        if (c == '<') fputs("&lt;", stdout);
        else if (c == '>') fputs("&gt;", stdout);
        else if (c == '&') fputs("&amp;", stdout);
        else putchar(c);
    }
}

/* маркер без пары остаётся буквальным; внутри `кода` разметки нет */
void md_inline(str s) {
    bool code = false, em = false;
    for (size_t i = 0; i < s.len; i++) {
        char c = s.ptr[i];
        bool pair = memchr(s.ptr + i + 1, c, s.len - i - 1) != nullptr;
        if (c == '`' && (code || pair)) fputs((code = !code) ? "<code>" : "</code>", stdout);
        else if (c == '*' && !code && (em || pair)) fputs((em = !em) ? "<em>" : "</em>", stdout);
        else md_escape((str){&s.ptr[i], 1});
    }
}
```

```c
/* render.c */
#include "md.h"
#include <stdio.h>
#include <string.h>

static str tail(str s, size_t n) {
    if (n > s.len) n = s.len;
    return (str){s.ptr + n, s.len - n};
}

void md_render(const md_tree *t, const str *ln) {
    for (size_t k = 0; k < t->n; k++) {
        md_block b = t->b[k];
        const str *l = ln + b.first;
        switch (b.kind) {
        case MD_HEAD:
            printf("<h%d>", b.arg); md_inline(tail(l[0], (size_t)b.arg + 1)); printf("</h%d>\n", b.arg);
            break;
        case MD_PARA:
            fputs("<p>", stdout);
            for (size_t i = 0; i < b.count; i++) { if (i) putchar(' '); md_inline(l[i]); }
            puts("</p>");
            break;
        case MD_LIST:
            puts("<ul>");
            for (size_t i = 0; i < b.count; i++) { fputs("<li>", stdout); md_inline(tail(l[i], 2)); puts("</li>"); }
            puts("</ul>");
            break;
        case MD_CODE:
            fputs("<pre>", stdout);
            for (size_t i = 1; i + (size_t)b.arg < b.count; i++) { md_escape(l[i]); putchar('\n'); }
            puts("</pre>");
            break;
        }
    }
}
```

```c
/* main.c */
#include "md.h"
#include <stdio.h>

#define S(x) {x, sizeof(x) - 1}

int main(void) {
    str ln[] = {S("# Title"), S(""), S("Some *text* and `a<b`."), S(""), S("- one"), S("- two"), S(""), S("end")};
    md_tree t;
    md_parse(&t, ln, 8);
    md_render(&t, ln);
    /* вывод: <h1>Title</h1> */
    /* вывод: <p>Some <em>text</em> and <code>a&lt;b</code>.</p> */
    /* вывод: <ul> */
    /* вывод: <li>one</li> */
    /* вывод: <li>two</li> */
    /* вывод: </ul> */
    /* вывод: <p>end</p> */
    ln[2] = (str)S("Some `code`");
    printf("%zu\n", md_reparse(&t, ln, 8, 2)); /* вывод: 2 */
    ln[4] = (str)S("```"); /* открыли ограду: остаток документа стал кодом */
    printf("%zu ", md_reparse(&t, ln, 8, 4));
    printf("%zu\n", t.n); /* вывод: 2 3 */
    return 0;
}
```

Правка строки 2 разобрала заново два блока из четырёх: правленый абзац и предыдущий заголовок, а на начале списка разбор совпал со старым деревом и остановился. Открытая ограда в строке 4 превратила все строки до конца документа в один блок кода, и дерево из четырёх блоков стало деревом из трёх: заново разобраны абзац и новый блок кода, а совпадения со старым деревом не нашлось, так что разбор шёл до конца. Правка рядом с оградой, стоящей выше по тексту, меняет блоки до самого её закрытия, и это следует из формата, а не из реализации.

#### Подводные камни

- Если правка меняет число строк, начала старых блоков после неё сдвигаются на дельту: перед сравнением на совпадение прибавляй её к `first` старых блоков, иначе синхронизация не найдёт совпадения или найдёт ложное.
- Инлайн разбирай в момент отрисовки блока, а не при разборе блоков: блок кода его не содержит, а результат нужен только видимым блокам.
- Маркеры без пары (`*`, одинокая обратная кавычка) остаются буквальным текстом: разметка не должна «съедать» остаток документа, пока пользователь не допечатал закрывающий символ.
- Экранируй `<`, `>` и `&` при выводе в HTML для всего текста, включая код, иначе содержимое документа превращается в разметку страницы ([безопасность](12-security.md)).
- Подмножество растёт неравномерно: вложенные списки, ссылки и таблицы требуют разбора с учётом отступов и контекста, и там ради корректности по всем краевым случаям берут готовый разборщик.

## Коротко

### Виртуализируй область просмотра и кэшируй раскладку строк

Документ в миллион строк отрисовывается за то же время, что и в сто: рисуют только строки от `top` до `top + высота`. Первую видимую строку находят бинарным поиском по индексу начал строк, а высоту прокрутки берут из числа строк или из накопленной суммы высот, если строки разной высоты. Всё, что стоит дорого (раскладка строки, подсветка, ширина), считают для видимых строк и хранят в кэше, ключом которого служат номер строки, ширина области и версия документа; правка сбрасывает записи затронутых строк, а не всего кэша.

### Переноси строки мягко, не меняя текст

Мягкий перенос - отображение: текст хранится с прежними переводами строк, а раскладка делит строку на визуальные ряды по ширине области. Ряд заканчивается на последнем пробеле, поместившемся в ширину (или на границе символа, если пробела нет), ширину считают столбцами отображения, как в разделе про позиции. Визуальные ряды кэшируются на строку, а движение курсора вверх и вниз использует запомненный столбец, который сбрасывается при горизонтальном движении.

## Что ещё стоит знать

Для полноценного разбора есть готовые библиотеки: tree-sitter строит инкрементальные деревья разбора для многих грамматик, cmark и md4c разбирают markdown целиком; поиск и регулярные выражения над буфером требуют своей реализации или библиотеки и учитывают границы UTF-8; шейпинг текста (лигатуры, сложные письменности, HarfBuzz) и двунаправленный текст (порядок символов в строке с письмом справа налево) выходят за простое «один символ - один столбец»; подсказки и диагностики от внешних серверов описывает протокол LSP, а одновременное редактирование несколькими пользователями строится на операционных преобразованиях или CRDT; сохранять файл безопасно нужно атомарно через временный файл и `rename` ([системный ввод-вывод](11-system-io.md)).
