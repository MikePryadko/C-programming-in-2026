# Сборка, предупреждения, санитайзеры

## Флаги и санитайзеры

### Собирай код с единым строгим набором предупреждений, а -Werror включай только в разработке

Одинаковый набор предупреждений у всех разработчиков и в CI превращает типичные ошибки (сужающие преобразования, затенение переменных, формат из недоверенной строки) в диагностику на этапе компиляции. Набор принимают Clang и GCC:

```sh
cc -std=c23 -Wall -Wextra -Wpedantic -Wconversion -Wshadow -Wformat=2 \
   -Werror=implicit-function-declaration -c module.c
```

В разработке и в CI добавляй `-Werror`, чтобы предупреждение нельзя было оставить на потом. В поставляемых исходниках (библиотека, которую соберут чужим, более новым компилятором) `-Werror` не задавай: каждая новая версия компилятора добавляет предупреждения, и чужая сборка упадёт на коде, который у тебя собирался. `-Werror=implicit-function-declaration` в новых компиляторах и так ошибка; флаг фиксирует это явно, чтобы сборка не зависела от версии. Что именно ловит набор:

```c
/* Ошибка: сужающие преобразования, затенение и формат из недоверенной строки */
#include <stdio.h>
#include <stddef.h>

int sum(const int *a, size_t n) {
    int total = 0;
    for (int i = 0; i < n; i++) total += a[i];
    return total;
}

unsigned char clamp(int v) {
    return v;
}

void log_msg(const char *msg) {
    int n = 0;
    {
        int n = 1;
        printf(msg);
        (void)n;
    }
    (void)n;
}
```

```text
7:23: warning: comparison of integers of different signs: 'int' and 'size_t' [-Wsign-compare]
12:12: warning: implicit conversion loses integer precision: 'int' to 'unsigned char' [-Wimplicit-int-conversion]
18:13: warning: declaration shadows a local variable [-Wshadow]
19:16: warning: format string is not a string literal (potentially insecure) [-Wformat-security]
```

Вывод Clang; GCC печатает те же четыре места другими словами (сужение он относит к `-Wconversion`).

#### Подводные камни

- `-Wvla` в набор не входит: он срабатывает и на безобидные параметры `int a[n]` и `a[static n]`, которые рекомендует [глава про API](07-api-and-modules.md). Массивы переменной длины в локальных переменных запрещай правилом ревью.
- Явное приведение типа заглушает `-Wconversion`, но не проверяет диапазон: правила преобразований разобраны в [главе про типы](03-types-and-ub.md).
- Два набора флагов (в Makefile и в CI) расходятся быстро и дают разные предупреждения на одном коде; задавай набор в одном месте: как это делается в системе сборки, описано в [главе про сборку проекта](02-build.md).
- `#if` с неопределённым именем молча считается нулём: опечатка в имени макроса выключает ветку без диагностики. Включай `-Wundef` и для обязательных предусловий платформы пиши `#error`.

### Запускай все тесты под ASan и UBSan и останавливайся на первой ошибке

ASan находит выход за границы кучи, стека и глобалей, использование после освобождения и утечки (LSan входит в ASan), UBSan находит знаковое переполнение, неверные сдвиги, выход за границы массивов с известным размером и другое неопределённое поведение. Тестовая сборка, на которой это включено всегда, ловит класс ошибок, которые иначе проявляются на проде как редкие падения.

```sh
cc -std=c23 -O1 -g -fno-omit-frame-pointer \
   -fsanitize=address,undefined -fno-sanitize-recover=all test.c -o test
ASAN_OPTIONS=strict_string_checks=1:detect_stack_use_after_return=1 ./test
```

Ключевой флаг здесь `-fno-sanitize-recover=all`. По умолчанию UBSan печатает отчёт и продолжает работу, процесс завершается с кодом 0, и тест остаётся зелёным с пропущенной ошибкой; с флагом выход 1.

```c
/* Ошибка: UB, знаковое переполнение int */
#include <limits.h>
#include <stdio.h>

int main(int argc, char **argv) {
    (void)argv;
    int x = INT_MAX - 1 + argc;
    x += 1;
    printf("x=%d\n", x);
    return 0;
}
```

```text
main.c:8:7: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```

Остальные флаги: `-O1` оставляет код близким к исходному и достаточно быстрым, `-g` даёт строки в отчётах, `-fno-omit-frame-pointer` делает стеки вызовов читаемыми. `ASAN_OPTIONS` задаёт поведение во время выполнения:

- `strict_string_checks=1`: функции `str*` проверяют всю строку до нуля, а не только прочитанную часть; `strchr` по буферу без завершающего нуля без опции проходит, с опцией ASan сообщает `heap-buffer-overflow`.
- `detect_stack_use_after_return=1`: находит обращение к локальной переменной после возврата из функции; в новых версиях включено по умолчанию, явная строка фиксирует поведение независимо от версии.

Утечки находит LSan при выходе из программы: отчёт начинается с `LeakSanitizer: detected memory leaks` и называет стек выделения. Результат `malloc` обязан использоваться: если указатель нигде не читается, оптимизатор выбрасывает выделение целиком, и утечка пропадает вместе с ним. Выход за границу блока в куче ASan печатает как `heap-buffer-overflow`: вид ошибки, операция и размер, стек обращения и ниже стек выделения блока.

#### Подводные камни

- Флаг `-fsanitize=...` нужен и при компиляции, и при линковке; в системе сборки он стоит в обоих списках флагов.
- Оптимизатор убирает код, результат которого не используется, вместе с ошибкой (`char c = p[8]; (void)c;` при `-O1` под ASan проходит с кодом 0). Тестовый код должен использовать то, что проверяет.
- Статическая линковка с ASan: GCC отказывает (`cannot specify -static with -fsanitize=address`), Clang линкует с предупреждением про `dlopen`; для тестов собирай динамически.
- Тестовую сборку держи отдельным профилем, а не флагом поверх релизной: релизные флаги (см. следующую практику) скрывают ошибки от санитайзеров.

### Собирай релиз с защитными флагами

Защитные флаги не исправляют ошибки, а превращают их эксплуатацию в аварийное завершение: переполнение буфера, возврат по затёртому адресу, прыжок в произвольное место. Цена у большинства мала, а включать их надо явно: в Ubuntu 24.04 GCC сам определяет `_FORTIFY_SOURCE=3` при оптимизации и включает `-fstack-protector-strong`, `BIND_NOW` и PIE, а Clang без флагов даёт только PIE и `relro`. Явные флаги делают сборку одинаковой на любом дистрибутиве и компиляторе.

```sh
cc -std=c23 -O2 -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=3 \
   -fstack-protector-strong -fstack-clash-protection \
   -fcf-protection=full -fstrict-flex-arrays=3 -fno-delete-null-pointer-checks \
   -ftrivial-auto-var-init=zero -fPIE -pie -Wl,-z,relro -Wl,-z,now -Wl,-z,noexecstack main.c -o main
```

Все перечисленные флаги принимают Clang и GCC, кроме отмеченных ниже.

| Флаг | Что защищает |
| --- | --- |
| `-O2 -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=3` | Проверка размеров в `memcpy`, `strcpy`, `sprintf` и подобных: переполнение известного размера завершает программу с сообщением `buffer overflow detected` |
| `-fstack-protector-strong` | Канарейка перед адресом возврата: затирание его переполнением стека обнаруживается при выходе из функции (`stack smashing detected`) |
| `-fstack-clash-protection` | Зондирование страниц при больших кадрах стека: нельзя перескочить гарантийную страницу и попасть в чужую память |
| `-fcf-protection=full` | Метки допустимых переходов и теневой стек: защита от подмены косвенных переходов и возвратов; только x86_64 |
| `-mbranch-protection=standard` | Аналог для AArch64 (подпись адреса возврата и метки переходов); на x86_64 все три компилятора его отклоняют, Clang принимает при `--target=aarch64-linux-gnu` |
| `-fPIE -pie` | Исполняемый файл с произвольным адресом загрузки: ASLR работает и для кода |
| `-Wl,-z,relro -Wl,-z,now` | Таблицы импорта разрешаются при запуске и становятся только для чтения: нельзя подменить адрес библиотечной функции |
| `-Wl,-z,noexecstack` | Стек не исполняемый |
| `-fstrict-flex-arrays=3` | Компилятор и FORTIFY считают границей хвостового массива его объявленный размер: проверки границ начинают работать на хвостовых массивах |
| `-fno-delete-null-pointer-checks` | Компилятор не вправе удалить проверку на `nullptr`, стоящую после разыменования того же указателя |
| `-ftrivial-auto-var-init=zero` | Локальные переменные без инициализатора обнуляются: чтение неинициализированной переменной не выдаёт содержимое стека (утечка данных, управляемое значение); ошибка не исправляется, только обезвреживается |

Эффект `-fstrict-flex-arrays=3` виден на старом приёме с массивом из одного элемента:

```c
/* Ошибка: хвостовой массив из одного элемента вместо гибкого члена */
#include <stdio.h>
#include <stdlib.h>

struct msg { int n; char data[1]; };

int main(int argc, char **argv) {
    (void)argv;
    struct msg *m = malloc(sizeof *m + 16);
    if (!m) abort();
    for (int i = 0; i < 8; i++) m->data[i] = (char)('a' + i);
    printf("%c\n", m->data[argc + 3]);
    free(m);
    return 0;
}
```

Без флага `data[1]` в конце структуры считается гибким массивом, и проверки границ молчат; при `-fstrict-flex-arrays=3` и `-fsanitize=undefined` UBSan сообщает на первой же записи: `main.c:11:33: runtime error: index 1 out of bounds for type 'char[1]'`. Исправление: `char data[];`, а выделение `sizeof *m + 16` остаётся тем же.

#### Подводные камни

- `_FORTIFY_SOURCE` работает только при оптимизации: при `-O0` переполнение `strcpy` не прерывается. Перед `-D_FORTIFY_SOURCE=3` ставь `-U_FORTIFY_SOURCE`: если значение уже задано (дистрибутивом или системой сборки), второе определение даёт предупреждение о переопределении.
- `-fstrict-flex-arrays=3` перестаёт считать гибким старый приём «хвостовой массив из одного элемента», и его переполнение становится ошибкой времени выполнения. Замени хвостовые `[0]` и `[1]` на гибкий член `[]` (подробности про выделение под него в [главе про владение](04-memory-ownership.md)).
- `-fcf-protection=full` и `-mbranch-protection=standard` зависят от архитектуры; в системе сборки выбирай флаг по процессору, иначе кросс-сборка на другую архитектуру упадёт на неизвестной опции.
- `-Wl,-z,nodlopen` запрещает загрузку объекта через `dlopen`. Ставь его только исполняемым файлам; загружаемые модули (плагины, см. [главу про паттерны](21-patterns.md)) с ним `dlopen` отвергает с ошибкой `shared object cannot be dlopen()ed`.
- Релизный набор нужно проверять на готовом файле: `readelf -lW main` показывает `GNU_RELRO` и строку `GNU_STACK` с флагами `RW` без `E`, `readelf -dW main` показывает `BIND_NOW`, а `readelf -nW main` на x86_64 показывает `IBT, SHSTK`.

### Не переноси релизные флаги в тестовую сборку

Часть релизных флагов маскирует ошибки, которые тестовая сборка должна находить, поэтому профили собираются отдельно, а не наслаиваются. Правило: санитайзеры, `-O1 -g` и предупреждения в тестах; защитные флаги в релизе; пересечения нет.

Два флага опасны для тестов особенно:

- `-ftrivial-auto-var-init=zero` (или `=pattern`) инициализирует локальные переменные, и неинициализированное чтение исчезает: без флага MSan сообщает `use-of-uninitialized-value`, с флагом программа печатает «not positive» и выходит с кодом 0. Так же слепнут любые другие детекторы чтений неинициализированной памяти.
- `-fwrapv` и `-fno-strict-overflow` определяют знаковое переполнение как оборачивание, и UBSan его не видит: проверка `signed-integer-overflow` молчит, программа выходит с кодом 0. Явный `-fsanitize=signed-integer-overflow` Clang всё равно ловит, GCC 14 нет.

Ошибку, которую маскирует `-ftrivial-auto-var-init`, находит MSan (только Clang). Значение в примере приходит из функции, вызванной через указатель: если оптимизатор видит его насквозь, он фиксирует значение ещё до инструментации и ошибка пропадает.

```c
/* Ошибка: чтение элемента массива, который не записан на этом пути */
#include <stdio.h>

static void fill(int *a, int n) {
    if (n > 5) a[1] = 1;
}

int main(int argc, char **argv) {
    (void)argv;
    int a[4];
    void (*volatile f)(int *, int) = fill;
    f(a, argc);
    if (a[1] > 0) puts("positive");
    else puts("not positive");
    return 0;
}
```

```sh
clang -std=c23 -O1 -g -fsanitize=memory test.c -o test_msan
clang -std=c23 -O1 -g -fsanitize=memory -ftrivial-auto-var-init=zero test.c -o test_msan
```

Первая сборка завершается с кодом 1 и сообщением `WARNING: MemorySanitizer: use-of-uninitialized-value`; вторая печатает `not positive` и завершается с кодом 0.

```sh
# правильно: два независимых профиля
cc -std=c23 -O1 -g -fsanitize=address,undefined -fno-sanitize-recover=all test.c
cc -std=c23 -O2 -ftrivial-auto-var-init=zero -fwrapv app.c
```

Если проекту нужны оба флага в релизе (например, код сознательно полагается на оборачивание), это решение принимается до написания кода и записывается; тесты всё равно идут без них, чтобы реальные ошибки не прятались. Оборачивание, на которое код полагается, надёжнее выражать беззнаковой арифметикой или `ckd_*` (см. [главу про типы](03-types-and-ub.md)), чем флагом.

### Запускай TSan отдельной сборкой

TSan ищет гонки данных, а для этого ему нужно видеть каждое обращение к памяти и каждую синхронизацию, поэтому он несовместим с ASan и требует пересборки всего кода программы. Сборка отдельная, запуск отдельным шагом CI.

```sh
cc -std=c23 -O1 -g -fsanitize=thread -pthread test.c -o test_tsan
./test_tsan
```

```c
/* Ошибка: гонка данных, счётчик увеличивают два потока без синхронизации */
#include <pthread.h>
#include <stdio.h>

static int counter;

static void *worker(void *arg) {
    (void)arg;
    for (int i = 0; i < 1000; i++) counter++;
    return nullptr;
}

int main(void) {
    pthread_t t[2];
    for (int i = 0; i < 2; i++) pthread_create(&t[i], nullptr, worker, nullptr);
    for (int i = 0; i < 2; i++) pthread_join(t[i], nullptr);
    printf("%d\n", counter);
    return 0;
}
```

```text
==================
WARNING: ThreadSanitizer: data race (pid=41)
  Write of size 4 at 0x5555569e8718 by thread T2:
    #0 worker main.c:9:43
```

Программа завершается с кодом 66 (по умолчанию, при обнаружении гонки). Допустимые сочетания с ASan и MSan см. в таблице совместимости ниже.

#### Подводные камни

- В контейнере на новых версиях Linux TSan может падать на старте (`personality(... ADDR_NO_RANDOMIZE)`): профиль seccomp по умолчанию запрещает этот вызов, запускай контейнер с `--security-opt seccomp=unconfined`. GCC 14 в такой среде ещё и падает на `unexpected memory mapping` из-за ASLR; прогоны TSan там делай Clang.
- Статическая libc не поддерживается: GCC отказывает при `-static`, Clang только предупреждает о `dlopen`; собирай динамически.
- TSan требует заметно больше памяти, чем ASan.
- Гонку находят только на реально исполненных путях: тесты должны создавать конкуренцию (несколько потоков на одних данных); подробности и приёмы по потокам в [главе про потоки](12-concurrency.md).

### Подключи статический анализ: clang-tidy и анализатор GCC

Статический анализ находит ошибки без запуска, в том числе на путях, которые тесты не исполняют, и дополняет санитайзеры. Два инструмента: clang-tidy (набор проверок, настраивается файлом `.clang-tidy`) и анализатор GCC `-fanalyzer` (рассчитан на C; находит утечки, двойное освобождение, использование после освобождения и разыменование `nullptr` по путям исполнения).

```text
Checks: >
  -*,
  bugprone-*,
  -bugprone-easily-swappable-parameters,
  cert-err33-c,
  cert-err34-c,
  clang-analyzer-*,
  -clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling
CheckOptions:
  - key: bugprone-unused-return-value.CheckedFunctions
    value: '::read;::write;::close'
  - key: bugprone-sizeof-expression.WarnOnSizeOfPointer
    value: true
```

Запуск: `clang-tidy -p build/dev файл.c` (`-p` указывает каталог с `compile_commands.json`, см. раздел «Подключи compile_commands.json» ниже). Что ловят перечисленные проверки:

```c
/* Ошибка: printf в обработчике сигнала, atoi, sizeof указателя, непроверенный результат */
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void on_term(int sig) {
    (void)sig;
    printf("stop\n");
}

int port(const char *s) { return atoi(s); }

void dup_text(char *dst, const char *src) {
    memcpy(dst, src, sizeof(src));
}

int main(void) {
    signal(SIGTERM, on_term);
    return 0;
}
```

```text
bad.c:9: printf may not be asynchronous-safe; calling it from a signal handler may be dangerous [bugprone-signal-handler]
bad.c:12: 'atoi' ... will not report conversion errors; consider using 'strtol' instead [cert-err34-c]
bad.c:15: suspicious usage of 'sizeof()' on an expression of pointer type [bugprone-sizeof-expression]
bad.c:19: the value returned by this function should not be disregarded [cert-err33-c]
```

`cert-err33-c` требует проверять результат стандартных функций (`fputs`, `signal`, `fclose`); `bugprone-unused-return-value` делает то же для функций из `CheckedFunctions` (функции POSIX вроде `read` добавляй сам). `cert-err34-c` ловит `atoi` и родственные функции без проверки ошибок, `bugprone-sizeof-expression` для указателей включается только опцией `WarnOnSizeOfPointer`. Ещё срабатывают `bugprone-suspicious-realloc-usage` на `p = realloc(p, n)`, который теряет блок при отказе, `bugprone-narrowing-conversions` на присваивании `long` в `int`, `cert-env33-c` на вызов `system`, `cert-msc30-c` на `rand`.

Анализатор GCC включается флагом при обычной компиляции: `gcc -std=c23 -fanalyzer -O1 -c` на функции ниже сообщает о двойном `free` (`-Wanalyzer-double-free`) и об утечке открытого файла (`-Wanalyzer-file-leak`); Clang делает то же через `clang --analyze` (или `clang-analyzer-*` в clang-tidy).

```c
/* Ошибка: двойное освобождение и незакрытый файл */
#include <stdio.h>
#include <stdlib.h>

int load(const char *path) {
    FILE *f = fopen(path, "r");
    if (!f) return -1;
    char *buf = malloc(64);
    if (!buf) return -1;
    free(buf);
    free(buf);
    return 0;
}
```

#### Подводные камни

- Проверка `clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling` требует заменить `memcpy` на `memcpy_s`, которого в glibc нет; отключи её в конфигурации, как показано выше.
- Анализатор не точен и не полон: он пропускает ошибки и иногда ошибается. Предупреждение разбирай, а не заглушай приведением; шум подавляй точечно (`// NOLINT(имя-проверки)` с причиной), а не отключением проверки целиком.
- Предупреждения `-fanalyzer` и clang-tidy не заменяют тесты под санитайзерами: они находят разные ошибки.

### Определяй feature-test macros в системе сборки или первой строкой файла

Под строгим `-std=c23` функции POSIX и расширения glibc не объявлены: компилятор сообщает, что функция или константа не объявлена. Определить нужно `_GNU_SOURCE`, и только в одном месте: в системе сборки (`-D_GNU_SOURCE`) или первой строкой файла, до любого `#include`.

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <time.h>
#include <unistd.h>

int main(void) {
    int fds[2];
    if (pipe2(fds, O_CLOEXEC) != 0) return 1;
    struct timespec ts;
    if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0) return 1;
    char c = 'x';
    if (write(fds[1], &c, 1) != 1) return 1;
    char r = 0;
    if (read(fds[0], &r, 1) != 1) return 1;
    close(fds[0]);
    close(fds[1]);
    printf("%c\n", r); /* вывод: x */
    return 0;
}
```

При `-std=c23` без макроса не видны `pipe2`, `O_CLOEXEC`, `clock_gettime`, `CLOCK_MONOTONIC` и `strlcpy`. `-D_DEFAULT_SOURCE` открывает `strlcpy`, но не `pipe2`; `-D_POSIX_C_SOURCE=200809L` не открывает ни то, ни другое; `pipe2` и подобным нужен именно `_GNU_SOURCE` (`-std=gnu23` их тоже не открывает).

#### Подводные камни

- Двойное определение: `-D_GNU_SOURCE` в системе сборки и `#define _GNU_SOURCE` в файле дают предупреждение о переопределении (Clang: `-Wmacro-redefined`, под `-Werror` это ошибка). Выбери одно место. Если макрос задан в файле, оберни его в `#ifndef _GNU_SOURCE`; clang-tidy при определении в файле дополнительно жалуется проверкой `bugprone-reserved-identifier`, определение в системе сборки это обходит.
- Макрос должен стоять раньше любого `#include`: заголовки читают его при первом включении, определённый позже он ничего не меняет.
- `_GNU_SOURCE` открывает нестандартное: код становится непереносимым за пределы glibc. Если нужна переносимость, ограничься `_POSIX_C_SOURCE` и проверь, что нужные функции видны.

## Коротко

### Подключи compile_commands.json для clangd и clang-tidy

Файл `compile_commands.json` хранит точную команду компиляции каждого файла; по нему clangd подсвечивает ошибки и переходит по символам, а clang-tidy берёт те же флаги и пути заголовков. CMake создаёт его при `CMAKE_EXPORT_COMPILE_COMMANDS=ON` в каталоге сборки; редакторы и `clang-tidy -p build` читают его оттуда, достаточно сослаться на каталог или положить ссылку в корень проекта. Для проектов на `Makefile` файл создаёт обёртка `bear -- make`.

### Знай совместимость санитайзеров

Санитайзеры подключаются флагом `-fsanitize=`; допустимые сочетания сведены в таблицу.

| Сочетание | Результат |
| --- | --- |
| `address,undefined` | Допустимо на обоих компиляторах; LSan входит в ASan, отдельного флага не нужно |
| `address,leak` | Допустимо; отдельный `-fsanitize=leak` включает только LSan |
| `thread,address`, `thread,leak` | Отвергают Clang и GCC; TSan собирается отдельно |
| `thread,undefined` | Допустимо |
| `memory,address`, `memory,thread` | Отвергает Clang |
| `memory,undefined` | Допустимо; GCC MSan не поддерживает вовсе |

### Включай MSan для чтений неинициализированной памяти

MSan находит ветвление по значениям, которые не были записаны; работает только в Clang и требует собрать с ним весь код программы вместе с зависимостями. Флаг `-fsanitize-memory-track-origins=2` добавляет в отчёт место появления значения (`Uninitialized value was created by an allocation of 'a' in the stack frame`). В контейнере MSan может падать по той же причине, что и TSan (запрещённый `personality`). Пример с результатом лежит в практике про отдельные профили сборки.

## Что ещё стоит знать

Valgrind запускает готовый бинарник без пересборки (`--leak-check=full --track-origins=yes`): медленнее санитайзеров, но подходит там, где собрать MSan или ASan нельзя; статические проверки в CI дополняют cppcheck и CodeQL.

Режим `-fhardened` в GCC 14 одним флагом включает набор защит (в том числе `-D_FORTIFY_SOURCE=3` и `-ftrivial-auto-var-init=zero`), но не `-fstrict-flex-arrays`; Clang 19 и 20 этот флаг отвергают, в тестовую сборку его не добавляй.

UBSan в продакшене с минимальной ценой: Clang `-fsanitize-minimal-runtime` печатает короткое сообщение вида `ubsan: add-overflow` и продолжает, `-fsanitize-trap=undefined` на Clang и GCC завершает процесс сигналом `SIGILL`.

Расширение Clang `-fbounds-safety` (границы указателей в типах) доступно не во всех сборках: Clang 19 и 20 его отвергают как неизвестный флаг.

Проверку блокировок задаёт флаг `-Wthread-safety` (атрибуты `guarded_by`, `requires_capability`, только Clang) в командной строке, а не через `#pragma`.
