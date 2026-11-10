# Сборка проекта

## Выбор системы

### Бери CMake с Ninja по умолчанию и держи Meson как равноправный вариант

Система сборки описывает проект целями (библиотека или программа со своими исходниками, флагами и зависимостями), а не списком команд, и CMake и Meson выражают это одинаково, поэтому выбор определяется окружением, а не возможностями. Умолчание для нового проекта: CMake с генератором Ninja, потому что его понимают IDE, clangd и пакетные менеджеры, а `FetchContent` и пресеты закрывают зависимости и профили. Модель Meson концептуально чище: короткий строгий язык, subprojects через wrap и cross-file из коробки; он разумен, когда проект начинается с нуля и внешних требований к CMake нет. Make остаётся для проектов из пары файлов, но только с `-MMD -MP`. Ниже один и тот же проект из библиотеки и программы в трёх системах.

```c
/* tally.h */
#ifndef TALLY_H
#define TALLY_H
#include <stddef.h>

/* tally_words - число слов, разделённых пробелами; s не NULL */
size_t tally_words(const char *s);
#endif
```

```c
/* tally.c */
#include "tally.h"
#include <ctype.h>

size_t tally_words(const char *s) {
    size_t n = 0;
    bool in_word = false;
    for (; *s; s++) {
        bool sp = isspace((unsigned char)*s);
        if (!sp && !in_word) n++;
        in_word = !sp;
    }
    return n;
}
```

```c
/* main.c */
#include <stdio.h>
#include "tally.h"

int main(void) {
    printf("%zu\n", tally_words("one two  three")); /* вывод: 3 */
    return 0;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(tally LANGUAGES C)

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

add_library(tally STATIC tally.c)
target_include_directories(tally PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_options(tally PRIVATE -Wall -Wextra -Wpedantic)

add_executable(demo main.c)
target_link_libraries(demo PRIVATE tally)
```

```text
# file: meson.build
project('tally', 'c', version: '1.0.0', meson_version: '>=1.1')

add_project_arguments('-std=c23', '-Wall', '-Wextra', '-Wpedantic', language: 'c')

tally = static_library('tally', 'tally.c', include_directories: include_directories('.'))
tally_dep = declare_dependency(link_with: tally, include_directories: include_directories('.'))

executable('demo', 'main.c', dependencies: tally_dep)
```

```text
# file: Makefile
CFLAGS = -std=c23 -Wall -Wextra -Wpedantic -MMD -MP
OBJ = main.o tally.o

demo: $(OBJ)
	$(CC) $(LDFLAGS) -o $@ $(OBJ)

-include $(OBJ:.o=.d)
```

```sh
cmake -S . -B build -G Ninja && cmake --build build && build/demo
meson setup build-meson && meson compile -C build-meson && build-meson/demo
make && ./demo
```

В CMake зависимость цели описывает `target_link_libraries`, а путь к заголовкам библиотеки едет вместе с ней (`PUBLIC`); в Meson то же делает `declare_dependency`. Компилятор выбирается переменной окружения `CC` при первой настройке (`CC=clang-20 cmake -S . -B build`).

#### Подводные камни

- Make без `-MMD -MP`. Правка заголовка не пересобирает объекты, которые его включают, и в программе остаётся старая раскладка структуры; `-MMD` пишет файлы зависимостей `.d`, `-MP` добавляет в них пустые цели для заголовков, чтобы удалённый заголовок не ломал сборку.
- Стандарт в CMake. `C_STANDARD 23` с `C_EXTENSIONS OFF` даёт `-std=c23` на Clang 19 и 20 и `-std=c2x` на GCC 14: это тот же режим языка под прежним именем; `C_EXTENSIONS ON` даёт `gnu23`. Канон книги, `-std=c23`, в примерах с CMake на GCC 14 означает именно этот режим.
- Стандарт в Meson 1.3.2. Опция `c_std=c23` не принимается (допустимо `c2x`), поэтому c23 задаётся через `add_project_arguments('-std=c23', language: 'c')`; настройка при этом печатает безвредное предупреждение о встроенной опции.
- Смена компилятора. CMake запоминает компилятор при первой настройке, и `CC=gcc-14` для уже настроенного каталога сборки игнорируется (в кэше остаётся прежний); для другого компилятора заводи новый каталог.

## Цели и профили

### Задавай флаги на целях и выражай профили пресетами

Набор флагов, заданный один раз на интерфейсной цели, подключается целиком, поэтому его части не расходятся между целями, а профили `dev`, `asan`, `tsan` и `release` отличаются только параметрами конфигурирования, поэтому набор не расходится между разработкой, CI и релизом. Какие флаги входят в какой профиль и почему, описано в [главе про инструментарий](01-toolchain.md); здесь только то, как это выражается.

Свойства принадлежат цели, а не проекту. `PRIVATE` нужен только самой цели, `PUBLIC` ей и всем, кто на неё ссылается (каталог заголовков библиотеки), `INTERFACE` только потребителям: интерфейсная цель без исходников служит пакетом флагов, который цель подключает одной строкой. Глобальный `CMAKE_C_FLAGS` в `CMakeLists.txt` не используй: он действует и на вендорные цели, подтянутые `FetchContent`, а профиль обязан менять пользователь, а не файл. Предупреждения как ошибки (`-Werror`) кладут в пресет разработки через `CMAKE_COMPILE_WARNING_AS_ERROR`, а не в цель: исходники, которые ставятся у других людей, этих флагов не получают.

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.24)
project(tally LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

set(SANITIZE "" CACHE STRING "санитайзеры: address,undefined или thread")

add_library(project_flags INTERFACE)
target_compile_options(project_flags INTERFACE
    -Wall -Wextra -Wpedantic -Wconversion -Wshadow -Wformat=2)
if(SANITIZE)
    target_compile_options(project_flags INTERFACE
        -fsanitize=${SANITIZE} -fno-sanitize-recover=all -fno-omit-frame-pointer)
    target_link_options(project_flags INTERFACE -fsanitize=${SANITIZE})
endif()

add_library(tally STATIC tally.c)
target_include_directories(tally PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(tally PRIVATE project_flags)

add_executable(demo main.c)
target_link_libraries(demo PRIVATE tally project_flags)
```

```text
# file: CMakePresets.json
{
  "version": 3,
  "configurePresets": [
    { "name": "base", "hidden": true, "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "cacheVariables": { "CMAKE_EXPORT_COMPILE_COMMANDS": "ON" } },
    { "name": "dev", "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug",
                          "CMAKE_COMPILE_WARNING_AS_ERROR": "ON" } },
    { "name": "asan", "inherits": "dev",
      "cacheVariables": { "SANITIZE": "address,undefined" } },
    { "name": "tsan", "inherits": "dev",
      "cacheVariables": { "SANITIZE": "thread" } },
    { "name": "release", "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" } }
  ]
}
```

```text
# file: meson.build
project('tally', 'c', meson_version: '>=1.1')

flags = ['-std=c23', '-Wall', '-Wextra', '-Wpedantic', '-Wconversion', '-Wshadow', '-Wformat=2']
add_project_arguments(flags, language: 'c')

tally = static_library('tally', 'tally.c')
tally_dep = declare_dependency(link_with: tally, include_directories: include_directories('.'))
executable('demo', 'main.c', dependencies: tally_dep)
```

Те же исходники, что в предыдущем разделе, лежат рядом.

```c
/* tally.h */
#ifndef TALLY_H
#define TALLY_H
#include <stddef.h>

size_t tally_words(const char *s);
#endif
```

```c
/* tally.c */
#include "tally.h"
#include <ctype.h>

size_t tally_words(const char *s) {
    size_t n = 0;
    bool in_word = false;
    for (; *s; s++) {
        bool sp = isspace((unsigned char)*s);
        if (!sp && !in_word) n++;
        in_word = !sp;
    }
    return n;
}
```

```c
/* main.c */
#include <stdio.h>
#include "tally.h"

int main(void) {
    printf("%zu\n", tally_words("one two  three")); /* вывод: 3 */
    return 0;
}
```

Профиль выбирается одной командой, и каждый получает свой каталог сборки.

```sh
cmake --preset asan && cmake --build build/asan && build/asan/demo
cmake --preset release && cmake --build build/release
meson setup build-asan -Db_sanitize=address,undefined -Dc_args=-fno-sanitize-recover=all
```

В Meson профиль задают встроенными опциями при `meson setup`; параллель между системами такая.

| профиль | CMake | Meson |
|---|---|---|
| dev | `--preset dev`: Debug, `CMAKE_COMPILE_WARNING_AS_ERROR=ON` | `-Dwerror=true` (тип сборки debug по умолчанию) |
| asan | `--preset asan`: `SANITIZE=address,undefined` | `-Db_sanitize=address,undefined -Dc_args=-fno-sanitize-recover=all` |
| tsan | `--preset tsan`: `SANITIZE=thread` | `-Db_sanitize=thread` |
| release | `--preset release`: `CMAKE_BUILD_TYPE=Release` | `--buildtype=release -Db_ndebug=true` |

#### Подводные камни

- Санитайзер задан только в флагах компиляции. Без `-fsanitize` на линковке (`target_link_options` в примере) сборка не соберёт программу; в Meson это делает `b_sanitize`.
- `NDEBUG` в Meson. Тип сборки `release` включает оптимизацию, но `assert` отключает отдельная опция `b_ndebug=true`; в CMake `Release` определяет `NDEBUG` сам.
- Копия набора флагов. Если в `CMakeLists.txt`, пресете и CI флаги повторены отдельно, они разойдутся; источник один, интерфейсная цель, остальные места только выбирают профиль.

## Объектные библиотеки

### Собирай модули с регистрацией через секцию объектной библиотекой или link_whole

Линкер берёт из статической библиотеки `.a` только те объекты, на которые есть ссылка, поэтому модули, которые лишь кладут запись в общую секцию (механизм описан в [главе про паттерны](21-patterns.md)), из архива не попадают и регистрация пуста. Объектная библиотека отдаёт объекты напрямую, а `link_whole` в Meson подключает архив целиком.

Два модуля регистрируются макросом, `main` считает записи секции: она получает оба объекта и выводит `2`.

```c
/* ext.h */
#ifndef EXT_H
#define EXT_H

struct entry { const char *name; };

#define REGISTER(n) \
    __attribute__((section("reg_entries"), used)) static const struct entry n##_entry = {#n};
#endif
```

```c
/* ext_a.c */
#include "ext.h"
REGISTER(ext_a)
```

```c
/* ext_b.c */
#include "ext.h"
REGISTER(ext_b)
```

```c
/* main.c */
#include <stdint.h>
#include <stdio.h>
#include "ext.h"

extern const struct entry __start_reg_entries[], __stop_reg_entries[];

int main(void) {
    size_t bytes = (size_t)((uintptr_t)__stop_reg_entries - (uintptr_t)__start_reg_entries);
    printf("%zu\n", bytes / sizeof(struct entry)); /* вывод: 2 */
    return 0;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(reg LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)

add_library(mods OBJECT ext_a.c ext_b.c)
add_executable(app main.c $<TARGET_OBJECTS:mods>)
```

```text
# file: meson.build
project('reg', 'c')
add_project_arguments('-std=c23', language: 'c')

mods = static_library('mods', 'ext_a.c', 'ext_b.c')
executable('app', 'main.c', link_whole: mods)
```

Если те же модули собрать обычной статической библиотекой (`STATIC` в CMake, `link_with` в Meson), компоновка падает с `undefined reference to __stop_reg_entries`; в CMake 3.24 и новее архив можно подключить целиком выражением `$<LINK_LIBRARY:WHOLE_ARCHIVE,mods>`. Выбирай объектную библиотеку, когда модули нужны только одной программе, и `link_whole`, когда архив собирается и для других целей.

## Зависимости

### Подключай системные библиотеки через pkg-config, а вендорные фиксируй версией и хэшем

Системная библиотека обновляется вместе с дистрибутивом (включая исправления безопасности), а вендорная версия записана в репозитории и даёт воспроизводимую сборку, поэтому для широко пакуемых библиотек бери системные, а для редких или требующих точной версии вендорные. pkg-config отдаёт флаги компиляции, флаги линковки и версию из файла `.pc`; обе системы читают его одной строкой, и версия отсекает слишком старую системную библиотеку. Вендорная зависимость, скачиваемая или распаковываемая при сборке, обязана быть закреплена хэшем архива: тогда подмена или повреждение источника останавливает сборку, а не попадает в программу.

В примере `zlib` берётся из системы, а библиотека `mini` сначала ищется в системе и только при отсутствии берётся из архива: `FIND_PACKAGE_ARGS` в `FetchContent_Declare` и файл `subprojects/mini.wrap` в Meson. Здесь архив лежит рядом (путь вместо адреса: так сборка не требует сети), в обычном проекте на его месте адрес релизного архива (в `source_url` у wrap и в `URL` у `FetchContent`). Хэш считает `sha256sum` по архиву. Файл `mini.wrap` лежит в каталоге `subprojects` проекта.

```c
/* main.c */
#include <stdio.h>
#include <zlib.h>
#include "mini.h"

int main(void) {
    printf("%d %d\n", mini_twice(21), zlibVersion()[0] == '1'); /* вывод: 42 1 */
    return 0;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.24)
project(app LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)

find_package(PkgConfig REQUIRED)
pkg_check_modules(ZLIB REQUIRED IMPORTED_TARGET zlib>=1.2)

include(FetchContent)
FetchContent_Declare(mini
    URL ${CMAKE_CURRENT_SOURCE_DIR}/third_party/mini-1.0.tar.gz
    URL_HASH SHA256=849f1d55a9e8db634a434b9fb4460c4561388fa8882e0947d2711c962131033d
    FIND_PACKAGE_ARGS 1.0 NAMES mini)
FetchContent_MakeAvailable(mini)

add_executable(demo main.c)
target_link_libraries(demo PRIVATE PkgConfig::ZLIB mini)
```

```text
# file: meson.build
project('app', 'c', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')

zlib = dependency('zlib', version: '>=1.2')
mini = dependency('mini', version: '>=1.0')

executable('demo', 'main.c', dependencies: [zlib, mini])
```

```text
# file: mini.wrap
[wrap-file]
directory = mini-1.0
source_url = mini-1.0.tar.gz
source_filename = mini-1.0.tar.gz
source_hash = 849f1d55a9e8db634a434b9fb4460c4561388fa8882e0947d2711c962131033d

[provide]
dependency_names = mini
```

Для сборки без сети в Meson есть режим `--wrap-mode=nodownload`; архив, уже лежащий в `subprojects/packagecache`, берётся оттуда, если хэш совпал.

#### Подводные камни

- Тег или ветка вместо хэша. `GIT_TAG` с именем ветки или тега можно переставить на другой коммит; закрепляй хэш коммита, а для архива `URL_HASH` и `source_hash`.
- Хэш не совпал. И CMake, и Meson останавливают настройку с ошибкой, а не собирают чужое; хэш обновляют сознательно вместе с версией.
- Системная библиотека подменяет вендорную молча. Так задумано (`FIND_PACKAGE_ARGS` и автоматический fallback wrap), но версия в сборке зависит от машины; для воспроизводимого релиза включай вендорный режим явно (`FETCHCONTENT_TRY_FIND_PACKAGE_MODE=NEVER` в CMake, `--force-fallback-for=mini` в Meson).
- Пакетные менеджеры. vcpkg и Conan решают ту же задачу вне репозитория (рецепт плюс бинарный кэш) и подключаются к CMake через `find_package`; для небольшого числа зависимостей хватает pkg-config и закреплённых архивов.

## Конфигурирование

### Выноси опции и проверки функций в config.h, который генерирует система сборки

Наличие функции libc или библиотеки при текущих флагах знает только сборка, а не препроцессор: `__has_include(<string.h>)` истинно везде и ничего не говорит о том, объявлена ли `strlcpy` при текущих feature-макросах. Результат проверки записывается в `config.h` как `0` или `1`, код ветвится по `#if HAVE_...` и содержит запасной вариант. Возможности языка и компилятора (атрибуты, встроенные функции) проверяют в коде через `__has_*`, см. [главу про возможности компилятора](08-compiler-features.md); версию библиотеки отдаёт pkg-config, а не макрос из её заголовка.

Здесь проверяется, объявлена ли `strlcpy` в `<string.h>` при `-std=c23` с `_GNU_SOURCE`. Опция сборки `WITH_STATS` попадает в тот же заголовок.

```text
# file: config.h.in
#ifndef CONFIG_H
#define CONFIG_H
#define APP_VERSION "@PROJECT_VERSION@"
#cmakedefine01 HAVE_STRLCPY
#cmakedefine01 WITH_STATS
#endif
```

```c
/* main.c */
#include "config.h"
#include <stdio.h>
#include <string.h>

int main(void) {
    char dst[8];
#if HAVE_STRLCPY
    strlcpy(dst, "hello world", sizeof dst);
#else
    snprintf(dst, sizeof dst, "%s", "hello world");
#endif
    printf("%s %s stats=%d\n", APP_VERSION, dst, WITH_STATS); /* вывод: 1.4.0 hello w stats=0 */
    return 0;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(app VERSION 1.4.0 LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)

option(WITH_STATS "собирать счётчики" OFF)

include(CheckSymbolExists)
set(CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
check_symbol_exists(strlcpy string.h HAVE_STRLCPY)
unset(CMAKE_REQUIRED_DEFINITIONS)

configure_file(config.h.in config.h)

add_executable(demo main.c)
target_compile_definitions(demo PRIVATE _GNU_SOURCE)
target_include_directories(demo PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
```

```text
# file: meson_options.txt
option('stats', type: 'boolean', value: false, description: 'собирать счётчики')
```

```text
# file: meson.build
project('app', 'c', version: '1.4.0', meson_version: '>=1.1')

cc_args = ['-std=c23', '-D_GNU_SOURCE']
add_project_arguments(cc_args, language: 'c')

cc = meson.get_compiler('c')
conf = configuration_data()
conf.set_quoted('APP_VERSION', meson.project_version())
conf.set10('HAVE_STRLCPY', cc.has_header_symbol('string.h', 'strlcpy', args: cc_args))
conf.set10('WITH_STATS', get_option('stats'))
configure_file(output: 'config.h', configuration: conf)

executable('demo', 'main.c')
```

В CMake шаблон `config.h.in` с `#cmakedefine01` превращается в `config.h` в каталоге сборки; в Meson заголовок строится из `configuration_data` без шаблона. Включение опции: `cmake -DWITH_STATS=ON` и `meson setup build -Dstats=true`.

#### Подводные камни

- Проверка с другими флагами, чем сборка. Результат верен только для тех флагов, при которых он получен: CMake применяет к проверкам `C_STANDARD`, но `_GNU_SOURCE` в `CMAKE_REQUIRED_DEFINITIONS` нужен явно (без него `strlcpy` не объявлена при `-std=c23`, и проверка даёт 0); проверки Meson не видят `add_project_arguments`, поэтому тот же список передают в `args:`.
- `has_function` без `prefix`. Он проверяет только компоновку, и для необъявленной функции ответит «есть»; с `prefix: '#include <string.h>'` учитывается и объявление, а `has_header_symbol` проверяет именно его (аналог `check_symbol_exists`).
- Кэш проверок. CMake хранит результат в `CMakeCache.txt` (`HAVE_STRLCPY:INTERNAL`) и не повторяет проверку при смене флагов; после смены удали запись или каталог сборки.
- `config.h` в публичном заголовке. Устанавливаемый заголовок не включает `config.h`: его макросы (`HAVE_...`) попадают в чужие единицы трансляции и конфликтуют с их собственными; подключай его только из `.c`-файлов, первой строкой после комментария.
- Проверка запуском. `try_run` в CMake и `run` в Meson запускают собранную программу и при кросс-сборке работают только с эмулятором (`CMAKE_CROSSCOMPILING_EMULATOR`, `exe_wrapper`); проверки компиляции и линковки эмулятора не требуют, см. [главу про кросс-компиляцию](22-cross-compilation.md).
- Версия libc вместо проверки. Где проверка системой сборки невозможна, остаётся `__GLIBC_PREREQ(2, 38)` из `<features.h>`, но она говорит о версии glibc, а не об объявлении при текущих feature-макросах.

## Генерация кода

### Генерируй исходники в каталоге сборки и объявляй зависимости явно

Генерируемый файл, у которого не объявлены вход и генератор, не пересобирается при их изменении, и в программе остаётся устаревшая таблица. Пример нейтральный: таблица `name value` строится из текстового файла `errors.def` программой `gen`, которая собирается той же системой.

```text
# file: errors.def
not_found 2
denied 13
busy 16
```

```c
/* gen.c */
#include <stdio.h>

int main(int argc, char **argv) {
    if (argc != 3) return 2;
    FILE *in = fopen(argv[1], "r"), *out = fopen(argv[2], "w");
    if (!in || !out) return 1;
    char name[64];
    int value;
    unsigned n = 0;
    fputs("#include \"table.h\"\n\nconst struct entry table[] = {\n", out);
    while (fscanf(in, "%63s %d", name, &value) == 2) {
        fprintf(out, "    {\"%s\", %d},\n", name, value);
        n++;
    }
    fprintf(out, "};\nconst unsigned table_len = %u;\n", n);
    fclose(in);
    return fclose(out) != 0;
}
```

```c
/* table.h */
#ifndef TABLE_H
#define TABLE_H

struct entry { const char *name; int value; };
extern const struct entry table[];
extern const unsigned table_len;
#endif
```

```c
/* main.c */
#include <stdio.h>
#include "table.h"

int main(void) {
    printf("%u %s\n", table_len, table[table_len - 1].name); /* вывод: 3 busy */
    return 0;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(app LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)

add_executable(gen gen.c)

set(table_c ${CMAKE_CURRENT_BINARY_DIR}/table.c)
add_custom_command(
    OUTPUT ${table_c}
    COMMAND gen ${CMAKE_CURRENT_SOURCE_DIR}/errors.def ${table_c}
    DEPENDS gen errors.def
    COMMENT "table.c из errors.def")

add_executable(demo main.c ${table_c})
target_include_directories(demo PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
```

```text
# file: meson.build
project('app', 'c', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')

gen = executable('gen', 'gen.c', native: true)

table_c = custom_target('table',
    input: 'errors.def',
    output: 'table.c',
    command: [gen, '@INPUT@', '@OUTPUT@'])

executable('demo', 'main.c', table_c, include_directories: include_directories('.'))
```

Результат кладётся в каталог сборки, а не в дерево исходников: репозиторий не пачкается, а очистка сборки удаляет и сгенерированное. Строка `DEPENDS` в CMake и `input` с `command` в Meson означают одно: перегенерируй, когда изменился генератор или входной файл.

#### Подводные камни

- Нет `errors.def` в `DEPENDS`. Правка файла не вызывает перегенерацию, и `ninja` отвечает «no work to do»; без `gen` в `DEPENDS` пересборка генератора не вызывает перегенерацию таблицы, поэтому перечисляй оба.
- Относительные пути. `COMMAND` в CMake выполняется в каталоге сборки, поэтому входной файл указывай абсолютным путём от `CMAKE_CURRENT_SOURCE_DIR`, как в примере.
- Генератор при кросс-сборке. Он запускается на машине сборки, а не на целевой: в Meson для этого `native: true` (при обычной сборке ничего не меняет, при кросс-сборке собирает цель хостовым компилятором), а в CMake единый проект собирает всё одним компилятором, поэтому хостовый генератор выносят в отдельный нативный проект (`ExternalProject_Add`) и подключают готовый исполняемый файл как `IMPORTED` цель; в кросс-режиме эта механика здесь не запускалась. Сама кросс-сборка описана в [главе про кросс-компиляцию](22-cross-compilation.md).
- Файлы-ресурсы. Встраивание данных в программу (картинки, шаблоны) устроено иначе и описано в [главе про ресурсы](23-resources-and-assembly.md).

## Установка и разделяемые библиотеки

### Устанавливай через GNUInstallDirs и задавай версию, видимость и rpath на цели

Установка должна работать с произвольным префиксом и с промежуточным каталогом `DESTDIR`, поэтому все пути строят из переменных системы сборки, а не пишут руками. Soname (имя, под которым программа запоминает нужную библиотеку; его номер меняют при несовместимом изменении интерфейса) задаётся здесь `SOVERSION`; зачем нужна скрытая по умолчанию видимость символов, объясняет [глава про API и модули](07-api-and-modules.md), здесь только механика.

Пример собирает разделяемую библиотеку `tally` с версией `1.2.0` и soname `libtally.so.1`, скрывает все символы, кроме помеченных `TALLY_API`, ставит программу с `RUNPATH`, указывающим на библиотеку рядом (`$ORIGIN/../lib`), и пишет файл `tally.pc`.

```c
/* tally.h */
#ifndef TALLY_H
#define TALLY_H
#include <stddef.h>

#if defined(__GNUC__)
#define TALLY_API __attribute__((visibility("default")))
#else
#define TALLY_API
#endif

TALLY_API size_t tally_words(const char *s);
#endif
```

```c
/* tally.c */
#include "tally.h"
#include <ctype.h>

static int is_word(char c) { return !isspace((unsigned char)c); }

size_t tally_words(const char *s) {
    size_t n = 0;
    for (; *s; s++)
        if (is_word(*s) && (s[1] == '\0' || !is_word(s[1]))) n++;
    return n;
}

size_t tally_internal(void) { return 0; }
```

```c
/* main.c */
#include <stdio.h>
#include "tally.h"

int main(void) {
    printf("%zu\n", tally_words("a bb ccc")); /* вывод: 3 */
    return 0;
}
```

```text
# file: tally.pc.in
prefix=@CMAKE_INSTALL_PREFIX@
libdir=${prefix}/@CMAKE_INSTALL_LIBDIR@
includedir=${prefix}/@CMAKE_INSTALL_INCLUDEDIR@

Name: tally
Description: подсчёт слов
Version: @PROJECT_VERSION@
Libs: -L${libdir} -ltally
Cflags: -I${includedir}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(tally VERSION 1.2.0 LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)
include(GNUInstallDirs)

add_library(tally SHARED tally.c)
set_target_properties(tally PROPERTIES
    VERSION ${PROJECT_VERSION} SOVERSION 1
    C_VISIBILITY_PRESET hidden)
target_include_directories(tally PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

add_executable(demo main.c)
target_link_libraries(demo PRIVATE tally)
set_target_properties(demo PROPERTIES
    INSTALL_RPATH "$ORIGIN/../${CMAKE_INSTALL_LIBDIR}")

configure_file(tally.pc.in tally.pc @ONLY)
install(TARGETS tally demo)
install(FILES tally.h TYPE INCLUDE)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/tally.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
```

```text
# file: meson.build
project('tally', 'c', version: '1.2.0', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')

tally = shared_library('tally', 'tally.c',
    version: meson.project_version(), soversion: '1',
    gnu_symbol_visibility: 'hidden', install: true)

executable('demo', 'main.c', link_with: tally,
    include_directories: include_directories('.'),
    install_rpath: '$ORIGIN/../' + get_option('libdir'), install: true)

install_headers('tally.h')
import('pkgconfig').generate(tally, description: 'подсчёт слов')
```

Установка во временный каталог и проверка результата:

```sh
cmake -S . -B build -G Ninja -DCMAKE_INSTALL_PREFIX=/opt/tally && cmake --build build
DESTDIR=/tmp/stage cmake --install build
nm -D --defined-only build/libtally.so | grep tally
readelf -d build/libtally.so | grep SONAME
cmake --install build && /opt/tally/bin/demo
PKG_CONFIG_PATH=/opt/tally/lib/pkgconfig pkg-config --cflags --libs tally
```

В списке символов только `tally_words`: `tally_internal` скрыта видимостью, а soname равен `libtally.so.1`. Внутри `DESTDIR` (`/tmp/stage/opt/tally`) лежат `bin/demo`, `include/tally.h` и `lib` с `libtally.so.1.2.0` и ссылками `libtally.so.1` и `libtally.so`; префикс в `tally.pc` остаётся `/opt/tally`, `DESTDIR` в файлы не попадает. Установленная программа находит библиотеку без `LD_LIBRARY_PATH`.

#### Подводные камни

- Разный каталог библиотек. При нестандартном префиксе на Ubuntu 24.04 CMake ставит библиотеки в `lib`, а Meson в `lib64`; `INSTALL_RPATH` и `.pc` должны строиться из `CMAKE_INSTALL_LIBDIR` и `get_option('libdir')`, а не из литерала, и при необходимости каталог задают явно (`--libdir=lib` в Meson).
- Префикс, заданный при установке. В CMake `tally.pc` создаётся при настройке и несёт `CMAKE_INSTALL_PREFIX` того момента; `cmake --install --prefix ...` сменит каталог установки, но не содержимое `.pc`, поэтому префикс задают при настройке (в Meson это `--prefix` у `meson setup`).
- `rpath` в оболочке. `$ORIGIN` должен дойти до линковщика буквально; в командной строке заключай его в одинарные кавычки, в Make пиши `$$ORIGIN`. Современный линковщик записывает его как `RUNPATH`.
- Статическая библиотека в разделяемой. Объекты обязаны быть позиционно-независимыми; не полагайся на умолчание компилятора и задай `POSITION_INDEPENDENT_CODE ON` в CMake (в Meson `b_staticpic` включён по умолчанию).

## Запуск тестов

### Регистрируй тесты в системе сборки и задавай таймаут, окружение и фильтры

Единая команда запуска со стабильным форматом результата нужна и разработчику, и CI, поэтому каждый тестовый исполняемый файл регистрируется в системе сборки, а не запускается скриптом. Какие тесты писать и чем, описывает [глава про тестирование](09-testing.md); здесь только механика запуска.

Два теста: `words` печатает вывод в формате TAP, `env` проверяет, что окружение теста задано системой сборки (так передают `ASAN_OPTIONS`).

```c
/* tally.h */
#ifndef TALLY_H
#define TALLY_H
#include <stddef.h>

size_t tally_words(const char *s);
#endif
```

```c
/* tally.c */
#include "tally.h"
#include <ctype.h>

size_t tally_words(const char *s) {
    size_t n = 0;
    bool in_word = false;
    for (; *s; s++) {
        bool sp = isspace((unsigned char)*s);
        if (!sp && !in_word) n++;
        in_word = !sp;
    }
    return n;
}
```

```c
/* test_words.c */
#include <stdio.h>
#include "tally.h"

static int failed;

static void check(int n, const char *name, size_t got, size_t want) {
    printf("%s %d - %s\n", got == want ? "ok" : "not ok", n, name);
    if (got != want) failed = 1;
}

int main(void) {
    printf("1..2\n");
    check(1, "empty", tally_words(""), 0);
    check(2, "three", tally_words(" a  b c "), 3);
    return failed;
}
```

```c
/* test_env.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    const char *v = getenv("ASAN_OPTIONS");
    printf("%s\n", v ? v : "unset");
    return v ? 0 : 1;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(tally LANGUAGES C)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_EXTENSIONS OFF)
enable_testing()

add_library(tally STATIC tally.c)
target_include_directories(tally PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

foreach(t words env)
    add_executable(test_${t} test_${t}.c)
    target_link_libraries(test_${t} PRIVATE tally)
    add_test(NAME ${t} COMMAND test_${t})
    set_tests_properties(${t} PROPERTIES
        TIMEOUT 10
        LABELS unit
        ENVIRONMENT "ASAN_OPTIONS=abort_on_error=1:detect_leaks=1")
endforeach()
```

```text
# file: meson.build
project('tally', 'c', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')

tally = static_library('tally', 'tally.c')
tally_dep = declare_dependency(link_with: tally, include_directories: include_directories('.'))
env = environment({'ASAN_OPTIONS': 'abort_on_error=1:detect_leaks=1'})

foreach t : ['words', 'env']
    exe = executable('test_' + t, 'test_' + t + '.c', dependencies: tally_dep)
    test(t, exe, timeout: 10, env: env, suite: 'unit',
         protocol: t == 'words' ? 'tap' : 'exitcode')
endforeach
```

```sh
cmake -S . -B build -G Ninja && cmake --build build
ctest --test-dir build --output-on-failure
ctest --test-dir build -R words -L unit
ctest --test-dir build --output-junit junit.xml
meson setup build-meson && meson test -C build-meson
meson test -C build-meson --suite unit words
```

```text
1/2 Test #1: words ............................   Passed    0.00 sec
2/2 Test #2: env ..............................   Passed    0.00 sec
100% tests passed, 0 tests failed out of 2

1/2 words OK              0.01s   2 subtests passed
2/2 env   OK              0.00s
```

`ctest -R` отбирает тесты по регулярному выражению имени, `-L` по метке, `-E` исключает; в Meson имя тестов идёт позиционным аргументом, `--suite` выбирает набор. JUnit пишет `ctest --output-junit` (файл ложится в каталог сборки), а Meson всегда кладёт `meson-logs/testlog.junit.xml`; протокол `tap` разбирает вывод теста на отдельные проверки, как в `2 subtests passed`.

#### Подводные камни

- Тесты не собраны. `ctest` ничего не собирает и на отсутствующий исполняемый файл отвечает `Not Run`; сначала `cmake --build`, а `meson test` пересобирает сам.
- Зависший тест. Без `TIMEOUT` (в Meson `timeout`) он блокирует CI навсегда; по истечении процесс убивается, и тест помечается `Timeout` (в Meson `TIMEOUT`, сигнал SIGTERM).
- Окружение. `ASAN_OPTIONS` и подобные переменные задавай свойством `ENVIRONMENT` или `env:` теста, а не в оболочке запуска: тогда они одинаковы у разработчика и в CI. Санитайзерные тесты запускай из каталога профиля `asan` (см. раздел про профили).
- TAP в CMake. `ctest` не разбирает вывод TAP и смотрит только код возврата, поэтому тест обязан возвращать ненулевое значение при провале, как `test_words`.
- Параллельность. Тесты в одном каталоге не должны делить временные файлы и порты: `ctest -j` и Meson запускают их одновременно.

## Коротко

### Ускоряй пересборку: Ninja, ccache и осторожность с unity

Ninja перестраивает только изменившееся и сам распараллеливает сборку; `ccache` кэширует результат компиляции и подключается как `-DCMAKE_C_COMPILER_LAUNCHER=ccache` (в Meson как префикс компилятора: `CC='ccache clang-20'`). Объединение файлов в одну единицу (`CMAKE_UNITY_BUILD=ON`, `-Db_unity=on`) ускоряет чистую сборку, но склеивает статические имена разных файлов в одну единицу: одинаковые имена `static`-функций начинают конфликтовать, поэтому включай его только на проверенных проектах.

```sh
cmake -S . -B build -G Ninja -DCMAKE_C_COMPILER_LAUNCHER=ccache
```

### Найди медленные файлы через -ftime-trace

Clang с `-ftime-trace` пишет рядом с объектным файлом JSON с разбивкой времени по этапам (разбор, включение заголовков, оптимизация, генерация кода), который открывают браузерные профилировщики трасс; GCC этой опции не знает. Включай на отдельных файлах, а не на всём проекте.

```sh
clang-20 -std=c23 -ftime-trace -c tally.c -o tally.o
```

### Ищи оптимизацию компоновки и отладочные файлы в соседних главах

Межмодульную оптимизацию, в том числе ThinLTO, разбирает [глава про производительность](20-performance.md); флаг `-gsplit-dwarf` (Clang 19 и 20, GCC 14) выносит отладочные данные в отдельные файлы `.dwo` рядом с объектными; смена компоновщика (`-fuse-ld=lld`) и всё, что касается чужой целевой платформы, описаны в [главе про кросс-компиляцию](22-cross-compilation.md). Файл `compile_commands.json` для clangd и clang-tidy описан в [главе про инструментарий](01-toolchain.md).

### Читай чужой ./configure

Проекты на autotools собираются тремя командами: `./configure --prefix=... && make && make install DESTDIR=...`; все опции показывает `./configure --help`, а `config.log` содержит команду каждой проверки и причину отказа, что полезнее всего при отладке чужой сборки. Свой новый проект на autotools не начинай.

### Собирай пакеты поверх установки в DESTDIR

Дистрибутивные пакеты (deb, rpm) собирают установкой в промежуточный каталог `DESTDIR` и упаковкой его содержимого; поэтому установка должна быть полной и не зависеть от машины сборки (см. практику про установку).

## Что ещё стоит знать

Для очень больших монорепозиториев с общим кэшем сборки существует Bazel, а для сборки из исходников чужого проекта нужно знать его систему, чаще всего CMake или autotools. Свои установленные библиотеки для потребителей на CMake экспортируют через `install(EXPORT)` и файл конфигурации пакета для `find_package`. Файлы цепочки инструментов (`CMAKE_TOOLCHAIN_FILE`, cross-file Meson), sysroot и `PKG_CONFIG_SYSROOT_DIR` относятся к кросс-сборке и описаны в [главе про кросс-компиляцию](22-cross-compilation.md).
