# Кросс-компиляция

## Компилятор и цель

### Задавай цель тройкой и выбирай компилятор, который умеет эту цель

Кросс-сборка различает три машины: build (на ней идёт сборка), host (на ней запустится результат) и target (нужна только самим компиляторам: код для какой машины они порождают). Для приложения важна пара build и host, и системе сборки сообщают лишь, что host другой: Meson называет его `[host_machine]`, CMake - `CMAKE_SYSTEM_NAME`. Машину описывает тройка (triple) вида «архитектура, иногда производитель, ОС, окружение»: `aarch64-linux-gnu`, `x86_64-linux-musl` (libc musl), `armv7-linux-gnueabihf` (32-битный ARM с аппаратными вещественными), `x86_64-w64-windows-gnu` (Windows, ABI mingw-w64), `s390x-linux-gnu`. Тройка определяет не только набор команд, но и размеры типов, ABI и libc, поэтому её задают явно.

Clang - один исполняемый файл на все цели, включённые в сборку (`clang --print-targets`), цель выбирает `--target=`. GCC собирается под одну тройку: кросс-компилятор ставится отдельным пакетом, а тройка стоит в имени файла (`aarch64-linux-gnu-gcc-14`, `x86_64-w64-mingw32-gcc`). Компилятор - лишь часть набора: нужны ещё заголовки и libc цели, стартовые файлы (`crt1.o`), `libgcc` или compiler-rt и линкер. На Debian и Ubuntu пакеты `gcc-14-aarch64-linux-gnu` и `libc6-dev-arm64-cross` приносят всё это вместе; для Clang к ним добавляют `lld`, один линкер на все цели.

Чтобы видеть, какая именно цель получилась, собери программу, которая печатает свойства своей платформы.

```c
/* target.c */
#include <limits.h>
#include <stdio.h>

int main(void) {
    unsigned one = 1;
    const char *order = *(unsigned char *)&one ? "little" : "big";
    printf("char %s, long %zu, ptr %zu, %s-endian\n",
           CHAR_MIN < 0 ? "signed" : "unsigned", sizeof(long),
           sizeof(void *), order);
}
```

Одна и та же программа на разных целях печатает разное.

```text
хост x86-64 (linux-gnu):  char signed, long 8, ptr 8, little-endian
aarch64-linux-gnu:        char unsigned, long 8, ptr 8, little-endian
s390x-linux-gnu:          char unsigned, long 8, ptr 8, big-endian
x86_64-linux-musl:        char signed, long 8, ptr 8, little-endian
```

Версия кросс-GCC определяется пакетом и может отличаться от GCC хоста: на Ubuntu 24.04 кросс-GCC для s390x и mingw-w64 имеют версию 13, где стандарт C23 называется `-std=c2x`. Clang 19 и 20 принимают `-std=c23` на любой цели.

### Для пакетной кросс-установки Clang хватает --target и lld, а --sysroot нужен настоящему rootfs

Clang знает архитектуру из `--target`, но не знает, где лежат заголовки, стартовые файлы и libc цели; пакеты `gcc-14-aarch64-linux-gnu` и `libc6-dev-arm64-cross` раскладывают их так, как ищет Clang для такой тройки, поэтому лишние флаги только мешают.

```sh
sudo apt install clang lld gcc-14-aarch64-linux-gnu libc6-dev-arm64-cross qemu-user
clang --target=aarch64-linux-gnu -std=c23 -fuse-ld=lld target.c -o target-a64
qemu-aarch64 -L /usr/aarch64-linux-gnu ./target-a64
```

Когда цель описана не пакетами, а готовым деревом файлов целевой системы (rootfs: распакованный образ, дерево сборщика образов), заголовки и библиотеки берутся оттуда: `--sysroot` указывает корень этого дерева, а `--gcc-install-dir` выбирает каталог `libgcc` внутри него, если там их несколько. Запуск под эмулятором использует тот же корень.

```sh
clang --target=aarch64-linux-gnu --sysroot=/opt/rootfs -fuse-ld=lld -std=c23 target.c -o target-a64
clang --target=aarch64-linux-gnu --sysroot=/opt/rootfs \
      --gcc-install-dir=/opt/rootfs/usr/lib/gcc/aarch64-linux-gnu/14 -fuse-ld=lld -std=c23 target.c -o target-a64
qemu-aarch64 -L /opt/rootfs ./target-a64
```

#### Подводные камни

- `--sysroot=/usr/aarch64-linux-gnu` на пакетной установке. Линковка падает: стартовые файлы (`crtbeginS.o`) и `libgcc` пакетного кросс-GCC лежат в `/usr/lib/gcc-cross/aarch64-linux-gnu`, вне этого корня, а подсказка `--gcc-install-dir` упирается уже в путь `libc.so.6`. Для пакетов `--sysroot` не задают.
- Rootfs без символических ссылок `lib` и `bin` на `usr/lib` и `usr/bin`. Скрипт `libc.so` в нём ссылается на абсолютный путь `/lib/aarch64-linux-gnu/libc.so.6`, и lld отвечает `cannot find ... inside /opt/rootfs`; пакеты, распакованные через `dpkg -x`, этих ссылок не создают, их добавляют руками.
- Неполный rootfs. Линковка прошла на таком наборе пакетов цели (Ubuntu 24.04): `libc6`, `libc6-dev`, `libgcc-s1`, `libgcc-14-dev`, `linux-libc-dev`, `libcrypt1`, `libcrypt-dev`; в нём должны быть и заголовки, и `libgcc`, и стартовые файлы.
- `-fuse-ld=lld` без `ld.lld` в `PATH`. Ubuntu ставит `ld.lld-19`, и Clang 20 отвечает `invalid linker name in argument '-fuse-ld=lld'`; помогает ссылка `ld.lld` на нужный бинарник в `PATH` или полный путь в `-fuse-ld=`.

## Системы сборки

### Описывай цель в toolchain-файле CMake и cross-файле Meson, а не в проекте

Компилятор, тройка и способ запуска тестов принадлежат окружению, а не проекту, поэтому их выносят в отдельный файл, и тот же `CMakeLists.txt` собирается нативно и для другой машины (профили и пресеты - в [главе про сборку](02-build.md)). Проект здесь - программа, которая проверяет собственную разрядность, и тест, который её запускает.

```c
/* main.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    printf("pointer %zu bytes\n", sizeof(void *));
    return sizeof(void *) == 8 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.28)
project(probe C)
set(CMAKE_C_STANDARD 23)
add_executable(probe main.c)
enable_testing()
add_test(NAME probe COMMAND probe)
```

Переменная `CMAKE_SYSTEM_NAME` включает режим кросс-сборки (`CMAKE_CROSSCOMPILING`). `CMAKE_C_COMPILER_TARGET` превращается в `--target`, `CMAKE_CROSSCOMPILING_EMULATOR` становится префиксом запуска для `ctest` и `try_run`. Переменные `CMAKE_FIND_ROOT_PATH_MODE_*` говорят, где искать: утилиты (`PROGRAM`) остаются на хосте, библиотеки, заголовки и пакеты (`LIBRARY`, `INCLUDE`, `PACKAGE`) берутся только из корня цели `CMAKE_FIND_ROOT_PATH`.

```text
# file: aarch64.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER clang)
set(CMAKE_C_COMPILER_TARGET aarch64-linux-gnu)
set(CMAKE_EXE_LINKER_FLAGS_INIT "-fuse-ld=lld")
set(CMAKE_CROSSCOMPILING_EMULATOR qemu-aarch64 -L /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

Для кросс-GCC в файле остаётся компилятор `aarch64-linux-gnu-gcc-14` (пакет `gcc-aarch64-linux-gnu` без номера ставит GCC 13), а строки `CMAKE_C_COMPILER_TARGET` и `CMAKE_EXE_LINKER_FLAGS_INIT` убираются.

В Meson та же роль у cross-файла: секция `[binaries]` называет компилятор (команда и `--target` списком), линкер `c_ld` и обёртку запуска `exe_wrapper`, секция `[host_machine]` описывает машину.

```text
# file: meson.build
project('probe', 'c', meson_version: '>=1.1')
add_project_arguments('-std=c23', language: 'c')
test('probe', executable('probe', 'main.c'))
```

```text
# file: aarch64.ini
[binaries]
c = ['clang', '--target=aarch64-linux-gnu']
c_ld = 'lld'
strip = 'aarch64-linux-gnu-strip'
exe_wrapper = ['qemu-aarch64', '-L', '/usr/aarch64-linux-gnu']

[host_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'aarch64'
endian = 'little'
```

Для кросс-GCC строка `c` превращается в `c = 'aarch64-linux-gnu-gcc-14'`, а `c_ld` убирается. Файлы подключают при настройке, и каждой цели нужен собственный каталог сборки.

```sh
cmake -S . -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=aarch64.cmake
cmake --build build && ctest --test-dir build --output-on-failure
meson setup build-meson --cross-file aarch64.ini
meson test -C build-meson
```

#### Подводные камни

- Одна цель на каталог сборки. CMake запоминает компилятор при первой настройке ([глава про сборку](02-build.md)), поэтому смена toolchain-файла в настроенном каталоге ненадёжна; заводи отдельный каталог на каждую цель.
- Утилита, которую сборка запускает сама (генератор кода). Собранная для цели, она не выполнится на машине сборки; такие цели собирают для хоста.
- Проверки configure, которые запускают программу (`try_run` в CMake, `cc.run` в Meson). Без эмулятора CMake требует заполнить кэш вручную, а с `CMAKE_CROSSCOMPILING_EMULATOR` или `exe_wrapper` проверка идёт под qemu и работает; проверки без запуска (компиляция, линковка, наличие символа) не зависят от эмулятора.
- Стандарт в Meson 1.3.2. Опция `c_std=c23` не принимается, поэтому, как и в главе про сборку, стандарт задаётся через `add_project_arguments`.

## Запуск

### Запускай бинарники другой архитектуры под qemu-user и подключай его к тестам

qemu-user исполняет программу чужой архитектуры, переводя её системные вызовы в вызовы хоста, поэтому кросс-собранные тесты можно прогнать без железа и без виртуальной машины. Динамическому бинарнику эмулятор должен подсунуть загрузчик и libc цели: каталог с ними передают флагом `-L` или переменной `QEMU_LD_PREFIX`; статический бинарник ничего не просит. Пакет `qemu-user-binfmt` регистрирует эмуляторы в binfmt_misc, после чего `./prog` запускается напрямую, но в контейнерах без binfmt_misc этого нет, и примеры главы вызывают эмулятор явно.

```sh
qemu-aarch64 -L /usr/aarch64-linux-gnu ./target-a64
QEMU_LD_PREFIX=/usr/aarch64-linux-gnu qemu-aarch64 ./target-a64
```

Системы сборки получают эмулятор один раз (`CMAKE_CROSSCOMPILING_EMULATOR`, `exe_wrapper`), и дальше `ctest` и `meson test` запускают тесты цели как обычные ([механика запуска тестов](02-build.md)).

#### Подводные камни

- Запуск без `-L`. Эмулятор отвечает `Could not open '/lib/ld-linux-aarch64.so.1': No such file or directory` и завершается с кодом 255; ошибка выглядит как отсутствие файла самой программы, а не загрузчика.
- Санитайзеры. Примеры этой главы собраны без ASan и UBSan: библиотеки санитайзеров Clang ставятся для архитектуры хоста, и для цели их надо собирать или ставить отдельно. Логику прогоняют нативно под санитайзерами, а кросс-сборка проверяет то, что от цели зависит: размеры типов, порядок байт, знаковость `char`, выравнивание.
- Скорость. Эмулятор заметно медленнее железа, поэтому таймауты, нагрузочные тесты и замеры производительности на нём бессмысленны.

## Зависимости

### Ставь библиотеки цели рядом с компилятором и направляй pkg-config на цель

Сборка против библиотеки хоста даёт либо ошибку линковки, либо, хуже, бинарник, который собран с заголовками одной архитектуры и линкуется с другой, поэтому у кросс-сборки отдельное хранилище заголовков и библиотек цели. На Debian и Ubuntu это multiarch: пакеты `имя:arm64` кладут файлы в `/usr/lib/aarch64-linux-gnu` рядом с пакетами хоста. Для arm64 на Ubuntu нужен отдельный источник, `ports.ubuntu.com`, а источники хоста ограничиваются архитектурой amd64; в формате deb822 это выглядит так.

```sh
sudo dpkg --add-architecture arm64
sudo sed -i 's/^Types: deb$/Types: deb\nArchitectures: amd64/' /etc/apt/sources.list.d/ubuntu.sources
sudo tee /etc/apt/sources.list.d/arm64.sources <<'EOF'
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports
Suites: noble noble-updates noble-security
Components: main universe
Architectures: arm64
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
sudo apt update && sudo apt install zlib1g-dev:arm64
```

Флаги библиотеки берут из её `.pc`-файла, но искать его надо только в дереве цели: `PKG_CONFIG_LIBDIR` заменяет путь поиска, а не дополняет его.

```sh
export PKG_CONFIG_LIBDIR=/usr/lib/aarch64-linux-gnu/pkgconfig:/usr/share/pkgconfig
aarch64-linux-gnu-gcc-14 zv.c $(pkg-config --cflags --libs zlib) -o zv
qemu-aarch64 -L /usr/aarch64-linux-gnu ./zv
```

Если цель описана rootfs, поиск ведут внутри него: `PKG_CONFIG_LIBDIR=/opt/rootfs/usr/lib/aarch64-linux-gnu/pkgconfig` и `PKG_CONFIG_SYSROOT_DIR=/opt/rootfs`, после чего `pkg-config --cflags --libs zlib` печатает `-I/opt/rootfs/usr/include -L/opt/rootfs/usr/lib/aarch64-linux-gnu -lz`. В CMake тот же корень задают `CMAKE_SYSROOT` рядом с `CMAKE_FIND_ROOT_PATH_MODE_*` из прошлого раздела, и `find_package(ZLIB)` находит `/opt/rootfs/usr/lib/aarch64-linux-gnu/libz.so`; запуск тестов получает `-L /opt/rootfs`. В Meson корень и каталог `.pc` лежат в `[properties]`, а флаги sysroot передаются компилятору и линкеру.

```text
set(CMAKE_SYSROOT /opt/rootfs)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_CROSSCOMPILING_EMULATOR qemu-aarch64 -L /opt/rootfs)
```

```text
[built-in options]
c_args = ['--sysroot=/opt/rootfs']
c_link_args = ['--sysroot=/opt/rootfs']

[properties]
sys_root = '/opt/rootfs'
pkg_config_libdir = '/opt/rootfs/usr/lib/aarch64-linux-gnu/pkgconfig'
```

#### Подводные камни

- `PKG_CONFIG_PATH` вместо `PKG_CONFIG_LIBDIR`. Каталог цели лишь добавляется к путям хоста; библиотека, которой нет у цели, находится в системе хоста, и ошибка проявляется поздно, при линковке. С `PKG_CONFIG_LIBDIR` такой запрос завершается ошибкой сразу.
- `PKG_CONFIG_SYSROOT_DIR` не задан при rootfs. Пути в `.pc`-файлах абсолютны внутри rootfs (`/usr/include`), и без префикса компилятор ищет их на хосте.
- Готовая обёртка вместо переменных. Пакет `pkgconf:arm64` приносит `aarch64-linux-gnu-pkg-config`, которая ищет `.pc`-файлы цели сама; в Meson её имя указывают в `pkg-config` секции `[binaries]` cross-файла.
- `CMAKE_FIND_ROOT_PATH` из toolchain-файла предыдущего раздела (`/usr/aarch64-linux-gnu`) не включает `/usr`, и `find_package(ZLIB)` для библиотеки из multiarch отвечает `Could NOT find ZLIB`. Для multiarch в корень добавляют `/usr`: `set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu /usr)`, и CMake находит `/usr/lib/aarch64-linux-gnu/libz.so`, потому что каталог библиотек выбирает по архитектуре компилятора.

## Переносимый Linux-бинарник

### Для разных дистрибутивов собирай статику на musl или динамику против старейшей glibc

Динамический бинарник требует у системы libc не младше той, символы которой он использует, и собранная на свежем дистрибутиве программа на старом не стартует: загрузчик сообщает, что не нашёл нужную версию символов `GLIBC_...`; версию поднимает любая одна функция. Статическая сборка на musl не зависит от libc системы и запускается в любой системе Linux нужной архитектуры.

```c
/* copy.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>

int main(void) {
    char dst[8];
    size_t n = strlcpy(dst, "portable", sizeof dst);
    printf("%zu %s\n", n, dst); /* вывод: 8 portabl */
}
```

Один вызов `strlcpy` (в glibc он появился в 2.38) делает бинарник непригодным для дистрибутивов старше этой версии; границу видно по символам. Статическую сборку на musl проверяет `file`.

```sh
musl-gcc -static -std=c2x -O2 copy.c -o copy-musl
file copy-musl
gcc -std=c23 -O2 copy.c -o copy-glibc
objdump -T copy-glibc | grep -o 'GLIBC_[0-9.]*' | sort -Vu | tail -1
```

```text
copy-musl: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, ...
GLIBC_2.38
```

Если нужна динамическая сборка, её ведут в контейнере или chroot со старейшим поддерживаемым дистрибутивом, а проверку `objdump -T` ставят в конвейер сборки, чтобы новый символ не поднял минимальную версию незаметно. Статическая glibc выбором не является: имена пользователей и хостов (`getpwnam`, `getaddrinfo`) и `dlopen` загружают модули NSS из системы, и такой бинарник всё равно привязан к системным библиотекам той же версии; линковка предупреждает об этом (`Using 'getpwnam' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking`).

#### Подводные камни

- Стек потока в musl по умолчанию 128 КиБ против 8 МиБ в glibc. Код с большими локальными массивами в потоках падает только на musl; размер задают явно: `pthread_attr_setstacksize(&attr, 1 << 20)` перед `pthread_create` (на musl проверено: `pthread_attr_getstacksize` после этого возвращает 1 МиБ).
- `ifunc` и `target_clones` на musl не работают: динамический загрузчик отвечает `unsupported relocation type 37`, статическая сборка завершается без результата. Выбор реализации во время выполнения делают через указатель и `__builtin_cpu_supports` ([возможности компилятора](08-compiler-features.md)).
- Пользователи и группы. `getpwnam` в статической сборке на musl работает, но модулей NSS (LDAP, sssd) у musl нет, и пользователи из сетевых каталогов не находятся; для серверной программы это может быть решающим ограничением. Переводы gettext при этом работают и без `libintl` ([локализация](18-localization.md)).

## Windows

### Собирай для Windows через mingw-w64 или Clang и держи платформенные вызовы в одном слое

Для Windows нужен компилятор с заголовками и библиотеками Windows API: пакет `mingw-w64` (`x86_64-w64-mingw32-gcc`) или Clang с тройкой `x86_64-w64-windows-gnu`, который берёт те же заголовки mingw-w64. Результат - исполняемый файл PE32+. Запускать Windows-программы в проверочной среде книги нечем (Wine нет), поэтому пример проверен только компиляцией и командой `file`, а поведение при запуске в главе не утверждается. Платформенные различия прячут за узким интерфейсом, как описано в [главе про системный уровень](13-system-io.md): здесь это монотонные миллисекунды и пауза, а реализацию выбирает сборка.

```c
/* plat.h */
#ifndef PLAT_H
#define PLAT_H
#include <stdint.h>

/* plat_now_ms - монотонное время в миллисекундах; точка отсчёта произвольна.
 * plat_sleep_ms - пауза не короче ms миллисекунд. */
int64_t plat_now_ms(void);
void plat_sleep_ms(unsigned ms);
#endif
```

```c
/* plat_posix.c */
#define _GNU_SOURCE
#include "plat.h"
#include <stdlib.h>
#include <time.h>

int64_t plat_now_ms(void) {
    struct timespec ts;
    if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0) abort();
    return (int64_t)ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}

void plat_sleep_ms(unsigned ms) {
    struct timespec ts = {ms / 1000, (long)(ms % 1000) * 1000000L};
    while (nanosleep(&ts, &ts) != 0) {}
}
```

```c
/* plat_win.c */
#include "plat.h"
#include <windows.h>

int64_t plat_now_ms(void) { return (int64_t)GetTickCount64(); }

void plat_sleep_ms(unsigned ms) { Sleep(ms); }
```

```c
/* main.c */
#include "plat.h"
#include <stdio.h>

int main(void) {
    int64_t t0 = plat_now_ms();
    plat_sleep_ms(20);
    printf("%d\n", plat_now_ms() - t0 >= 20); /* вывод: 1 */
}
```

Одна и та же `main.c` собирается с `plat_posix.c` на Linux и с `plat_win.c` на Windows.

```sh
x86_64-w64-mingw32-gcc -std=c2x main.c plat_win.c -o app.exe
clang --target=x86_64-w64-windows-gnu -std=c23 -fuse-ld=lld main.c plat_win.c -o app.exe
file app.exe
```

```text
app.exe: PE32+ executable (console) x86-64, for MS Windows, ...
```

#### Подводные камни

- `long` на 64-битной Windows 32-битный (модель LLP64), а `size_t` и указатели 64-битные: `__SIZEOF_LONG__` равен 4, `__SIZEOF_SIZE_T__` равен 8. Формат `%lu` для `size_t` GCC отвергает при `-Wformat`: `size_t` там `long long unsigned int`. Размеры печатают `%zu`, ширину целых фиксируют типами из `<stdint.h>` и `PRIu64`.
- `wchar_t` занимает 2 байта (`__SIZEOF_WCHAR_T__` равен 2), на Linux 4: код, который считает его кодовой точкой Unicode, на Windows работает с UTF-16.
- Отсутствие POSIX. `fork` не объявлен, заголовков `sys/socket.h` и `threads.h` нет, сокеты даёт Winsock (`winsock2.h`, библиотека `ws2_32`), потоки - Windows API или pthread из winpthreads (`-lpthread` при линковке). Эти места собирают в слой, а не размазывают по коду условной компиляцией.
- Пути и строки. Разделитель `\` и буква диска ломают код, разбирающий путь по `/`, а узкие строки API Windows живут в системной кодовой странице, не в UTF-8.
- Текстовый режим файлов. Windows превращает `\n` в `\r\n` при записи в текстовом режиме; для данных открывают `"rb"` и `"wb"` ([системный уровень](13-system-io.md)).
- ABI Microsoft. Тройка `x86_64-pc-windows-msvc` и драйвер `clang-cl` собирают код под ABI MSVC и требуют Windows SDK; для библиотек, которые линкуют с MSVC-кодом, нужна именно она, а не mingw.

## Переносимость кода

### Фиксируй допущения о платформе через static_assert и проверяй код на другой архитектуре

Размеры типов, знаковость `char` и порядок байт меняются от цели к цели, и код, который молча их предполагает, ломается на другой машине без единого предупреждения, поэтому допущения записывают в сам код, где ошибка остановит сборку. Три основные модели размеров отличаются числом байт в `long` и указателе.

| Модель | Где | int | long | указатель |
|---|---|---|---|---|
| ILP32 | 32-битные Linux и ARM | 4 | 4 | 4 |
| LP64 | 64-битные Linux и macOS | 4 | 8 | 8 |
| LLP64 | 64-битная Windows | 4 | 4 | 8 |

Значения цели читают у самого компилятора: `clang --target=ТРОЙКА -dM -E -x c /dev/null` печатает все предопределённые макросы (`__SIZEOF_LONG__`, `__BYTE_ORDER__`, `__CHAR_UNSIGNED__`), для GCC тот же вывод даёт `gcc -dM -E -x c /dev/null` кросс-компилятора.

```sh
clang --target=aarch64-linux-gnu -dM -E -x c /dev/null | grep -E '__CHAR_UNSIGNED__|__SIZEOF_LONG__|__BYTE_ORDER__'
```

```text
#define __BYTE_ORDER__ __ORDER_LITTLE_ENDIAN__
#define __CHAR_UNSIGNED__ 1
#define __SIZEOF_LONG__ 8
```

Допущения собирают в одном заголовке: он включается в начале и отказывается компилироваться там, где они неверны (в C23 `static_assert` - ключевое слово, сообщение необязательно).

```c
/* abi.h */
#ifndef ABI_H
#define ABI_H
#include <limits.h>
#include <stddef.h>
#include <stdint.h>

static_assert(CHAR_BIT == 8);
static_assert(sizeof(long) == 8, "код рассчитан на LP64, не на Windows");
static_assert(sizeof(size_t) == sizeof(uintptr_t));
#endif
```

Программа ниже показывает остальное: на aarch64 и s390x `char` беззнаковый, поэтому ловушка с `isalpha(c)` из [главы про строки](06-strings-and-buffers.md) превращается в воспроизводимую ошибку, а s390x ещё и big-endian.

```c
/* chars.c */
#include "abi.h"
#include <stdio.h>

int main(void) {
    char c = (char)0xE9;
    printf("char 0xE9 is %d\n", (int)c);
    puts(__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__ ? "little" : "big");
}
```

```text
x86-64:   char 0xE9 is -23, little
aarch64:  char 0xE9 is 233, little
s390x:    char 0xE9 is 233, big
```

Эти три прогона делают на живых машинах или под qemu-user: aarch64 через `qemu-aarch64 -L /usr/aarch64-linux-gnu`, big-endian s390x через `s390x-linux-gnu-gcc -std=c2x` и `qemu-s390x -L /usr/s390x-linux-gnu`.

#### Подводные камни

- `long` для 64-битных значений. На Windows он 32-битный; берут `int64_t` и `PRId64` из `<inttypes.h>`.
- Структуры пишут в файл и сокет целиком. Дополнение (padding) и порядок байт зависят от цели; сообщение собирают по полям ([системный уровень](13-system-io.md), [сериализация](14-serialization.md)).
- `-march=native` в поставляемой сборке. Компилятор берёт набор команд процессора машины сборки, и на более старом процессоре программа падает с `SIGILL`. Для распространения задают базовый уровень: `-march=x86-64-v2` (без AVX2) или `-march=x86-64-v3` (с AVX2), на AArch64 `-march=armv8-a` с нужными расширениями, а быстрые ветки выбирают во время выполнения ([возможности компилятора](08-compiler-features.md)).
- Порядок байт, определяемый во время выполнения через приведение указателя. Макрос `__BYTE_ORDER__` (расширение GCC и Clang, есть и в mingw-w64) вычисляется при компиляции и годится для `static_assert`; в C23 то же даёт `__STDC_ENDIAN_NATIVE__` из `<stdbit.h>`, который есть в glibc 2.39, но отсутствует в mingw-w64.

## Коротко

### Для микроконтроллеров собирай freestanding и подключай свой скрипт компоновки

Без операционной системы нет libc, загрузчика и `main` как точки входа: код компилируют с `-ffreestanding`, а начальный адрес, таблицу векторов и размещение секций задают файл запуска и скрипт компоновки (`MEMORY` и `SECTIONS`); если нужны `printf` и `malloc`, подключают newlib или picolibc. Пакеты `gcc-arm-none-eabi` и `libnewlib-arm-none-eabi` дают компилятор и такую libc, `-mcpu=cortex-m4 -mthumb` выбирают модель процессора и набор команд. Файл запуска и скрипт компоновки микроконтроллера здесь не разобраны; ассемблерные файлы для x86-64 и AArch64 разбирает [глава про ассемблер и ресурсы](23-resources-and-assembly.md).

```sh
arm-none-eabi-gcc -std=c2x -mcpu=cortex-m4 -mthumb -ffreestanding -nostdlib -Os -T link.ld start.c main.c -o fw.elf
file fw.elf
```

Результат проверен компиляцией и командой `file` (`ELF 32-bit LSB executable, ARM, EABI5`); на железе прошивка не запускалась.

### Отлаживай процесс другой архитектуры через qemu -g и LLDB

Эмулятор принимает отладчик по протоколу gdb-remote: `qemu-aarch64 -g 1234 -L /usr/aarch64-linux-gnu ./prog` останавливает программу до первой команды и ждёт подключения, а LLDB подключается командами `target create prog` и `gdb-remote 1234`; работа отладчика описана в [главе про отладку](10-debugging.md). Программу собирают с `-g`.

## Что ещё стоит знать

Для macOS и iOS Clang умеет генерировать код (`--target=arm64-apple-macos`), но линковке нужен SDK Apple, который берут с их системы; для Android нужен NDK с готовыми обёртками компилятора под каждый уровень API и libc bionic. Clang поддерживает и другие цели, например `riscv64-linux-gnu` и `wasm32`, а сборщики образов Buildroot и Yocto строят целиком и кросс-компилятор, и rootfs для встраиваемых систем.
