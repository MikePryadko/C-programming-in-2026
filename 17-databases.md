# Работа с базами данных

## PostgreSQL и слой доступа

### Не ищи ORM: пиши тонкий слой из SQL-констант, репозитория и отображения строки

Популярного ORM для C нет: отображение таблиц на объекты держится на рефлексии и динамических типах, которых в C нет, а SQL уже и есть язык описания данных, так что промежуточный язык ничего не добавляет. Вместо ORM строят три плоских уровня: SQL лежит константами в одном файле, функции-репозитории (`user_find`, `user_save`) прячут запрос и параметры, отображение строки результата в `struct` описано таблицей полей. Схему базы версионируют вне C: файлами SQL и инструментом миграций (sqitch, dbmate, Flyway); встроенный SQL прямо в исходниках умеет препроцессор ecpg из состава PostgreSQL, но чаще берут явные запросы.

Дальше в разделе: как отправить запрос безопасно и не потерять память (libpq), как собрать строку в `struct`, как повторять транзакции при конфликтах и как не блокировать поток на сети. Затем Redis, подмена базы в тестах и коротко SQLite, массовая загрузка и уведомления. Первый соблазн самодельного слоя - собирать SQL склейкой строк, и он же главная уязвимость:

```c
/* Ошибка: значение попадает в текст запроса, возможна SQL-инъекция */
#include <libpq-fe.h>
#include <stdio.h>

PGresult *user_by_name(PGconn *conn, const char *name) {
    char sql[256];
    snprintf(sql, sizeof sql, "SELECT id FROM users WHERE name = '%s'", name);
    return PQexec(conn, sql);
}
```

Имя `' OR '1'='1` делает условие истинным для всех строк, а имя с `;` добавляет вторую команду. Репозиторий же показывает наружу только типизированные функции и отдаёт значения серверу параметрами; контракт одного репозитория:

```c
/* user_repo.h */
#ifndef USER_REPO_H
#define USER_REPO_H
#include "pg.h"

typedef struct { int32_t id; const char *name; const char *email; int32_t age; } user;

/* user_find - читает пользователя; строки копируются в арену a и живут вместе с ней.
 * Возврат: 0, -ENOENT (строки нет), -EINVAL, -ERANGE или -ENOENT (нет колонки) из pg_row, либо код pg_query. */
int user_find(pg *db, int32_t id, user *out, arena *a);

/* user_save - вставляет или обновляет по id; email == nullptr пишет SQL NULL.
 * Возврат: 0 или код pg_query. */
int user_save(pg *db, const user *u);
#endif
```

Заголовок ссылается на `pg.h` и `arena`: их разбирают следующие практики. Реализация держит SQL константами рядом с функциями (практика про отображение строки).

#### Подводные камни

- Динамические фильтры. Условия, которых заранее не знаешь, строят из константных фрагментов запроса (`" AND age > $3"`), а значения всё равно идут параметрами; имена колонок для сортировки выбирают из белого списка.
- Один и тот же SQL в нескольких местах. Копии расходятся со схемой после миграции; текст запроса живёт в одном файле у репозитория.
- `SELECT *`. Миграция меняет состав и порядок колонок; перечисляй колонки явно и отображай по имени, а не по номеру.

### Передавай значения параметрами и освобождай всё, что вернул libpq

Параметр `$n` уходит на сервер отдельно от текста запроса, поэтому значение не может стать частью SQL; экранирование вручную эту защиту не заменяет. Помощник `pg_query` оборачивает `PQexecParams` и сводит исход к схеме книги: объект или `NULL` с кодом в `err`, а SQLSTATE сервера остаётся в `db->sqlstate`. Внедрение в запросы закрывается именно так ([безопасность](19-security.md)); строка подключения `""` заставляет libpq взять параметры из `PGHOST`, `PGDATABASE`, `PGUSER` и файла паролей, поэтому пароль не лежит ни в коде, ни в argv ([аргументы и конфигурация](15-cli-and-config.md)).

Схема стенда и общий заголовок с определениями арены ([память](04-memory-ownership.md)) и дескриптора поля ([сериализация](14-serialization.md)):

```text
-- file: schema.sql
CREATE TABLE users (
    id    int4 PRIMARY KEY,
    name  text NOT NULL,
    email text,
    age   int4 NOT NULL
);
CREATE TABLE account (id int4 PRIMARY KEY, balance int4 NOT NULL CHECK (balance >= 0));
CREATE TABLE counter (n int4 NOT NULL);
INSERT INTO counter VALUES (0);
```

```c
/* kit.h */
#ifndef KIT_H
#define KIT_H
#include <assert.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

/* Упрощённая копия арены из главы про владение и дескриптора из главы про сериализацию,
 * чтобы пример собирался сам. */
typedef struct { char *beg, *end; } arena;

static inline arena arena_init(void *buf, size_t cap) {
    char *p = buf;
    return (arena){p, p + cap};
}

static inline void *arena_alloc(arena *a, size_t size, size_t align, size_t count) {
    assert(align && (align & (align - 1)) == 0);
    size_t pad = -(uintptr_t)a->beg & (align - 1);
    size_t room = (size_t)(a->end - a->beg);
    if (pad > room || (count && size > (room - pad) / count)) abort();
    char *p = a->beg + pad;
    a->beg = p + size * count;
    return memset(p, 0, size * count);
}
#define NEW(a, T, n) ((T *)arena_alloc((a), sizeof(T), alignof(T), (n)))

typedef enum { F_I32, F_U32, F_U8, F_STR } ftype;
typedef struct { const char *name; ftype type; size_t off; } fdesc;
#define TYPE_OF(T, f) _Generic(((T *)0)->f, int32_t: F_I32, uint32_t: F_U32, uint8_t: F_U8, const char *: F_STR)
#define FIELD(T, f) {#f, TYPE_OF(T, f), offsetof(T, f)}
#endif
```

Тонкая обёртка над соединением и запросом:

```c
/* pg.h */
#ifndef PG_H
#define PG_H
#include <libpq-fe.h>
#include "kit.h"

typedef struct { PGconn *conn; char sqlstate[6]; char msg[128]; } pg;

/* pg_open - подключается; conninfo "" берёт параметры из окружения и файла паролей.
 * Возврат: 0 или -ECONNREFUSED; при ошибке db->conn == nullptr, причина в db->msg. */
int pg_open(pg *db, const char *conninfo);
void pg_close(pg *db);

/* pg_query - одна команда с параметрами $1..$n; vals: n строк, nullptr значит SQL NULL.
 * Возврат: результат (освободить PQclear) или nullptr; *err: 0, -ECONNRESET (связь потеряна),
 * -EIO (ошибка сервера, SQLSTATE в db->sqlstate), -ENOMEM. */
PGresult *pg_query(pg *db, const char *sql, int n, const char *const *vals, int *err);
PGresult *pg_check(pg *db, PGresult *r, int *err);  /* разбирает готовый результат, забирает владение */
int pg_exec(pg *db, const char *sql, int n, const char *const *vals);
PGresult *pg_query_timeout(pg *db, const char *sql, int n, const char *const *vals, int ms, int *err);
int pg_count(pg *db, const char *table, long *out);
int pg_row(const PGresult *r, int row, const fdesc *d, size_t nd, void *obj, arena *a);
#endif
```

```c
/* pg.c */
#include "pg.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>

int pg_open(pg *db, const char *conninfo) {
    memset(db, 0, sizeof *db);
    db->conn = PQconnectdb(conninfo);
    if (db->conn && PQstatus(db->conn) == CONNECTION_OK) return 0;
    snprintf(db->msg, sizeof db->msg, "%s", PQerrorMessage(db->conn));
    PQfinish(db->conn);
    db->conn = nullptr;
    return -ECONNREFUSED;
}

void pg_close(pg *db) { PQfinish(db->conn); db->conn = nullptr; }

PGresult *pg_check(pg *db, PGresult *r, int *err) {
    ExecStatusType st = r ? PQresultStatus(r) : PGRES_FATAL_ERROR;
    db->sqlstate[0] = '\0';
    *err = 0;
    if (st == PGRES_TUPLES_OK || st == PGRES_COMMAND_OK) return r;
    const char *s = r ? PQresultErrorField(r, PG_DIAG_SQLSTATE) : nullptr;
    if (s) snprintf(db->sqlstate, sizeof db->sqlstate, "%s", s);
    *err = PQstatus(db->conn) == CONNECTION_BAD ? -ECONNRESET : r ? -EIO : -ENOMEM;
    PQclear(r);
    return nullptr;
}

PGresult *pg_query(pg *db, const char *sql, int n, const char *const *vals, int *err) {
    return pg_check(db, PQexecParams(db->conn, sql, n, NULL, vals, NULL, NULL, 0), err);
}

int pg_exec(pg *db, const char *sql, int n, const char *const *vals) {
    int err;
    PQclear(pg_query(db, sql, n, vals, &err));
    return err;
}
```

Демонстрация: опасная строка сохраняется как обычное значение, SQL NULL передаётся как `NULL` в массиве, а пустой результат отличается от `NULL` через `PQgetisnull`.

```c
/* query_demo.c */
#include "pg.h"
#include <stdio.h>

int main(void) {
    pg db;
    if (pg_open(&db, "") < 0) { fprintf(stderr, "%s", db.msg); return 1; }
    const char *vals[] = {"1", "x'); DROP TABLE users; --", nullptr, "30"};
    int err = pg_exec(&db, "INSERT INTO users (id, name, email, age) VALUES ($1::int4, $2, $3, $4::int4)", 4, vals);
    if (err < 0) return 1;
    PGresult *r = pg_query(&db, "SELECT name, email FROM users WHERE id = $1::int4", 1, vals, &err);
    if (!r) return 1;
    printf("%d\n", PQntuples(r)); /* вывод: 1 */
    printf("%s\n", PQgetvalue(r, 0, 0)); /* вывод: x'); DROP TABLE users; -- */
    printf("%d [%s]\n", PQgetisnull(r, 0, 1), PQgetvalue(r, 0, 1)); /* вывод: 1 [] */
    PQclear(r);
    pg_close(&db);
    return 0;
}
```

Идентификатор (имя таблицы, колонки) параметром не передать: `FROM $1` не разбирается сервером. Если имя приходит извне, его экранирует `PQescapeIdentifier` (результат выделен libpq и освобождается `PQfreemem`, а не `free`), а лучше выбирают из белого списка.

```c
/* pg_count.c */
#include "pg.h"
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int pg_count(pg *db, const char *table, long *out) {
    char *q = PQescapeIdentifier(db->conn, table, strlen(table));
    if (!q) return -EINVAL;
    char sql[128];
    int n = snprintf(sql, sizeof sql, "SELECT count(*) FROM %s", q);
    PQfreemem(q);
    if (n < 0 || (size_t)n >= sizeof sql) return -ENAMETOOLONG;
    int err;
    PGresult *r = pg_query(db, sql, 0, NULL, &err);
    if (!r) return err;
    *out = strtol(PQgetvalue(r, 0, 0), NULL, 10);
    PQclear(r);
    return 0;
}
```

#### Подводные камни

- `PQgetvalue` возвращает указатель внутрь `PGresult`. После `PQclear` он висит; значение копируют до очистки (отображение в `struct` ниже делает это в арену).
- Для SQL NULL `PQgetvalue` возвращает пустую строку: отличай `NULL` от `''` только через `PQgetisnull`.
- Тип параметра сервер выводит из контекста (`WHERE id = $1` берёт тип колонки); без контекста (`$1 IS NULL`) он отвечает ошибкой, поэтому пиши приведение `$1::int4`.
- В `PQexecParams` допустима одна команда: `SELECT 1; SELECT 2` с параметрами отвергается; несколько команд шлют несколькими вызовами.
- `PQconnectdb` возвращает объект и при неудаче: без `PQfinish` он утекает, а текст ошибки из `PQerrorMessage` принадлежит соединению и пропадает вместе с ним, поэтому `pg_open` копирует его до `PQfinish`.
- Источники пароля для libpq: файл паролей (`~/.pgpass`, права 0600), переменные окружения или файл служб.
- Результат по умолчанию текстовый: числа, даты и логические значения приходят строками и разбираются явно (следующая практика).

### Отображай строку результата в struct по таблице дескрипторов

Один цикл по таблице полей заменяет десятки ручных `PQgetvalue` и собирает ошибки типа и имени колонки в одном месте. Механизм дескриптора (имя, тип, смещение) описан в [главе про сериализацию](14-serialization.md); здесь он применён к строкам: колонка находится по имени, SQL NULL в строковое поле даёт `NULL`, в числовое поле - ошибку, а строки копируются в арену, чтобы пережить `PQclear`.

```c
/* map.h */
#ifndef MAP_H
#define MAP_H
#include "kit.h"

/* map_set - кладёт текстовое значение s в поле d объекта obj; s == nullptr означает SQL NULL.
 * Строки копируются в арену a. Возврат: 0, -EINVAL (не число или NULL в числе), -ERANGE. */
int map_set(const fdesc *d, void *obj, const char *s, arena *a);
#endif
```

```c
/* map.c */
#include "map.h"
#include <errno.h>
#include <stdlib.h>

static int num(const char *s, long long lo, long long hi, long long *out) {
    char *end;
    errno = 0;
    long long v = strtoll(s, &end, 10);
    if (end == s || *end) return -EINVAL;
    if (errno == ERANGE || v < lo || v > hi) return -ERANGE;
    *out = v;
    return 0;
}

int map_set(const fdesc *d, void *obj, const char *s, arena *a) {
    char *p = (char *)obj + d->off;
    long long v = 0;
    if (d->type == F_STR) {
        size_t n = s ? strlen(s) + 1 : 0;
        const char *copy = s ? memcpy(arena_alloc(a, 1, 1, n), s, n) : nullptr;
        memcpy(p, &copy, sizeof copy);
        return 0;
    }
    if (!s) return -EINVAL;
    int err = d->type == F_I32 ? num(s, INT32_MIN, INT32_MAX, &v)
            : d->type == F_U32 ? num(s, 0, UINT32_MAX, &v) : num(s, 0, UINT8_MAX, &v);
    if (err) return err;
    int32_t i = (int32_t)v; uint32_t u = (uint32_t)v; uint8_t b = (uint8_t)v;
    if (d->type == F_I32) memcpy(p, &i, sizeof i);
    else if (d->type == F_U32) memcpy(p, &u, sizeof u);
    else memcpy(p, &b, sizeof b);
    return 0;
}
```

```c
/* rows.c */
#include "map.h"
#include "pg.h"
#include <errno.h>

int pg_row(const PGresult *r, int row, const fdesc *d, size_t nd, void *obj, arena *a) {
    for (size_t i = 0; i < nd; i++) {
        int col = PQfnumber(r, d[i].name);
        if (col < 0) return -ENOENT;
        const char *s = PQgetisnull(r, row, col) ? nullptr : PQgetvalue(r, row, col);
        int err = map_set(&d[i], obj, s, a);
        if (err < 0) return err;
    }
    return 0;
}
```

Реализация репозитория: SQL константами, значения параметрами, таблица полей рядом со структурой. Целые параметры форматируются в текст `snprintf` - сервер принимает их через приведение `$n::int4`.

```c
/* user_repo.c */
#include "user_repo.h"
#include <errno.h>
#include <stdio.h>

static const char sql_find[] = "SELECT id, name, email, age FROM users WHERE id = $1::int4";
static const char sql_save[] =
    "INSERT INTO users (id, name, email, age) VALUES ($1::int4, $2, $3, $4::int4) "
    "ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name, email = EXCLUDED.email, age = EXCLUDED.age";
static const fdesc user_desc[] = {FIELD(user, id), FIELD(user, name), FIELD(user, email), FIELD(user, age)};

int user_find(pg *db, int32_t id, user *out, arena *a) {
    char buf[16];
    snprintf(buf, sizeof buf, "%d", id);
    const char *vals[] = {buf};
    int err;
    PGresult *r = pg_query(db, sql_find, 1, vals, &err);
    if (!r) return err;
    err = PQntuples(r) == 0 ? -ENOENT : pg_row(r, 0, user_desc, sizeof user_desc / sizeof *user_desc, out, a);
    PQclear(r);
    return err;
}

int user_save(pg *db, const user *u) {
    char id[16], age[16];
    snprintf(id, sizeof id, "%d", u->id);
    snprintf(age, sizeof age, "%d", u->age);
    const char *vals[] = {id, u->name, u->email, age};
    return pg_exec(db, sql_save, 4, vals);
}
```

Демонстрация: `NULL` и пустая строка различимы, строки живут в арене после `PQclear`, отсутствие строки даёт `-ENOENT`.

```c
/* rows_demo.c */
#include "user_repo.h"
#include <stdio.h>

static void show(pg *db, int32_t id, arena *a) {
    user u = {0};
    int err = user_find(db, id, &u, a);
    if (err < 0) printf("%d: err %d\n", id, err);
    else printf("%d: %s [%s] %d\n", u.id, u.name, u.email ? u.email : "NULL", u.age);
}

int main(void) {
    pg db;
    if (pg_open(&db, "") < 0) return 1;
    user ann = {1, "ann", nullptr, 31}, bob = {2, "bob", "", 40};
    if (user_save(&db, &ann) < 0 || user_save(&db, &bob) < 0) return 1;
    char mem[512];
    arena a = arena_init(mem, sizeof mem);
    show(&db, 1, &a); /* вывод: 1: ann [NULL] 31 */
    show(&db, 2, &a); /* вывод: 2: bob [] 40 */
    show(&db, 3, &a); /* вывод: 3: err -2 */
    pg_close(&db);
    return 0;
}
```

#### Подводные камни

- Имя в таблице дескрипторов должно совпадать с именем колонки в результате; для вычисляемых колонок ставь алиас `AS`. Нет колонки - `-ENOENT` на первой же строке, а не молчаливый ноль.
- Значение вне диапазона поля (колонка `int8` со значением 3000000000 в поле `int32_t`) даёт `-ERANGE`, а не усечение: проверка границ делается при разборе текста.
- Колонку с NULL нельзя отображать в числовое поле `NOT NULL`-типа без решения: либо `COALESCE` в запросе, либо строковое поле и явная обработка `NULL`.
- Даты, `numeric` и логические значения приходят текстом и в `struct` попадают строкой (`const char *`); разбирай их отдельной функцией, а не расширяй таблицу без нужды.
- Структуру перед заполнением обнуляй: колонки, не попавшие в таблицу, остаются нулями.

### Повторяй транзакцию целиком, когда сервер отменил её из-за конфликта

При конкурентной работе сервер завершает ошибкой одну из транзакций: SQLSTATE `40001` (сбой сериализации) или `40P01` (взаимная блокировка), и единственный правильный ответ - откатить и повторить всю транзакцию с начала, а не отдельный запрос. Помощник `tx_run` явно отправляет `BEGIN`, вызывает функцию, делает `COMMIT`, откатывает при любой ошибке и повторяет только при этих двух кодах, не больше `max_tries` раз: без предела при постоянном конфликте цикл не закончится.

```c
/* tx.h */
#ifndef TX_H
#define TX_H
#include "pg.h"

typedef int (*tx_fn)(pg *db, void *ctx);

/* tx_run - выполняет fn в транзакции SERIALIZABLE; при SQLSTATE 40001 и 40P01 откатывает и
 * повторяет всю транзакцию (fn вызывается заново) не более max_tries раз.
 * fn повторяема: побочные эффекты вне базы делай после успешного возврата.
 * Возврат: 0; -EAGAIN (попытки кончились); иначе код ошибки fn или COMMIT. -ECONNRESET:
 * связь потеряна, исход неизвестен. */
int tx_run(pg *db, int max_tries, tx_fn fn, void *ctx);

/* pg_reconnect - новое соединение с теми же параметрами; подготовленные запросы и SET теряются. */
int pg_reconnect(pg *db);
#endif
```

```c
/* tx.c */
#include "tx.h"
#include <errno.h>
#include <string.h>

static bool conflict(const pg *db) {
    return !strcmp(db->sqlstate, "40001") || !strcmp(db->sqlstate, "40P01");
}

int tx_run(pg *db, int max_tries, tx_fn fn, void *ctx) {
    for (int i = 0; i < max_tries; i++) {
        int err = pg_exec(db, "BEGIN ISOLATION LEVEL SERIALIZABLE", 0, nullptr);
        if (err == 0) err = fn(db, ctx);
        if (err == 0) err = pg_exec(db, "COMMIT", 0, nullptr);
        if (err == 0) return 0;
        bool again = conflict(db);
        if (err != -ECONNRESET) pg_exec(db, "ROLLBACK", 0, nullptr);
        if (!again) return err;
    }
    return -EAGAIN;
}

int pg_reconnect(pg *db) {
    PQreset(db->conn);
    return PQstatus(db->conn) == CONNECTION_OK ? 0 : -ECONNRESET;
}
```

Демонстрация воспроизводит конфликт двумя соединениями: между чтением и записью другой клиент меняет ту же строку, первая попытка получает `40001`, вторая проходит. Вставка уже существующего ключа даёт `23505`: это не повторяемый конфликт, а ответ на вопрос приложения (его решают `ON CONFLICT` или явной веткой).

```c
/* tx_demo.c */
#include "tx.h"
#include <stdio.h>

typedef struct { pg *other; int attempts; } ctx;

static int withdraw(pg *db, void *p) {
    ctx *c = p;
    int err;
    PQclear(pg_query(db, "SELECT balance FROM account WHERE id = 1", 0, NULL, &err));
    if (err) return err;
    if (c->attempts++ == 0)  /* другой клиент меняет строку между чтением и записью */
        pg_exec(c->other, "UPDATE account SET balance = balance + 1 WHERE id = 1", 0, nullptr);
    return pg_exec(db, "UPDATE account SET balance = balance - 10 WHERE id = 1", 0, nullptr);
}

int main(void) {
    pg db, other;
    if (pg_open(&db, "") < 0 || pg_open(&other, "") < 0) return 1;
    pg_exec(&db, "INSERT INTO account VALUES (1, 100)", 0, nullptr);
    ctx c = {&other, 0};
    int err = tx_run(&db, 3, withdraw, &c);
    printf("err=%d attempts=%d\n", err, c.attempts); /* вывод: err=0 attempts=2 */
    c.attempts = 0;
    err = tx_run(&db, 1, withdraw, &c);
    printf("err=%d\n", err); /* вывод: err=-11 */
    err = pg_exec(&db, "INSERT INTO account VALUES (1, 5)", 0, nullptr);
    printf("%d %s\n", err, db.sqlstate); /* вывод: -5 23505 */
    pg_close(&db);
    pg_close(&other);
    return 0;
}
```

Обрыв связи вне конфликтов - другая ситуация: сервер сам откатывает открытую транзакцию, а `pg_reconnect` (`PQreset`) открывает новое соединение, на котором подготовленные запросы (`PQprepare`) и параметры сессии (`SET`) уже не существуют. Если связь оборвалась на `COMMIT`, транзакция могла как зафиксироваться, так и нет: повторять её вслепую нельзя, результат проверяют по данным (например, по уникальному ключу операции). Внешние пулеры соединений (pgbouncer) по-разному обращаются с подготовленными запросами в зависимости от режима и версии; полагаться на `PQprepare` за пулером можно только после проверки по его документации.

#### Подводные камни

- Функция в транзакции с побочными эффектами (письмо, запись в другое хранилище) выполнится несколько раз; эффекты делают после успешного `tx_run`.
- После ошибки без `ROLLBACK` соединение остаётся в прерванной транзакции и отвергает все команды до отката; `tx_run` откатывает всегда, кроме потери связи.
- Повтор без паузы под нагрузкой усиливает конфликт: между попытками добавляют небольшую случайную задержку.
- `40001` возможен и на `COMMIT`, а не только на запросе: проверяй код после каждой команды транзакции, как делает `tx_run`.
- Исход оборванного `COMMIT` неизвестен: `-ECONNRESET` из `tx_run` не повторяют автоматически, если операция не идемпотентна.

### Не блокируй поток на сети: ограничивай время запроса и держи пул соединений

Синхронный `PQexecParams` блокирует поток до ответа, и зависший сервер подвешивает вызывающего без срока; поэтому запрос отправляют (`PQsendQueryParams`) и ждут сокет с предельным временем. Простейший предел задаёт сервер: параметр `statement_timeout` (в строке подключения `options='-c statement_timeout=5000'` или командой `SET`) обрывает долгий запрос ошибкой, а клиентская версия ниже нужна там, где ожидание встроено в цикл событий. Сокет соединения (`PQsocket`) кладётся в тот же `poll` ([системный уровень](13-system-io.md)): по `POLLIN` вызывают `PQconsumeInput`, пока `PQisBusy` истинно, ждут дальше, затем забирают результаты `PQgetResult` до `NULL`.

```c
/* pg_async.c */
#define _GNU_SOURCE
#include "pg.h"
#include <errno.h>
#include <poll.h>
#include <string.h>
#include <time.h>

static long now_ms(void) {
    struct timespec t;
    clock_gettime(CLOCK_MONOTONIC, &t);
    return (long)t.tv_sec * 1000 + t.tv_nsec / 1000000;
}

static void cancel(PGconn *conn) {
    char buf[256];
    PGcancel *c = PQgetCancel(conn);
    if (c) { PQcancel(c, buf, sizeof buf); PQfreeCancel(c); }
}

PGresult *pg_query_timeout(pg *db, const char *sql, int n, const char *const *vals, int ms, int *err) {
    long end = now_ms() + ms;
    bool cancelled = false;
    if (!PQsendQueryParams(db->conn, sql, n, NULL, vals, NULL, NULL, 0)) { *err = -ECONNRESET; return nullptr; }
    while (PQconsumeInput(db->conn) && PQisBusy(db->conn)) {
        long left = end - now_ms();
        if (left <= 0 && cancelled) { *err = -ETIMEDOUT; return nullptr; }  /* нет ответа даже на отмену */
        if (left <= 0) { cancel(db->conn); cancelled = true; end = now_ms() + 1000; left = 1000; }
        struct pollfd p = {PQsocket(db->conn), POLLIN, 0};
        poll(&p, 1, (int)left);
    }
    PGresult *r = PQgetResult(db->conn), *rest;
    while ((rest = PQgetResult(db->conn))) PQclear(rest);
    r = pg_check(db, r, err);
    if (cancelled && !r && !strcmp(db->sqlstate, "57014")) *err = -ETIMEDOUT;
    return r;
}
```

По истечении срока посылается отмена: в libpq 16 это `PQcancel` (блокирующая, открывает отдельное соединение); в заголовке 16.x нет семейства `PQcancelCreate`, поэтому версию клиентской библиотеки сверяй по установленному `libpq-fe.h`. Сервер отвечает ошибкой `57014`, помощник превращает её в `-ETIMEDOUT`, а соединение остаётся пригодным. Если сервер не ответил и на отмену (процесс остановлен), помощник ждёт ещё секунду и тоже возвращает `-ETIMEDOUT`, но соединение остаётся занятым неоконченной командой (`PQtransactionStatus` равен `PQTRANS_ACTIVE`): его заменяют `pg_reconnect`.

```c
/* async_demo.c */
#include "pg.h"
#include <errno.h>
#include <stdio.h>

int main(void) {
    pg db;
    if (pg_open(&db, "") < 0) return 1;
    int err;
    const char *sleep5[] = {"5"}, *num[] = {"41"};
    PGresult *r = pg_query_timeout(&db, "SELECT pg_sleep($1::float8)", 1, sleep5, 200, &err);
    printf("%d\n", err == -ETIMEDOUT); /* вывод: 1 */
    PQclear(r);
    r = pg_query_timeout(&db, "SELECT $1::int4 + 1", 1, num, 1000, &err);
    if (!r) return 1;
    printf("%s\n", PQgetvalue(r, 0, 0)); /* вывод: 42 */
    PQclear(r);
    pg_close(&db);
    return 0;
}
```

Для нескольких независимых запросов подряд есть режим конвейера: после `PQenterPipelineMode` запросы отправляются без ожидания ответов, `PQpipelineSync` ставит границу, а результаты читаются по порядку (после каждого запроса идёт `NULL`, после границы - результат `PGRES_PIPELINE_SYNC`); он экономит сетевые задержки там, где запросы не зависят друг от друга. Одно соединение одновременно используется одним потоком ([потоки](12-concurrency.md)); для многопоточной программы соединения кладут в пул, устроенный как очередь свободных соединений под мьютексом и условной переменной: поток берёт соединение, работает и возвращает.

```c
/* pool.h */
#ifndef POOL_H
#define POOL_H
#include "pg.h"
#include <pthread.h>

typedef struct pool { pthread_mutex_t mu; pthread_cond_t cv; int nidle; pg *conn; pg **idle; } pool;

/* pool_create - открывает size соединений. Возврат: пул или nullptr (открытые закрыты). */
pool *pool_create(const char *conninfo, int size);

/* pool_get - блокируется, пока не освободится соединение; оно принадлежит вызывающему до pool_put. */
pg *pool_get(pool *p);

/* pool_put - возвращает соединение: незавершённая транзакция откатывается, оборванное или занятое
 * неоконченной командой соединение открывается заново. */
void pool_put(pool *p, pg *db);
void pool_destroy(pool *p);  /* все соединения должны быть возвращены */
#endif
```

```c
/* pool.c */
#include "pool.h"
#include <stdlib.h>

void pool_destroy(pool *p) {
    for (int i = 0; i < p->nidle; i++) pg_close(p->idle[i]);
    pthread_mutex_destroy(&p->mu);
    pthread_cond_destroy(&p->cv);
    free(p->idle);
    free(p->conn);
    free(p);
}

pool *pool_create(const char *conninfo, int size) {
    pool *p = calloc(1, sizeof *p);
    if (!p) return nullptr;
    p->conn = calloc((size_t)size, sizeof *p->conn);
    p->idle = calloc((size_t)size, sizeof *p->idle);
    pthread_mutex_init(&p->mu, NULL);
    pthread_cond_init(&p->cv, NULL);
    while (p->conn && p->idle && p->nidle < size && pg_open(&p->conn[p->nidle], conninfo) == 0) {
        p->idle[p->nidle] = &p->conn[p->nidle];
        p->nidle++;
    }
    if (p->nidle == size) return p;
    pool_destroy(p);
    return nullptr;
}

pg *pool_get(pool *p) {
    pthread_mutex_lock(&p->mu);
    while (p->nidle == 0) pthread_cond_wait(&p->cv, &p->mu);
    pg *db = p->idle[--p->nidle];
    pthread_mutex_unlock(&p->mu);
    return db;
}
```

Возврат соединения лежит в отдельном файле: он сбрасывает состояние соединения до того, как его увидит следующий поток.

```c
/* pool_put.c */
#include "pool.h"
#include "tx.h"

void pool_put(pool *p, pg *db) {
    PGTransactionStatusType ts = PQtransactionStatus(db->conn);
    if (PQstatus(db->conn) == CONNECTION_BAD || ts == PQTRANS_ACTIVE) pg_reconnect(db);
    else if (ts != PQTRANS_IDLE) pg_exec(db, "ROLLBACK", 0, nullptr);
    pthread_mutex_lock(&p->mu);
    p->idle[p->nidle++] = db;
    pthread_cond_signal(&p->cv);
    pthread_mutex_unlock(&p->mu);
}
```

Четыре потока делят два соединения; итоговое значение счётчика показывает, что все обновления выполнены.

```c
/* pool_demo.c */
#include "pool.h"
#include <pthread.h>
#include <stdio.h>

static void *worker(void *arg) {
    pool *p = arg;
    for (int i = 0; i < 25; i++) {
        pg *db = pool_get(p);
        pg_exec(db, "UPDATE counter SET n = n + 1", 0, nullptr);
        pool_put(p, db);
    }
    return nullptr;
}

int main(void) {
    pool *p = pool_create("", 2);
    if (!p) return 1;
    pthread_t th[4];
    for (int i = 0; i < 4; i++) pthread_create(&th[i], NULL, worker, p);
    for (int i = 0; i < 4; i++) pthread_join(th[i], NULL);
    pg *db = pool_get(p);
    int err;
    PGresult *r = pg_query(db, "SELECT n FROM counter", 0, NULL, &err);
    if (r) printf("%s\n", PQgetvalue(r, 0, 0)); /* вывод: 100 */
    PQclear(r);
    pool_put(p, db);
    pool_destroy(p);
    return 0;
}
```

#### Подводные камни

- Ответ нужно вычитать до конца: пока результаты не забраны `PQgetResult` до `NULL`, следующая команда на соединении отвергается.
- Размер пула ограничен числом соединений сервера (`max_connections`), общим для всех клиентов; пул больше числа активных потоков бесполезен, а при многих процессах нужен внешний пулер.
- Вернувшееся в пул соединение с открытой или прерванной транзакцией отравит следующего владельца: `pool_put` откатывает её и переподключает оборванное соединение.
- Соединение, пока оно у потока, не отдают другому потоку и не используют из обработчика сигнала.
- Таймаут клиента без отмены оставляет запрос работать на сервере: помощник посылает `PQcancel` и вычитывает ответ, прежде чем вернуть соединение. Сама отмена блокирует на сетевом соединении к серверу, поэтому не вызывай её из потока, который обязан отвечать быстро.

## Redis через hiredis

### Проверяй ответ Redis, бери длину из reply и читай хэш в struct по таблице

Каждая команда Redis возвращает `redisReply`, тип которого заранее не известен: строка, число, массив, `NIL` или ошибка сервера, поэтому помощник сводит исход к схеме книги: ответ или `NULL` с кодом в `err`. Двоичные данные передают только по `argv` (`redisCommandArgv` с длинами) или через `%b` в `redisCommand`; обычный `%s` обрывает значение на первом нулевом байте. Ответ `NULL` означает ошибку контекста (разрыв, таймаут): такой контекст непригоден, нужно переподключение; ответ типа `REDIS_REPLY_ERROR` - ошибка сервера, а не связи.

```c
/* kv.h */
#ifndef KV_H
#define KV_H
#include <hiredis/hiredis.h>
#include "kit.h"

typedef struct { redisContext *c; char msg[128]; } kv;

/* kv_open - соединяется по сокету из REDIS_SOCK или с 127.0.0.1:6379; timeout_ms действует
 * на соединение и на каждую команду. Возврат: 0 или -ECONNREFUSED (k->c == nullptr, причина в k->msg). */
int kv_open(kv *k, int timeout_ms);
void kv_close(kv *k);
int kv_reconnect(kv *k);  /* новое соединение на том же контексте; ответы прежнего потока теряются */

/* kv_cmd - команда из argc слов; lens: длины слов (двоичные данные) или nullptr, тогда strlen.
 * Возврат: ответ (освободить freeReplyObject) или nullptr; *err: 0, -ECONNRESET (контекст сломан),
 * -ETIMEDOUT, -EIO (Redis ответил ошибкой, текст в k->msg). */
redisReply *kv_cmd(kv *k, int *err, int argc, const char **argv, const size_t *lens);

/* kv_load - HGETALL ключа в struct по таблице дескрипторов; строки копируются в арену a.
 * Возврат: 0, -ENOENT (ключа нет), -EBADMSG (неожиданный ответ), -EINVAL, -ERANGE или код kv_cmd. */
int kv_load(kv *k, const char *key, const fdesc *d, size_t nd, void *obj, arena *a);
#endif
```

```c
/* kv.c */
#include "kv.h"
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>

int kv_open(kv *k, int timeout_ms) {
    struct timeval tv = {timeout_ms / 1000, (timeout_ms % 1000) * 1000};
    const char *sock = getenv("REDIS_SOCK");
    k->c = sock ? redisConnectUnixWithTimeout(sock, tv) : redisConnectWithTimeout("127.0.0.1", 6379, tv);
    if (k->c && !k->c->err && redisSetTimeout(k->c, tv) == REDIS_OK) return 0;
    snprintf(k->msg, sizeof k->msg, "%s", k->c ? k->c->errstr : "no memory");
    redisFree(k->c);
    k->c = nullptr;
    return -ECONNREFUSED;
}

void kv_close(kv *k) { redisFree(k->c); k->c = nullptr; }

int kv_reconnect(kv *k) { return redisReconnect(k->c) == REDIS_OK ? 0 : -ECONNREFUSED; }

redisReply *kv_cmd(kv *k, int *err, int argc, const char **argv, const size_t *lens) {
    redisReply *r = redisCommandArgv(k->c, argc, argv, lens);
    *err = 0;
    if (!r) {
        bool slow = k->c->err == REDIS_ERR_TIMEOUT || (k->c->err == REDIS_ERR_IO && errno == EAGAIN);
        snprintf(k->msg, sizeof k->msg, "%s", k->c->errstr);
        *err = slow ? -ETIMEDOUT : -ECONNRESET;
    } else if (r->type == REDIS_REPLY_ERROR) {
        snprintf(k->msg, sizeof k->msg, "%s", r->str);
        freeReplyObject(r);
        r = nullptr;
        *err = -EIO;
    }
    return r;
}
```

Чтение хэша в структуру использует то же `map_set` и ту же арену, что и строки PostgreSQL (`kit.h`, `map.h` и `map.c` из предыдущего раздела). После `HELLO 3` ответ на `HGETALL` приходит типом `REDIS_REPLY_MAP`, а не массивом, поэтому `switch` по типу ответа всегда содержит `default`.

```c
/* kv_hash.c */
#include "kv.h"
#include "map.h"
#include <errno.h>
#include <string.h>

int kv_load(kv *k, const char *key, const fdesc *d, size_t nd, void *obj, arena *a) {
    const char *argv[] = {"HGETALL", key};
    int err;
    redisReply *r = kv_cmd(k, &err, 2, argv, nullptr);
    if (!r) return err;
    switch (r->type) {
    case REDIS_REPLY_ARRAY:
    case REDIS_REPLY_MAP: break;
    default: err = -EBADMSG;
    }
    if (!err && r->elements == 0) err = -ENOENT;
    for (size_t i = 0; !err && i + 1 < r->elements; i += 2) {
        const redisReply *f = r->element[i], *v = r->element[i + 1];
        if (f->type != REDIS_REPLY_STRING || v->type != REDIS_REPLY_STRING) continue;
        for (size_t j = 0; j < nd; j++)
            if (!strcmp(f->str, d[j].name)) { err = map_set(&d[j], obj, v->str, a); break; }
    }
    freeReplyObject(r);
    return err;
}
```

Двоичное значение с нулевым байтом сохраняет длину, `strlen` её теряет; ошибка сервера (`WRONGTYPE`) приходит кодом `-EIO` с текстом в `k->msg`; после таймаута контекст непригоден до `kv_reconnect`. Таймаут чтения hiredis 1.2 сообщает как `REDIS_ERR_IO` с `errno == EAGAIN`, поэтому `kv_cmd` проверяет оба признака и читает `errno` сразу после вызова.

```c
/* kv_demo.c */
#include "kv.h"
#include <stdio.h>
#include <string.h>

int main(void) {
    kv k;
    if (kv_open(&k, 300) < 0) return 1;
    int err;
    const char *set[] = {"SET", "blob", "a\0b"};
    const size_t sl[] = {3, 4, 3};
    freeReplyObject(kv_cmd(&k, &err, 3, set, sl));
    const char *get[] = {"GET", "blob"};
    redisReply *r = kv_cmd(&k, &err, 2, get, nullptr);
    if (!r) return 1;
    printf("%zu %zu\n", r->len, strlen(r->str)); /* вывод: 3 1 */
    freeReplyObject(r);
    const char *bad[] = {"LPUSH", "blob", "x"};
    r = kv_cmd(&k, &err, 3, bad, nullptr);
    printf("%d %.9s\n", err, k.msg); /* вывод: -5 WRONGTYPE */
    const char *slow[] = {"BLPOP", "nolist", "2"};
    r = kv_cmd(&k, &err, 3, slow, nullptr);
    printf("%d\n", err); /* вывод: -110 */
    r = kv_cmd(&k, &err, 2, get, nullptr);
    printf("%d\n", err != 0); /* вывод: 1 */
    printf("%d\n", kv_reconnect(&k)); /* вывод: 0 */
    r = kv_cmd(&k, &err, 2, get, nullptr);
    printf("%d\n", r != nullptr && r->len == 3); /* вывод: 1 */
    freeReplyObject(r);
    kv_close(&k);
    return 0;
}
```

Схема ключей - префиксы с разделителем `:` по сущности и идентификатору (`app:session:7`), у временных данных сразу задают срок жизни (`EXPIRE` или `SET ... EX`); `KEYS` и `SCAN` по всему пространству в горячем пути не используют: `KEYS` блокирует сервер на время обхода. Структуру пишут в хэш одной командой `HSET` и читают в `struct` по таблице полей.

```c
/* hash_demo.c */
#include "kv.h"
#include <stdio.h>

typedef struct { uint32_t uid; const char *name; uint8_t role; int32_t hits; } session;
static const fdesc session_desc[] = {FIELD(session, uid), FIELD(session, name), FIELD(session, role), FIELD(session, hits)};

static void show(kv *k, const char *key) {
    char mem[256];
    arena a = arena_init(mem, sizeof mem);
    session s = {0};
    int err = kv_load(k, key, session_desc, 4, &s, &a);
    if (err < 0) printf("err %d\n", err);
    else printf("%u %s %u %d\n", s.uid, s.name ? s.name : "-", s.role, s.hits);
}

int main(void) {
    kv k;
    if (kv_open(&k, 300) < 0) return 1;
    int err;
    const char *set[] = {"HSET", "app:session:7", "uid", "7", "name", "bob", "role", "2", "hits", "-5"};
    const char *ttl[] = {"EXPIRE", "app:session:7", "3600"}, *hello[] = {"HELLO", "3"};
    freeReplyObject(kv_cmd(&k, &err, 10, set, nullptr));
    freeReplyObject(kv_cmd(&k, &err, 3, ttl, nullptr));
    show(&k, "app:session:7"); /* вывод: 7 bob 2 -5 */
    freeReplyObject(kv_cmd(&k, &err, 2, hello, nullptr));
    show(&k, "app:session:7"); /* вывод: 7 bob 2 -5 */
    show(&k, "app:session:8"); /* вывод: err -2 */
    kv_close(&k);
    return 0;
}
```

Конвейер (pipelining) складывает несколько команд в выходной буфер и читает ответы по порядку, поэтому пачка стоит одной сетевой задержки: `redisAppendCommand` для каждой команды, затем `redisGetReply` столько же раз. Прочитать нужно все ответы: пропущенный оставит поток рассинхронизированным.

```c
/* pipe_demo.c */
#include "kv.h"
#include <stdio.h>

int main(void) {
    kv k;
    if (kv_open(&k, 300) < 0) return 1;
    freeReplyObject(redisCommand(k.c, "DEL hits"));
    for (int i = 1; i <= 3; i++)
        if (redisAppendCommand(k.c, "INCRBY hits %d", i) != REDIS_OK) return 1;
    long long last = 0;
    for (int i = 0; i < 3; i++) {
        redisReply *r = nullptr;
        if (redisGetReply(k.c, (void **)&r) != REDIS_OK) return 1;
        last = r->integer;
        freeReplyObject(r);
    }
    printf("%lld\n", last); /* вывод: 6 */
    kv_close(&k);
    return 0;
}
```

Подписка занимает соединение целиком: в режиме подписки (RESP2) разрешены только команды подписки и `PING`, остальные отвергаются, поэтому подписчик получает собственное соединение и читает сообщения `redisGetReply`, а публикуют с другого.

```c
/* sub_demo.c */
#include "kv.h"
#include <stdio.h>

int main(void) {
    kv sub, pub;
    if (kv_open(&sub, 300) < 0 || kv_open(&pub, 300) < 0) return 1;
    int err;
    const char *s[] = {"SUBSCRIBE", "news"}, *p[] = {"PUBLISH", "news", "hello"}, *get[] = {"GET", "blob"};
    freeReplyObject(kv_cmd(&sub, &err, 2, s, nullptr));
    freeReplyObject(kv_cmd(&pub, &err, 3, p, nullptr));
    redisReply *r = nullptr;
    if (redisGetReply(sub.c, (void **)&r) != REDIS_OK) return 1;
    printf("%s %s\n", r->element[0]->str, r->element[2]->str); /* вывод: message hello */
    freeReplyObject(r);
    freeReplyObject(kv_cmd(&sub, &err, 2, get, nullptr));
    printf("%d\n", err); /* вывод: -5 */
    kv_close(&sub);
    kv_close(&pub);
    return 0;
}
```

#### Подводные камни

- Данные из внешнего источника не вставляют в саму строку формата: `redisCommand(c, "SET k a b")` разбирает пробел как границу аргументов, а `%` как спецификатор. Данные передают аргументом `%s` (значение остаётся одним аргументом, но обрывается на нулевом байте), `%b` с длиной или через `redisCommandArgv`.
- `reply->str` завершается нулём, но двоичное значение может содержать нулевые байты внутри: размер берут из `reply->len`, а не `strlen`.
- Ответ `NIL` (ключа нет) - тоже ответ, не ошибка и не `NULL`; освобождается он так же, как все: `freeReplyObject` на каждый полученный ответ.
- `switch` по `reply->type` без `default` ломается на типах RESP3 (`REDIS_REPLY_MAP`, `REDIS_REPLY_SET`, `REDIS_REPLY_PUSH` и других).
- Контекст `redisContext` не делят между потоками: каждому потоку своё соединение или пул.
- Сразу после таймаута или обрыва контекст пригоден только для переподключения: повторять нужно лишь идемпотентные команды, и не бесконечно.

## Тестируемость

### Прячь базу за таблицей операций и держи SQL в константах

Логику, которая зависит от базы, проверяют без сервера: репозиторий скрыт за таблицей операций ([паттерны](21-patterns.md)), боевая реализация вызывает libpq или hiredis, а в тесте подставляется массив в памяти с принудительными отказами. SQL лежит константами рядом с реализацией, значения идут параметрами, поэтому интеграционный тест на настоящем сервере прогоняет каждую константу один раз, а логику покрывают быстрые модульные тесты ([тесты](09-testing.md)).

```c
/* store.h */
#ifndef STORE_H
#define STORE_H
#include <stdint.h>

typedef struct { int32_t id; char text[32]; } note;

typedef struct store_ops {
    int (*find)(void *ctx, int32_t id, note *out);
    int (*save)(void *ctx, const note *n);
} store_ops;
typedef struct store { const store_ops *vt; void *ctx; } store;

/* note_edit - читает заметку, меняет текст, сохраняет.
 * Возврат: 0, -ENAMETOOLONG (текст не помещается), -ENOENT или код хранилища. */
int note_edit(store *s, int32_t id, const char *text);
#endif
```

```c
/* note.c */
#include "store.h"
#include <errno.h>
#include <string.h>

int note_edit(store *s, int32_t id, const char *text) {
    size_t len = strlen(text);
    if (len >= sizeof(((note *)0)->text)) return -ENAMETOOLONG;
    note n;
    int err = s->vt->find(s->ctx, id, &n);
    if (err < 0) return err;
    memcpy(n.text, text, len + 1);
    return s->vt->save(s->ctx, &n);
}
```

```c
/* main.c */
#include "store.h"
#include <errno.h>
#include <stdio.h>

typedef struct { note rows[4]; size_t n; int fail_save; } fake;

static int fake_find(void *ctx, int32_t id, note *out) {
    fake *f = ctx;
    for (size_t i = 0; i < f->n; i++)
        if (f->rows[i].id == id) { *out = f->rows[i]; return 0; }
    return -ENOENT;
}

static int fake_save(void *ctx, const note *n) {
    fake *f = ctx;
    if (f->fail_save) return -EIO;
    for (size_t i = 0; i < f->n; i++)
        if (f->rows[i].id == n->id) { f->rows[i] = *n; return 0; }
    return -ENOENT;
}

static const store_ops fake_ops = {fake_find, fake_save};

int main(void) {
    fake f = {.rows = {{1, "old"}}, .n = 1};
    store s = {&fake_ops, &f};
    printf("%d\n", note_edit(&s, 1, "new")); /* вывод: 0 */
    printf("%s\n", f.rows[0].text); /* вывод: new */
    printf("%d\n", note_edit(&s, 2, "x")); /* вывод: -2 */
    f.fail_save = 1;
    printf("%d\n", note_edit(&s, 1, "again")); /* вывод: -5 */
    return 0;
}
```

#### Подводные камни

- Подмена, повторяющая SQL (фальшивое хранилище, считающее по-своему), расходится с настоящим: ограничения, `NULL`, порядок и конфликты проверяет интеграционный тест на настоящем сервере.
- Интерфейс по числу операций растёт вместе с запросами: делай его по потребностям логики (найти, сохранить), а не по таблицам базы.
- Отказ внедряют в подмену флагом, как `fail_save` выше: ветку ошибки хранилища иначе не вызвать ([тесты](09-testing.md)).

## Коротко

### SQLite: встроенная база без сервера

Файл или память вместо сервера, цикл тот же: `sqlite3_prepare_v2`, привязка параметров `sqlite3_bind_*` (номера с единицы), `sqlite3_step` до `SQLITE_DONE`, `sqlite3_finalize`. Текст колонки из `sqlite3_column_text` действителен до следующего `step` или `finalize`; для постоянных строк `SQLITE_STATIC`, для временных `SQLITE_TRANSIENT`.

```c
#include <sqlite3.h>
#include <stdio.h>

int main(void) {
    sqlite3 *db = nullptr;
    if (sqlite3_open(":memory:", &db) != SQLITE_OK) { sqlite3_close(db); return 1; }
    sqlite3_exec(db, "CREATE TABLE t (id INTEGER PRIMARY KEY, name TEXT)", NULL, NULL, NULL);
    sqlite3_stmt *st = nullptr;
    sqlite3_prepare_v2(db, "INSERT INTO t (name) VALUES (?1)", -1, &st, NULL);
    sqlite3_bind_text(st, 1, "ann", -1, SQLITE_STATIC);
    printf("%d\n", sqlite3_step(st) == SQLITE_DONE); /* вывод: 1 */
    sqlite3_finalize(st);
    sqlite3_prepare_v2(db, "SELECT name FROM t WHERE id = ?1", -1, &st, NULL);
    sqlite3_bind_int(st, 1, 1);
    if (sqlite3_step(st) == SQLITE_ROW) printf("%s\n", (const char *)sqlite3_column_text(st, 0)); /* вывод: ann */
    sqlite3_finalize(st);
    sqlite3_close(db);
    return 0;
}
```

### COPY: массовая загрузка

Для тысяч строк `COPY таблица FROM STDIN` через `PQexec`, затем `PQputCopyData` порциями и `PQputCopyEnd`, после чего итог забирают `PQgetResult`; поток данных идёт одной командой вместо отдельного обмена на каждую строку.

### LISTEN и NOTIFY

После `LISTEN канал` сокет соединения (`PQsocket`) ждут в `poll`, по `POLLIN` вызывают `PQconsumeInput`, а уведомления выбирают `PQnotifies`; структуру `PGnotify` освобождают `PQfreemem`.

### Асинхронный hiredis

`redisAsyncContext` с адаптерами для циклов событий (`hiredis/adapters/libuv.h`, `libevent.h`) отдаёт ответы в функцию обратного вызова и не блокирует цикл; это замена синхронному `redisContext` в цикле событий libuv или libevent.

## Что ещё стоит знать

Для MariaDB и MySQL есть собственная клиентская библиотека, для встроенного хранилища ключ-значение - LMDB. Режим построчной выдачи `PQsetSingleRowMode` читает огромные результаты по одной строке без загрузки всего в память. Шифрование соединения с PostgreSQL включает параметр `sslmode` в строке подключения.
