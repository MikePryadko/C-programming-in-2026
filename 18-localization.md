# Локализация

## Каталоги сообщений

### Оборачивай тексты для людей в _() и подключай каталог в начале main

Программа хранит исходные строки на одном языке, а перевод подставляется из каталога `.mo` в момент вывода, поэтому добавить язык можно без пересборки кода. Библиотека gettext входит в glibc: `bindtextdomain` задаёт каталог установки, `textdomain` выбирает домен, `gettext` ищет перевод строки и при отсутствии возвращает её саму.

Код использует три макроса. `_(s)` вызывает `gettext`, `N_(s)` только помечает строку для извлечения (нужен в статических таблицах, где вызывать функцию нельзя), `C_(ctx, s)` ищет перевод с контекстом: каталог хранит ключ `контекст`, `\004`, `строка`, а при отсутствии перевода помощник возвращает исходную строку. Множественные формы даёт `ngettext`.

```c
/* i18n.h */
#ifndef I18N_H
#define I18N_H
#include <libintl.h>
#include <locale.h>

#ifndef LOCALEDIR
#define LOCALEDIR "/usr/local/share/locale"
#endif

#define _(s) gettext(s)
#define N_(s) s
#define C_(ctx, s) i18n_ctx(ctx "\004" s, s)

static inline const char *i18n_ctx(const char *key, const char *msg) {
    const char *t = gettext(key);
    return t == key ? msg : t;
}
#endif
```

Программа вызывает `setlocale` один раз в начале `main`, затем подключает домен. Комментарий `TRANSLATORS` попадает в каталог и виден переводчику.

```c
/* main.c */
#include "i18n.h"
#include <stdio.h>

static const char *const modes[] = {N_("read"), N_("write")};

static void files(unsigned long n) {
    printf(ngettext("%lu file\n", "%lu files\n", n), n);
}

int main(void) {
    setlocale(LC_ALL, "");
    setlocale(LC_NUMERIC, "C");
    bindtextdomain("app", LOCALEDIR);
    textdomain("app");
    /* TRANSLATORS: %s - имя файла */
    printf(_("Cannot open %s\n"), "a.txt"); /* вывод: Не удаётся открыть a.txt */
    files(1); /* вывод: 1 файл */
    files(2); /* вывод: 2 файла */
    files(5); /* вывод: 5 файлов */
    files(21); /* вывод: 21 файл */
    puts(C_("menu", "Open")); /* вывод: Открыть */
    puts(C_("state", "Open")); /* вывод: Открыт */
    puts(_(modes[1])); /* вывод: запись */
    return 0;
}
```

Вывод в комментариях дан для `LC_ALL=ru_RU.UTF-8` с установленным `ru.mo`; в локали `C` печатаются исходные строки. Перевод живёт в `.po`: `msgctxt` задаёт контекст, `msgstr[N]` хранит формы множественного числа, строка `Plural-Forms` описывает правило выбора формы по числу (для русского три формы).

```text
# file: ru.po
msgid ""
msgstr ""
"Language: ru\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Plural-Forms: nplurals=3; plural=(n%10==1 && n%100!=11 ? 0 : n%10>=2 && n%10<=4 && (n%100<10 || n%100>=20) ? 1 : 2);\n"

#. TRANSLATORS: %s - имя файла
#, c-format
msgid "Cannot open %s\n"
msgstr "Не удаётся открыть %s\n"

#, c-format
msgid "%lu file\n"
msgid_plural "%lu files\n"
msgstr[0] "%lu файл\n"
msgstr[1] "%lu файла\n"
msgstr[2] "%lu файлов\n"

msgctxt "menu"
msgid "Open"
msgstr "Открыть"

msgctxt "state"
msgid "Open"
msgstr "Открыт"

msgid "read"
msgstr "чтение"

msgid "write"
msgstr "запись"
```

Конвейер: `xgettext` собирает шаблон `.pot` из исходников, `msginit` создаёт `.po` для языка (с готовым `Plural-Forms`), при изменении кода `msgmerge` переносит старые переводы в новый шаблон, `msgfmt` компилирует `.po` в бинарный `.mo`. Ключи `-k` сообщают `xgettext` о наших макросах: `C_:1c,2` означает «первый аргумент контекст, второй строка».

```sh
xgettext -k_ -kN_ -kC_:1c,2 --add-comments=TRANSLATORS --from-code=UTF-8 -o app.pot main.c
msginit --no-translator -l ru_RU.UTF-8 -i app.pot -o ru.po
msgmerge --update ru.po app.pot
msgfmt --check-format -o ru/LC_MESSAGES/app.mo ru.po
```

Каталог должен лечь в `LOCALEDIR/ru/LC_MESSAGES/app.mo`, где `app` совпадает с именем домена. Общую установку с префиксом описывает [сборка проекта](02-build.md), а `msgfmt` и `LOCALEDIR` разобраны здесь: путь `LOCALEDIR` передают макросом из префикса установки, `.mo` собирают явной командой `msgfmt` (хелпер `gettext_process_po_files` называет `.mo` по имени `.po`-файла, а не по домену).

```text
# file: CMakeLists.txt
cmake_minimum_required(VERSION 3.22)
project(hello C)
find_package(Intl REQUIRED)
find_package(Gettext REQUIRED)
add_executable(hello main.c)
set_target_properties(hello PROPERTIES C_STANDARD 23 C_EXTENSIONS OFF)
target_compile_options(hello PRIVATE -Wall -Wextra -Wpedantic -Wformat=2 -Werror)
target_compile_definitions(hello PRIVATE LOCALEDIR="${CMAKE_INSTALL_PREFIX}/share/locale")
target_link_libraries(hello PRIVATE Intl::Intl)

set(MO ${CMAKE_BINARY_DIR}/ru/LC_MESSAGES/app.mo)
add_custom_command(OUTPUT ${MO}
  COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/ru/LC_MESSAGES
  COMMAND ${GETTEXT_MSGFMT_EXECUTABLE} --check-format -o ${MO} ${CMAKE_SOURCE_DIR}/ru.po
  DEPENDS ru.po)
add_custom_target(catalogs ALL DEPENDS ${MO})
install(TARGETS hello)
install(FILES ${MO} DESTINATION share/locale/ru/LC_MESSAGES)
```

#### Подводные камни

- Локаль не сгенерирована в системе (`locale -a` её не показывает): `setlocale(LC_ALL, "")` молча возвращает `NULL` и остаётся локаль `C`, перевода нет. Проверяй `locale -a` и результат `setlocale`, в контейнере ставь пакет `locales`.
- В локали `C` перевод не включается никогда, даже при `LANGUAGE`: это типичная причина «в контейнере не переводится» (подробнее в разделе «Командная строка» этой главы).
- Нет вызова `textdomain` или домен в коде не совпадает с именем `.mo`: `gettext` возвращает исходную строку без ошибок. Каталог ищется строго по `LOCALEDIR/язык/LC_MESSAGES/домен.mo`.
- Строка в статической таблице без `N_()` не попадёт в шаблон, а `_()` в инициализаторе не скомпилируется; помечай `N_()` в таблице и вызывай `_()` при выводе.
- Вне glibc функции gettext находятся в отдельной библиотеке `libintl`; система сборки находит её сама (`find_package(Intl)` на glibc сообщает, что она встроена в libc). Тот же пример собирается musl-gcc без `libintl` и переводится (проверено на этом примере).

### Пиши цельные фразы и клади форму числа в каталог

Переводчик видит строку отдельно от кода, поэтому склеенная из кусков фраза не переводится верно: порядок слов и падежи в других языках другие. Любое сообщение, зависящее от числа, идёт через `ngettext`, а не через `if (n == 1)`.

```c
/* Ошибка: фраза собрана из частей, порядок слов и форма числа заданы кодом */
#include <libintl.h>
#include <stdio.h>

void report(int n) {
    printf("%s %d %s\n", gettext("Found"), n, n == 1 ? gettext("file") : gettext("files"));
}
```

Правильная форма из примера выше: одна фраза `"%lu file\n"` с формами в `msgstr[N]`. Если порядок слов в переводе должен отличаться, его меняет переводчик в `msgstr` позиционными спецификаторами (`%2$s %1$s`): в исходнике остаются обычные `%s`, потому что GCC под `-Wpedantic -Wformat=2` отвергает `%1$s` в самом коде (Clang принимает). `msgfmt --check-format` сверяет набор спецификаторов оригинала и перевода.

#### Подводные камни

- Число в `ngettext` принимается как `unsigned long`: `int` под `-Wconversion` не проходит, приводи явно `(unsigned long)n` и печатай `%lu`.
- Русскому нужны три формы (1, 2-4, 5-20 и так далее), и «файл» с числом 21 снова первая форма; не выводи формы кодом, а бери `Plural-Forms` из `msginit` и не правь его.
- Перед сообщением с неочевидным смыслом (`%s` - имя файла или имя пользователя, короткое «Open») ставь комментарий `TRANSLATORS` и контекст `C_()`: одно слово переводится по-разному в зависимости от роли.
- Не переводи: строки лога (их читают `grep` и разбор, текст должен быть одинаков в любой локали), поля протокола и ключи конфигурации, имена опций, коды возврата, любой вывод, который читает скрипт. Переводится только то, что читает человек.

## Числа и правила локали

### Вызывай setlocale один раз, держи LC_NUMERIC в "C", числа для людей форматируй на узком участке

Программа с gettext вызывает `setlocale(LC_ALL, "")`, но тогда `printf` и `strtod` начнут работать с запятой в окружении `ru_RU`. Ловушка `LC_NUMERIC` описана в разделе про локаль в [строках и буферах](06-strings-and-buffers.md), здесь правило для программ с gettext, единое для всей книги: `setlocale(LC_ALL, "")` один раз в начале `main`, до потоков, сразу за ним `setlocale(LC_NUMERIC, "C")`. Числа для пользователя форматируют отдельно: объект локали создают из копии текущей, а числовую категорию берут из окружения, и включают его на одну операцию в текущем потоке.

```c
/* numfmt.h */
#ifndef NUMFMT_H
#define NUMFMT_H
#include <stddef.h>

/* num_ui - печатает v с двумя знаками в форме числовой категории окружения.
 * Возврат: 0 или -1; глобальная локаль процесса не меняется. */
int num_ui(char *buf, size_t cap, double v);
#endif
```

```c
/* numfmt.c */
#define _GNU_SOURCE
#include "numfmt.h"
#include <locale.h>
#include <stdio.h>

int num_ui(char *buf, size_t cap, double v) {
    locale_t ui = newlocale(LC_NUMERIC_MASK, "", duplocale(LC_GLOBAL_LOCALE));
    if (!ui) return -1;
    locale_t old = uselocale(ui);
    int n = snprintf(buf, cap, "%.2f", v);
    uselocale(old);
    freelocale(ui);
    return n < 0 ? -1 : 0;
}
```

```c
/* main.c */
#include "numfmt.h"
#include <locale.h>
#include <stdio.h>

int main(void) {
    setlocale(LC_ALL, "");
    setlocale(LC_NUMERIC, "C");
    char ui[32];
    printf("%.2f\n", 3.14159); /* вывод: 3.14 */
    if (num_ui(ui, sizeof ui, 3.14159)) return 1;
    printf("%s\n", ui); /* вывод: 3,14 */
    printf("%.2f\n", 3.14159); /* вывод: 3.14 */
    return 0;
}
```

Вывод показан для окружения `LC_ALL=ru_RU.UTF-8`; в локали `C` обе строки покажут точку. База `duplocale(LC_GLOBAL_LOCALE)` оставляет остальные категории (кодировку, `LC_CTYPE`, `LC_MESSAGES`) такими же, как у процесса.

#### Подводные камни

- Если базой взять `(locale_t)0`, остальные категории на участке станут «C»: `nl_langinfo(CODESET)` вернул `ANSI_X3.4-1968` вместо `UTF-8`, а обработка многобайтных символов и сообщения изменились бы. Для чисел берут копию глобальной локали.
- `%'d` не используется: флаг `'` берёт разделитель из того же `LC_NUMERIC`, поэтому после `setlocale(LC_NUMERIC, "C")` он даёт формат C, а GCC под `-Wpedantic -Wformat=2` его не принимает. Группировку разрядов делают по `localeconv()` внутри участка с `uselocale` либо не показывают.
- Числа в протоколе, файлах и логе форматируют в локали C; локаль окружения нужна только тому, что читает человек.

## Безопасность каталогов

### Считай каталог перевода внешним вводом и проверяй форматы при сборке

Формат-строка `printf(_("Cannot open %s\n"), name)` берётся из `.mo`, то есть из файла, а компилятор видит только литерал. Подменённый перевод с лишним спецификатором (`%n`, второй `%s`) превращает безобидный вызов в чтение чужой памяти или запись. Под `-Wformat=2 -Werror` вызов `printf(_("..."), x)` проходит: объявление `gettext` в `libintl.h` несёт `format_arg`, и компилятор проверяет литерал-оригинал (Clang 19, 20 и GCC 14), но перевод он проверить не может.

```text
# file: bad.po
msgid ""
msgstr ""
"Content-Type: text/plain; charset=UTF-8\n"

#, c-format
msgid "Cannot open %s\n"
msgstr "Не удаётся открыть %s %n\n"
```

Защита в сборке: `msgfmt --check-format` сверяет число и типы спецификаторов оригинала и перевода и обрывает сборку.

```sh
msgfmt --check-format -o bad.mo bad.po
```

```text
bad.po:7: number of format specifications in 'msgid' and 'msgstr' does not match
msgfmt: found 1 fatal error
```

#### Подводные камни

- `msgfmt` без `--check-format` формат не проверяет: ключ ставь в рецепт сборки, как в `CMakeLists.txt` выше, и не принимай `.mo`, собранные не у тебя.
- Каталог читается из каталога `bindtextdomain`: он должен принадлежать root и быть не записываемым для пользователей. Переменная `TEXTDOMAINDIR` на выбор каталога в библиотеке не влияет (проверено), но `LANGUAGE` и `LC_ALL` остаются под контролем вызывающего даже в setuid-программе (`LANGUAGE=en` в setuid-копии вернул исходную строку): язык, выбранный пользователем, не считай признаком чего-либо в проверках безопасности.
- Свой макрос с контекстом, возвращающий строку формата, объявляй с атрибутом `format_arg` (расширение GCC/Clang), иначе `printf(C_(...), x)` не пройдёт `-Wformat=2`: `__attribute__((format_arg(2)))` для второго аргумента.

## Зависящее от локали

### Не полагайся на strcmp, strftime и strerror там, где нужен стабильный результат

После `setlocale(LC_ALL, "")` часть функций меняет поведение: `strcoll` сравнивает по правилам языка, `strftime` с `%x` и `%c` даёт локальный формат даты, `strerror` возвращает перевод (по категории `LC_MESSAGES`, если у системы есть каталог libc). Для людей это правильно, для ключей, сортировки в протоколе и лога нет: там используют `strcmp` по байтам, формат даты с явными полями и стабильные тексты.

```c
#define _GNU_SOURCE
#include <errno.h>
#include <locale.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    if (!setlocale(LC_ALL, "ru_RU.UTF-8")) return 1;
    printf("%d\n", strcmp("a", "B") > 0); /* вывод: 1 */
    printf("%d\n", strcoll("a", "B") < 0); /* вывод: 1 */
    puts(strerrordesc_np(ENOENT)); /* вывод: No such file or directory */
    return 0;
}
```

`strerrordesc_np` (GNU, glibc 2.32 и новее) возвращает английский текст независимо от локали, `strerrorname_np` имя константы (`ENOENT`): для лога берут их, а не `strerror`. Для человека (сообщение об ошибке в интерфейсе) остаётся `strerror`, которая следует локали.

#### Подводные камни

- `%x` и `%c` в `strftime` на GCC 14 под `-Wformat=2` дают предупреждение `-Wformat-y2k` (год в некоторых локалях двузначный), которое под `-Werror` ломает сборку; Clang молчит. Либо `-Wno-format-y2k` на файл, либо явные `%d.%m.%Y` там, где порядок не должен зависеть от локали.
- Регистр: `toupper` работает по байтам и ломает многобайтный UTF-8 ([строки и буферы](06-strings-and-buffers.md)); для текста на разных языках нужна библиотека вроде ICU или utf8proc.
- Порядок слов, плюрализация и форматы дат различаются глубже, чем видит `setlocale`: форму числа бери из `ngettext`, а не из кода.

## Командная строка

### Переводи справку и ошибки, но не имена опций, коды возврата и вывод для скриптов

Справку читает человек, а имена опций и коды возврата читают скрипты: первое переводится, второе обязано быть одинаковым во всех локалях. Тексты в таблице опций помечай `N_()` и переводи при выводе; имена, короткие ключи и коды остаются в исходнике как есть (таблица опций - в [аргументах и конфигурации](15-cli-and-config.md)).

```c
/* help.h */
#ifndef HELP_H
#define HELP_H

typedef struct { const char *name; char key; const char *help; } opt_desc;

extern const opt_desc opts[];
void help_line(const opt_desc *o);
#endif
```

```c
/* help.c */
#include "help.h"
#include <libintl.h>
#include <stdio.h>

#define _(s) gettext(s)
#define N_(s) s

const opt_desc opts[] = {
    {"verbose", 'v', N_("print more details")},
    {"output", 'o', N_("write the result to FILE")},
};

void help_line(const opt_desc *o) {
    printf("  -%c, --%-8s %s\n", o->key, o->name, _(o->help));
}
```

```c
/* main.c */
#include "help.h"
#include <libintl.h>
#include <locale.h>
#include <stdio.h>

#ifndef LOCALEDIR
#define LOCALEDIR "/usr/local/share/locale"
#endif
enum { EXIT_USAGE = 2 };

int main(int argc, char **argv) {
    setlocale(LC_ALL, "");
    setlocale(LC_NUMERIC, "C");
    bindtextdomain("app", LOCALEDIR);
    textdomain("app");
    if (argc > 1) {
        fprintf(stderr, gettext("unknown option: %s\n"), argv[1]);
        return EXIT_USAGE;
    }
    puts(gettext("Usage: tool [options] FILE"));  /* вывод: Использование: tool [параметры] ФАЙЛ */
    help_line(&opts[0]); /* вывод:   -v, --verbose  печатать подробности */
    help_line(&opts[1]); /* вывод:   -o, --output   записать результат в ФАЙЛ */
    return 0;
}
```

Вывод дан для `LC_ALL=ru_RU.UTF-8` с установленным `ru.mo`.

```text
# file: ru.po
msgid ""
msgstr ""
"Language: ru\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Plural-Forms: nplurals=3; plural=(n%10==1 && n%100!=11 ? 0 : n%10>=2 && n%10<=4 && (n%100<10 || n%100>=20) ? 1 : 2);\n"

msgid "Usage: tool [options] FILE"
msgstr "Использование: tool [параметры] ФАЙЛ"

msgid "print more details"
msgstr "печатать подробности"

msgid "write the result to FILE"
msgstr "записать результат в ФАЙЛ"

#, c-format
msgid "unknown option: %s\n"
msgstr "неизвестный параметр: %s\n"
```

#### Подводные камни

- `LANGUAGE` не действует в локали `C` и в `C.UTF-8` (проверено): в контейнере без `LANG` перевода нет, пока не задана сгенерированная локаль (`LC_ALL=ru_RU.UTF-8`). Для `LANGUAGE` нужна не-C локаль плюс установленный перевод.
- Выравнивание колонок под переведённым текстом считай по ширине отображения, а не по байтам и не по `strlen`: кириллица занимает два байта на символ, иероглифы две ячейки; нужен `wcwidth` ([техники текстового редактора](25-text-editing.md)). В примере выровнены только имена опций (ASCII), перевод стоит последним.
- Сообщения об ошибках для скрипта (`--json`, машинный вывод) не переводи; код возврата `EXIT_USAGE` одинаков во всех локалях.

## Графический интерфейс

### Храни идентификатор сообщения в модели и переводи при построении view

Тулкиты приносят свой слой: GTK использует gettext, Qt свой `tr`, SDL перевода не даёт. Для собственного интерфейса ([архитектура интерфейса](24-ui-architecture.md)) достаточно таблицы строк по перечислению идентификаторов на каждый язык; модель хранит идентификатор (и числа), а view превращает его в текст при построении, поэтому смена языка сводится к перестройке view, а сохранённых в виджетах строк нет. Множественная форма в такой таблице выбирается функцией от числа, своей для каждого языка.

```c
/* ui_text.h */
#ifndef UI_TEXT_H
#define UI_TEXT_H

typedef enum { LANG_EN, LANG_RU, LANG_COUNT } lang;
typedef enum { MSG_SAVE, MSG_FILES, MSG_COUNT } msg_id;

/* ui_text - текст сообщения id на языке l; n выбирает форму множественного числа. */
const char *ui_text(lang l, msg_id id, unsigned long n);
#endif
```

```c
/* ui_text.c */
#include "ui_text.h"

typedef struct { const char *form[3]; } entry;

static const entry table[LANG_COUNT][MSG_COUNT] = {
    [LANG_EN] = {[MSG_SAVE] = {{"Save"}}, [MSG_FILES] = {{"file", "files"}}},
    [LANG_RU] = {[MSG_SAVE] = {{"Сохранить"}}, [MSG_FILES] = {{"файл", "файла", "файлов"}}},
};

static unsigned form_en(unsigned long n) { return n != 1; }

static unsigned form_ru(unsigned long n) {
    unsigned long m10 = n % 10, m100 = n % 100;
    if (m10 == 1 && m100 != 11) return 0;
    if (m10 >= 2 && m10 <= 4 && (m100 < 10 || m100 >= 20)) return 1;
    return 2;
}

const char *ui_text(lang l, msg_id id, unsigned long n) {
    unsigned f = l == LANG_RU ? form_ru(n) : form_en(n);
    const entry *e = &table[l][id];
    return e->form[f] ? e->form[f] : e->form[0];
}
```

```c
/* main.c */
#include "ui_text.h"
#include <stdio.h>

typedef struct { lang l; unsigned long files; } model;

static void view(const model *m) {
    printf("%s: %lu %s\n", ui_text(m->l, MSG_SAVE, 0), m->files, ui_text(m->l, MSG_FILES, m->files));
}

int main(void) {
    model m = {LANG_RU, 21};
    view(&m); /* вывод: Сохранить: 21 файл */
    m.files = 22;
    view(&m); /* вывод: Сохранить: 22 файла */
    m.files = 25;
    view(&m); /* вывод: Сохранить: 25 файлов */
    m.l = LANG_EN;
    view(&m); /* вывод: Save: 25 files */
    return 0;
}
```

Здесь собственная таблица, без gettext: язык выбирает модель, а не локаль. Правило множественной формы держи отдельной функцией и проверяй тестом по значениям вокруг границ (1, 2, 4, 5, 11, 12, 21, 22, 25). Число стоит рядом со словом в `printf`, так что порядок «число слово» задан кодом: для языков, где он другой, форму кладут в строку с `%lu`.

Если интерфейс построен на gettext, язык можно сменить на лету: после `setenv("LANGUAGE", ...)` вызови `setlocale(LC_MESSAGES, "")`, иначе `gettext` отдаёт закэшированный прежний результат (проверено); `LC_NUMERIC` при этом остаётся в `"C"`, а повторный `setlocale(LC_ALL, "")` вернул бы запятую. Это работает, только если локаль окружения не `C`.

#### Подводные камни

- Строка, уже скопированная в виджет или буфер, остаётся на старом языке; поэтому в модели хранится идентификатор, а переводится при построении view, и смена языка вызывает полную перестройку.
- Перевод длиннее оригинала: не фиксируй ширину элементов по английскому тексту.
- Языки с письмом справа налево (RTL) и шрифты с нужными начертаниями решают тулкит и платформа; при собственной отрисовке про них помнят заранее.

## Коротко

### bind_textdomain_codeset

`bind_textdomain_codeset("app", "UTF-8")` задаёт кодировку, в которой `gettext` возвращает строки, независимо от кодировки локали; вызывают после `bindtextdomain`, когда интерфейс принимает только UTF-8. Функция возвращает установленную кодировку.

### uselocale

`uselocale(loc)` включает объект локали только для текущего потока, `uselocale(LC_GLOBAL_LOCALE)` возвращает глобальную, а результат вызова позволяет восстановить прежнюю. Остальные потоки не затронуты, в отличие от `setlocale`, который меняет процесс целиком и не потокобезопасен.

## Что ещё стоит знать

Для сложных правил (сортировка по языку, форматы дат и чисел, нормализация) подключают ICU, у которой свой каталог форматов независимо от `setlocale`. Псевдолокализацию (проверка кода на жёстко заданную ширину и пропущенные строки) получают командами `msgen` и `msgfilter` в UTF-8 локали: спецификаторы формата они сохраняют, и `msgfmt --check-format` проходит. Для Meson есть модуль `i18n.gettext`, которому нужны файлы `LINGUAS` и `POTFILES`.
