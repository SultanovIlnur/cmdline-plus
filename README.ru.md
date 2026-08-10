# cmdline-plus

Простой парсер командной строки для C++.

*[English version](README.md)*


Это форк проекта [tanakh/cmdline](https://github.com/tanakh/cmdline) за авторством Hideyuki Tanaka, который
больше не поддерживается. В этом форке есть ряд улучшений и фиксов по сравнению с оригинальным проектом.

Текст ниже представляет собой модифицированную и переведенную с японского статью с [исходного сайта] (https://tanakh.hatenablog.com/entries/2009/10/28).

## О библиотеке

Cmdline - это библиотека, которая помогает разбирать аргументы командной строки. Для этой же задачи есть и
другие библиотеки - стандартный сишный `getopt`, гугловский `gflags`, - но cmdline является более простым проектом без необходимости ставить лишние либы. `getopt` неудобен, и текст usage приходится писать самому. `gflags` тоже не очень удобен в работе, так как он требует установки библиотеки и линковки с ней. cmdline же состоит из одного заголовочного файла, с которым удобнее работать, просто скопировав его.

## Установка

Скопируйте [`cmdline.h`](cmdline.h) рядом со своими исходниками (или укажите на этот каталог
через `-I`) и подключите его. Зависимостей, кроме стандартной библиотеки, нет. Линковать ничего не нужно.

```cpp
#include "cmdline.h"
```

Сборка примеров из репозитория:

```sh
g++ -std=c++11 -Wall -o test  test.cpp
g++ -std=c++11 -Wall -o test2 test2.cpp
```

Поддерживается стандарт C++98 и новее.

## Использование

```cpp
// подключаем cmdline.h
#include "cmdline.h"

int main(int argc, char *argv[])
{
  // создаём парсер
  cmdline::parser a;

  // добавляем переменную указанного типа.
  // 1-й аргумент - полное имя
  // 2-й аргумент - короткое имя (если указать '\0', короткого имени не будет)
  // 3-й аргумент - описание
  // 4-й аргумент - обязательность (необязательный, по умолчанию false)
  // 5-й аргумент - значение по умолчанию (необязательный, используется при mandatory = false)
  a.add<std::string>("host", 'h', "host name", true, "");

  // 6-й аргумент - дополнительное ограничение.
  // Здесь номер порта обязан быть от 1 до 65535, это обеспечивает cmdline::range().
  a.add<int>("port", 'p', "port number", false, 80, cmdline::range(1, 65535));

  // cmdline::oneof() ограничивает набор допустимых значений.
  a.add<std::string>("type", 't', "protocol type", false, "http",
                     cmdline::oneof<std::string>("http", "https", "ssh", "ftp"));

  // Булевы флаги тоже можно определять.
  // Для этого вызываем add без параметра-типа.
  a.add("gzip", '\0', "gzip when transfer");

  // Запускаем парсер.
  // Управление возвращается только если аргументы командной строки корректны.
  // Если они некорректны, парсер печатает сообщения об ошибках и завершает программу.
  // Если задан флаг помощи ('--help' или '-?'), парсер печатает usage и завершает программу.
  a.parse_check(argc, argv);

  // используем значения флагов
  std::cout << a.get<std::string>("type") << "://"
            << a.get<std::string>("host") << ":"
            << a.get<int>("port") << std::endl;

  // булевы флаги проверяются методом exist()
  if (a.exist("gzip")) std::cout << "gzip" << std::endl;
}
```

### Пример запуска

```
$ ./test
usage: ./test --host=string [options] ...
options:
  -h, --host    host name (string)
  -p, --port    port number (int [=80])
  -t, --type    protocol type (string [=http])
      --gzip    gzip when transfer
  -?, --help    print this message

$ ./test --host=github.com
http://github.com:80

$ ./test --host=github.com -t ftp
ftp://github.com:80

$ ./test --host=github.com -p 4545
http://github.com:4545

$ ./test --host=github.com --gzip
http://github.com:80
gzip

$ ./test --host=github.com -t ttp
option value is invalid: --type=ttp
usage: ./test --host=string [options] ...
options:
  -h, --host    host name (string)
  ...

$ ./test --host=github.com -p 100000
option value is invalid: --port=100000
usage: ./test --host=string [options] ...
options:
  -h, --host    host name (string)
  ...
```

## Дополнительные возможности

### Остальные аргументы

Аргументы, не распознанные как опции, доступны через `rest()` - метод возвращает вектор строк.
Обычно это имена файлов и тому подобное.

```cpp
for (size_t i = 0; i < a.rest().size(); i++)
  std::cout << a.rest()[i] << std::endl;
```

### Footer

`footer()` добавляет текст в строку `usage:`.

```cpp
a.footer("filename ...");
```

Результат:

```
$ ./test
usage: ./test --host=string [options] ... filename ...
options:
  -h, --host    host name (string)
  ...
```

### Имя программы

Парсер показывает имя программы в тексте usage. По умолчанию оно берётся из `argv[0]`;
`set_program_name()` позволяет задать любую строку.

## Ручная обработка флагов

`parse_check()` разбирает аргументы командной строки и сам обрабатывает и ошибки, и флаг помощи.
Эту работу можно проделать самостоятельно: `bool parse()` разбирает аргументы и возвращает,
корректны ли они. Проверяйте результат и делайте с ним что нужно.

```cpp
cmdline::parser p;
p.add("hoge", 'h', "hoge flag with no value");
p.add<int>("moge", 'm', "moge flag with int value", false, 123);
p.add("help", 0, "print help");

if (!p.parse(argc, argv) || p.exist("help")){
  std::cout << p.error_full() << p.usage();
  return 0;
}

if (p.exist("hoge"))
  std::cout << "there is hoge flag" << std::endl;

std::cout << "moge value: " << p.get<int>("moge") << std::endl;
```

(Подробнее - в [`test2.cpp`](test2.cpp).)

## Пользовательские reader'ы

По умолчанию для чтения значения флага применяется `lexical_cast` (точнее, самописный его
аналог). Поэтому `add<int>(...)` читает аргумент командной строки как десятичное целое, а при
некорректном значении `parse` завершается неудачей. Это поведение настраиваемое: вместо
`lexical_cast` можно передать произвольный функциональный объект - так задаются, например,
чтение в шестнадцатеричном виде или значение из определённого диапазона. Для удобства в cmdline
уже заготовлены reader'ы для значения с диапазоном, выбора из набора и тому подобного. Поведение
по умолчанию доступно как `cmdline::default_reader`.

```cpp
p.add<int>("hoge", 'h', "int value (100 - 999)", false, 100, cmdline::range(100, 999));
p.add<std::string>("moge", 'm', "one of abc, def, ghi", false, "abc",
                   cmdline::oneof<std::string>("abc", "def", "ghi"));
```

## Справочник API

### `void parser::add(const std::string &name, char short_name=0, const std::string &desc="")`

Определяет флаг без значения. Аргументы по порядку: полное имя, короткое имя, описание. Если в
качестве короткого имени передать `0`, короткой формы у флага не будет.

### `template <class T> void parser::add(const std::string &name, char short_name=0, const std::string &desc="", bool need=true, const T def=T())`

Определяет флаг со значением. Обязательны полное имя, короткое имя и описание - плюс тип,
задаваемый параметром шаблона. Дальше идут два необязательных аргумента: обязателен ли сам флаг
(если да, то при его отсутствии в командной строке `parse` завершится ошибкой) и значение по
умолчанию, которое используется, когда значение не задано.

### `template <class T, class F> void parser::add(const std::string &name, char short_name=0, const std::string &desc="", bool need=true, const T def=T(), F reader=F())`

Тоже определяет флаг со значением, но позволяет задать для него собственный reader. Метод выше
использует в качестве reader'а по умолчанию `lexical_cast`; сюда же можно передать любую функцию
или функциональный объект, преобразующий `std::string` в тип `T`. Если эта функция бросает
исключение, значение считается некорректным и `parse()` завершается неудачей.

### `bool parser::parse(int argc, const char * const argv[])`

Разбирает аргументы командной строки - аргументы `main` передаются как есть. Возвращает `false`
при неудаче и `true` при успехе. В случае неудачи конкретное сообщение доступно через `error()`.
Есть также перегрузки, принимающие одну `std::string` или `std::vector<std::string>`.

### `void parser::parse_check(int argc, char *argv[])`

Разбирает аргументы и сам обрабатывает ошибки и флаг помощи. Если флаг `help` не был определён,
регистрирует его как `-?, --help`. При ошибке разбора или запросе помощи печатает сообщение и
завершает программу; возврат происходит, только если аргументы корректны.

### `bool parser::exist(const std::string &name) const`

Возвращает, присутствовал ли указанный флаг в командной строке. Использовать после `parse()`.

### `template <class T> const T &parser::get(const std::string &name) const`

Возвращает значение указанного флага. Использовать после `parse()`. Если имя флага некорректно
(он не был добавлен через `add`) или тип не совпадает с указанным при `add`, бросается исключение
`cmdline::cmdline_error`.

### `const std::vector<std::string> &parser::rest() const`

Возвращает аргументы, не распознанные как опции.

### `std::string parser::error() const`

Возвращает первое сообщение об ошибке.

### `std::string parser::error_full() const`

Возвращает все сообщения об ошибках.

### `std::string parser::usage() const`

Возвращает текст usage.

### `void parser::footer(const std::string &f)`

Добавляет строку после строки `usage:` в тексте, который возвращает `usage()`.

### `void parser::set_program_name(const std::string &name)`

Задаёт имя программы, отображаемое в строке `usage:`. Если метод не вызывать, используется
`argv[0]`.

## Переносимость

В рамках этого форка были внесены доработки для совместимости с MSVC. Имена типов в тексте usage приводятся к читаемому виду через `<cxxabi.h>`. Это заголовок, который поставляется с либами libstdc++ и libc++, но отсутствует в MSVC. Поэтому подключенение защищено условной компиляцией: там, где заголовка нет, используется сырое имя от `typeid`. Нужно определить значение `CMDLINE_NO_CXXABI`, чтобы принудительно включить это.

## Лицензия

BSD 3-Clause, см. [LICENSE](LICENSE).

Оригинальный код Copyright (c) 2009 Hideyuki Tanaka, изменения Copyright (c) 2026 Ilnur
Sultanov.
