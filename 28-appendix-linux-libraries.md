# Приложение: справочник библиотек Linux

Справочник для выбора библиотеки из репозитория дистрибутива: какой пакет ставить, под каким именем её видит pkg-config и на каких условиях она распространяется. Контрактов здесь нет: для библиотек, которые разобраны в книге с главным контрактом, каждый раздел заканчивается ссылкой на [каталог библиотек](27-appendix-libraries.md). Все строки сверены с индексом пакетов Ubuntu 24.04 (noble); библиотеки, которых там нет, названы в последнем разделе.

## Как найти пакет и флаги

Заголовки и ссылочные файлы лежат в пакете с суффиксом `-dev`, сама библиотека в одноимённом пакете без него: для сборки ставят `-dev`, для запуска хватает библиотеки. Имя модуля pkg-config берётся из файла `.pc` внутри `-dev`-пакета и часто отличается от имени пакета, поэтому его читают из пакета, а не угадывают. Подключение через систему сборки описано в [главе про сборку](02-build.md), под другую архитектуру пакеты ставят в [главе про кросс-компиляцию](22-cross-compilation.md).

```sh
apt-cache search --names-only '^libzstd'
dpkg -L libzstd-dev | grep '\.pc$'
pkg-config --cflags --libs libzstd
```

Лицензия взята из основного раздела (`Files: *`) файла `copyright` runtime-пакета; где такого раздела нет, написано «см. `copyright` пакета». У пакета часто есть отдельные части под другими условиями (утилиты, тесты, примеры), поэтому перед поставкой проверяй файл пакета целиком. Строки про OpenSSL, libsodium и libsystemd повторяют каталог из книги намеренно: здесь пакет и лицензия, там контракт. Копилефт-лицензии (GPL, LGPL) налагают условия на код, который с ними связан: для статической линковки и для поставки бинарников это отдельный вопрос.

## Сжатие и архивы

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Универсальное сжатие deflate | `zlib1g-dev` | `zlib` | zlib | Базовая зависимость почти всего: сам модуль есть, строка лицензии в `copyright` не выделена |
| Быстрое сжатие с высоким коэффициентом | `libzstd-dev` | `libzstd` | BSD-3-clause или GPL-2 на выбор | |
| Очень быстрое сжатие | `liblz4-dev` | `liblz4` | BSD-2-clause (библиотека), GPL-2+ (утилиты) | |
| Сжатие для веба | `libbrotli-dev` | `libbrotlicommon`, `libbrotlidec`, `libbrotlienc` | MIT | Три модуля: общий, распаковка и сжатие |
| Формат xz | `liblzma-dev` | `liblzma` | общественное достояние (библиотека) | Утилиты и часть файлов под LGPL и GPL |
| Формат bzip2 | `libbz2-dev` | нет файла `.pc` | BSD-подобная | Линкуют вручную `-lbz2` |
| Чтение и запись архивов tar, zip и других | `libarchive-dev` | `libarchive` | BSD-2-clause | Один интерфейс на множество форматов |

Библиотеки сжатия в книге не разбираются, эти строки единственные.

## Криптография и TLS

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| TLS и базовая криптография | `libssl-dev` | `libssl`, `libcrypto`, `openssl` | Apache-2.0 | Самая распространённая зависимость; сравнение за постоянное время в [безопасности](19-security.md) |
| TLS с копилефтом LGPL | `libgnutls28-dev` | `gnutls` | см. `copyright` пакета | Лицензия влияет на статическую линковку |
| Высокоуровневая криптография без настройки | `libsodium-dev` | `libsodium` | ISC | Рекомендация книги: свою криптографию не писать |
| TLS для встраиваемых систем | `libmbedtls-dev` | `mbedtls`, `mbedcrypto`, `mbedx509` | Apache-2.0 или GPL-2+ на выбор | |
| Криптографические примитивы GnuPG | `libgcrypt20-dev` | `libgcrypt` | см. `copyright` пакета | Низкоуровневый набор шифров и хешей |

Разобраны в книге: OpenSSL и libsodium (сравнение секретов за постоянное время), см. [каталог](27-appendix-libraries.md) и [безопасность](19-security.md).

## Сеть и протоколы

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Асинхронный DNS | `libc-ares-dev` | `libcares` | см. `copyright` пакета | |
| HTTP/2 | `libnghttp2-dev` | `libnghttp2` | MIT (Expat) | Только протокол, без сокетов и TLS |
| Клиент SSH | `libssh2-1-dev` | `libssh2` | BSD-3-clause | |
| Захват пакетов | `libpcap-dev` | `libpcap` | см. `copyright` пакета | Файл `.pc` приходит из зависимости `libpcap0.8-dev`; есть и утилита `pcap-config` |
| Сетевой стек Linux (netlink) | `libnl-3-dev` | `libnl-3.0` | см. `copyright` пакета | |
| Веб-сокеты и HTTP-сервер | `libwebsockets-dev` | `libwebsockets` | см. `copyright` пакета | Файл смешивает несколько условий: проверь его для своей поставки |
| Встраиваемый HTTP-сервер | `libmicrohttpd-dev` | `libmicrohttpd` | LGPL-2.1+ | |
| Обмен сообщениями ZeroMQ | `libzmq3-dev` | `libzmq` | MPL-2.0 | |
| Обмен сообщениями nanomsg-next-gen | `libnng-dev` | нет файла `.pc` в пакете | MIT или Apache-2.0 на выбор | |
| Клиент Kafka | `librdkafka-dev` | `rdkafka`, `rdkafka++` | BSD-2-clause | Runtime-пакет тянет и библиотеку объектного интерфейса |
| Клиент MQTT | `libmosquitto-dev` | `libmosquitto` | EPL-2.0 или EDL-1.0 на выбор | |

Разобраны в книге: libcurl и libuv, см. [каталог](27-appendix-libraries.md) и [системный уровень](13-system-io.md).

## Форматы данных

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| XML с деревом и XPath | `libxml2-dev` | `libxml-2.0` | MIT | Один из самых частых зависимых пакетов |
| XML потоковым разбором | `libexpat1-dev` | `expat` | MIT | |
| YAML | `libyaml-dev` | `yaml-0.1` | MIT (Expat) | Для недоверенного ввода YAML не рекомендуется: см. [сериализацию](14-serialization.md) |
| JSON со счётчиком ссылок | `libjansson-dev` | `jansson` | MIT (Expat) | |

Разобраны в книге: cJSON, jsmn, protobuf-c, nanopb, libcbor и msgpack-c, см. [каталог](27-appendix-libraries.md) и [сериализацию](14-serialization.md).

## Текст и ввод в терминале

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Юникод и локали сложных языков | `libicu-dev` | `icu-uc`, `icu-i18n`, `icu-io` | MIT (данные Unicode, см. `copyright`) | Тяжёлая зависимость: данные локалей |
| Регулярные выражения (текущая версия) | `libpcre2-dev` | `libpcre2-8`, `libpcre2-16`, `libpcre2-32` | BSD-3-clause | |
| Регулярные выражения (старая версия) | `libpcre3-dev` | `libpcre` | см. `copyright` пакета | Не развивается; новые проекты берут PCRE2 |
| Юникод-строки GNU | `libunistring-dev` | нет файла `.pc` | LGPL-3+ или GPL-2+ на выбор | |
| Строка ввода с историей | `libreadline-dev` | `readline` | GPL-3+ | Копилефт: для закрытого кода берут `libedit` |
| Строка ввода с историей без GPL | `libedit-dev` | `libedit` | см. `copyright` пакета | |
| Текстовые интерфейсы терминала | `libncurses-dev` | `ncursesw`, `form`, `menu`, `panel` | MIT (X11) | Версия для широких символов называется `ncursesw` |

Разобраны в книге: `utf8proc` (нормализация и регистр), см. [каталог](27-appendix-libraries.md) и [строки](06-strings-and-buffers.md).

## Системные возможности Linux

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Журнал и сервисы systemd | `libsystemd-dev` | `libsystemd` | LGPL-2.1+ | `sd_journal_send`: [логирование](16-logging.md) |
| Фильтры системных вызовов | `libseccomp-dev` | `libseccomp` | LGPL-2.1 | |
| Привилегии процесса (capabilities) | `libcap-dev` | `libcap` | BSD-3-clause или GPL-2 на выбор | |
| Функции BSD для Linux (`strlcpy` и другие) | `libbsd-dev` | `libbsd`, `libbsd-overlay` | см. `copyright` пакета | Пока нужна на системах, где glibc старше 2.38 |
| События устройств | `libudev-dev` | `libudev` | LGPL-2.1+ | |
| Шина сообщений D-Bus | `libdbus-1-dev` | `dbus-1` | GPL-2+ или AFL-2.1 на выбор | |
| Разбор ELF | `libelf-dev` | `libelf` | см. `copyright` пакета (GPL и LGPL) | |
| Размотка стека | `libunwind-dev` | `libunwind`, `libunwind-ptrace` | MIT (Expat) | Для стеков вне отладчика |
| Вызов функций по описанию типов | `libffi-dev` | `libffi` | MIT (Expat) | |
| Топология NUMA | `libnuma-dev` | `numa` | см. `copyright` пакета | |
| Асинхронный ввод-вывод io_uring | `liburing-dev` | `liburing` | MIT или LGPL-2.1+ на выбор | |
| Чтение-копирование-обновление в пространстве пользователя | `liburcu-dev` | `liburcu` | LGPL-2.1+ | |
| Файловая система в пространстве пользователя (старая версия) | `libfuse-dev` | `fuse` | см. `copyright` пакета (GPL-2 и LGPL) | Для новых проектов берут версию 3 |
| Файловая система в пространстве пользователя (текущая версия) | `libfuse3-dev` | `fuse3` | см. `copyright` пакета (GPL-2 и LGPL) | |

Разобраны в книге: потоки POSIX, см. [каталог](27-appendix-libraries.md) и [потоки](12-concurrency.md).

## Хранение, кэш и служебное

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Встраиваемое ключ-значение LMDB | `liblmdb-dev` | `lmdb` | OpenLDAP | Отображение файла в память |
| Клиент MariaDB и MySQL | `libmariadb-dev` | `libmariadb`, `mariadb` | LGPL (клиентская библиотека; в `copyright` есть и GPL-части) | Практики работы с базой: [базы данных](17-databases.md) |
| Клиент memcached | `libmemcached-dev` | `libmemcached` | BSD-3-clause | |
| Аллокатор для многопоточной нагрузки | `libjemalloc-dev` | `jemalloc` | BSD-2-clause | Подмена `malloc`: [память](04-memory-ownership.md) |
| Быстрый хеш | `libxxhash-dev` | `libxxhash` | BSD-2-clause (библиотека), GPL-2+ (утилита) | Не криптографический |
| Идентификаторы UUID | `uuid-dev` | `uuid` | см. `copyright` пакета (смесь лицензий) | Пакет `uuid-dev` приносит `libuuid` |

Разобраны в книге: PostgreSQL (libpq), Redis (hiredis) и SQLite, mimalloc, см. [каталог](27-appendix-libraries.md).

## Графика, изображения и медиа

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Окно, ввод и звук | `libsdl2-dev` | `sdl2` | zlib | Тулкит для игр и инструментов |
| Растеризация шрифтов | `libfreetype-dev` | `freetype2` | FreeType или GPL-2+ | |
| Формирование текста | `libharfbuzz-dev` | `harfbuzz` | MIT | Для сложных систем письма |
| Двухмерная векторная графика | `libcairo2-dev` | `cairo` | см. `copyright` пакета | |
| Формат PNG | `libpng-dev` | `libpng`, `libpng16` | libpng | |
| Формат JPEG | `libjpeg-turbo8-dev` | `libjpeg` | смесь BSD и zlib, см. `copyright` пакета | Точные условия проверяй до поставки |
| Кодеки FFmpeg | `libavcodec-dev` | `libavcodec` | LGPL-2.1+ | Часть кодеков под GPL |

Графические библиотеки в книге не разбираются; интерфейсы без тулкитов описаны в [архитектуре интерфейса](24-ui-architecture.md).

## Математика

| Задача | Пакет | Модуль pkg-config | Лицензия | Замечание |
| --- | --- | --- | --- | --- |
| Целые и рациональные произвольной точности | `libgmp-dev` | `gmp`, `gmpxx` | LGPL-3+ или GPL-2+ на выбор | |
| Вещественные произвольной точности | `libmpfr-dev` | `mpfr` | см. `copyright` пакета | |
| Научные вычисления | `libgsl-dev` | `gsl` | см. `copyright` пакета (GPL) | Копилефт: для закрытого кода нужна отдельная проверка |

Разобраны в книге: арифметика с проверкой переполнения, см. [целые типы](03-types-and-ub.md).

## Чего в репозитории Ubuntu 24.04 нет

В индексе noble нет пакетов: `yyjson`, `linenoise`, `libbacktrace`, `wasm3`, WAMR, wasmtime, `tomlc99`, `flatcc`, `tinycbor`, `cmp`, `zlog`, `argtable3`, `greatest`, `munit`. Их берут исходниками в дерево проекта или из менеджера пакетов проекта с фиксацией версии и контрольной суммы ([зависимости в сборке](02-build.md)).
