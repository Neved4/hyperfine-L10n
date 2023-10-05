# hyperfine
[![CICD](https://github.com/sharkdp/hyperfine/actions/workflows/CICD.yml/badge.svg)](https://github.com/sharkdp/hyperfine/actions/workflows/CICD.yml)
[![Информация о версии](https://img.shields.io/crates/v/hyperfine.svg)](https://crates.io/crates/hyperfine)
[中文](https://github.com/chinanf-boy/hyperfine-zh) |
[한국어](https://github.com/Neved4/hyperfine-L10n/blob/main/hyperfine-ko.md) |
[Русский](https://github.com/Neved4/hyperfine-L10n/blob/main/hyperfine-ru.md)

Инструмент для бенчмаркинга командной строки.

**Демонстрация**: Бенчмаркинг [`fd`](https://github.com/sharkdp/fd) и
[`find`](https://www.gnu.org/software/findutils/):

![hyperfine](https://i.imgur.com/z19OYxE.gif)

## Особенности

- Статистический анализ по нескольким запускам.
- Поддержка произвольных команд оболочки.
- Постоянная обратная связь о ходе бенчмарка и текущих оценках.
- Предварительные прогонки могут выполняться перед фактическим бенчмарком.
- Команды очистки кеша могут быть настроены перед каждым временным запуском.
- Статистическое обнаружение выбросов для выявления влияния других программ и эффектов кеширования.
- Экспорт результатов в различные форматы: [AsciiDoc](https://asciidoc.org/), [CSV](https://datatracker.ietf.org/doc/html/rfc4180), [JSON](https://www.json.org/json-en.html), [Markdown](https://commonmark.org/), [Org Mode](https://orgmode.org/).
- Параметризованные бенчмарки (например, изменение количества потоков).
- Кросс-платформенный

## Использование

### Основные бенчмарки

Для запуска бенчмарка вы можете просто вызвать `hyperfine <команда>...`. Аргумент(ы) могут быть любой
командой оболочки. Например:
```sh
hyperfine 'sleep 0.3'
```

Hyperfine автоматически определит количество запусков для каждой команды. По умолчанию
будет выполнено по меньшей мере 10 запусков бенчмарка и измерено по меньшей мере 3 секунды. Чтобы изменить это,
вы можете использовать опцию `-r` / `--runs`:
```sh
hyperfine --runs 5 'sleep 0.3'
```

Если вы хотите сравнить время выполнения разных программ, вы можете передать несколько команд:
```sh
hyperfine 'hexdump file' 'xxd file'
```

### Предварительные прогонки и команды подготовки

Для программ, выполняющих много дисковых операций, результаты бенчмарка могут быть сильно
влияют кешем диска и тем, являются ли они холодными или горячими.

Если вы хотите выполнить бенчмарк на горячем кеше, вы можете использовать опцию `-w` / `--warmup`,
чтобы выполнить определенное количество выполнений программы перед фактическим бенчмарком:
```sh
hyperfine --warmup 3 'grep -R TODO *'
```
Наоборот, если вы хотите запустить бенчмарк на холодном кеше, вы можете использовать опцию `-p` / `--prepare` чтобы выполнить специальную команду перед каждым временным запуском. Например, чтобы очистить кеш жесткого диска
в Linux, вы можете выполнить

```sh
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
```
Чтобы использовать эту конкретную команду с hyperfine, вызовите `sudo -v`, чтобы временно получить права суперпользователя
и затем вызовите:
```sh
hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' 'grep -R TODO *'
```
### Параметризованные бенчмарки
Если вы хотите выполнить ряд бенчмарков, где один параметр изменяется (например, количество потоков), вы можете использовать опцию `-P` / `--parameter-scan`` и вызвать:

```sh
hyperfine --prepare 'make clean' --parameter-scan num_threads 1 12 'make -j {num_threads}'
```
Это также работает с десятичными числами. Опция `-D` / `--parameter-step-size` может использоваться
для управления шагом:

```sh
hyperfine --parameter-scan delay 0.3 0.7 -D 0.2 'sleep {delay}'
```
Это запускает `sleep 0.3`, `sleep 0.5` и `sleep 0.7`.

Для нечисловых параметров вы также можете предоставить список значений с опцией `-L`/  `--parameter-list`

```sh
hyperfine -L compiler gcc,clang '{compiler} -O2 main.cpp'
```
### Промежуточная оболочка

По умолчанию команды выполняются с использованием предопределенной оболочки (`/bin/sh` в Unix, `cmd.exe` в Windows).
Если вы хотите использовать другую оболочку, вы можете использовать опцию `-S, --shell <SHELL>`:

```sh
hyperfine --shell zsh 'for i in {1..10000}; do echo test; done'
```
Обратите внимание, что hyperfine всегда корректирует время запуска оболочки. Для этого выполняется процедура калибровки, где запускается оболочка с пустой командой (несколько раз), чтобы измерить время запуска оболочки. Затем это время вычитается из общего
времени для отображения фактического времени, используемого командой вопроса.

Если вы хотите выполнить бенчмарк без промежуточной оболочки, вы можете использовать опцию -N или `--shell=none`. Это полезно для очень быстрых команд (< 5 мс), где коррекция времени запуска оболочки создаст значительное количество шума.
Обратите внимание, что в этом случае нельзя использовать синтаксис оболочки, такой как `*` или `~`.

```sh
hyperfine -N 'grep TODO /home/user'
```











### Функции оболочки и псевдонимы
Если вы используете bash, вы можете экспортировать функции оболочки, чтобы выполнять над ними бенчмарки с hyperfine:

```sh
my_function() { sleep 1; }
export -f my_function
hyperfine --shell=bash my_function
```
В противном случае встроьте их в программу, которую вы бенчмаркаете, или подключите их оттуда:

```sh
hyperfine 'my_function() { sleep 1; }; my_function'
echo 'alias my_alias="sleep 1"' > /tmp/my_alias.sh
hyperfine '. /tmp/my_alias.sh; my_alias'
```

### Экспорт результатов

Hyperfine имеет несколько опций для экспорта результатов бенчмарков в форматы CSV, JSON, Markdown и другие (см. текст справки `--help` для получения подробной информации).

Markdown
Вы можете использовать опцию `--export-markdown` <файл> для создания таблиц, подобных следующим:

| Команда                          |   Среднее [с] | Мин. [с] | Макс. [с] | Относительно |
| :------------------------------- | ------------: | -------: | --------: | -----------: |
| `find . -iregex '.*[0-9]\.jpg$'` | 2.275 ± 0.046 |    2.243 |     2.397 |  9.79 ± 0.22 |
| `find . -iname '*[0-9].jpg'`     | 1.427 ± 0.026 |    1.405 |     1.468 |  6.14 ± 0.13 |
| `fd -HI '.*[0-9]\.jpg$'`         | 0.232 ± 0.002 |    0.230 |     0.236 |         1.00 |

#### JSON

JSON-вывод полезен, если вы хотите подробно проанализировать результаты бенчмарков.
Папка scripts/ включает в себя множество
полезных программ на Python для дальнейшего анализа результатов бенчмарка и создания полезных
визуализаций, таких как гистограмма времени выполнения или усовая диаграмма для сравнения
нескольких бенчмарков:

| ![](https://github.com/sharkdp/hyperfine/blob/master/doc/histogram.png) | ![](https://github.com/sharkdp/hyperfine/blob/master/doc/whisker.png) |
| ---------------------: | -------------------: |


### Подробная схема бенчмарка

В следующей схеме объясняется порядок выполнения различных временных запусков при использовании опций
типа `--warmup`, `--prepare <cmd>`, `--setup <cmd>` или `--cleanup <cmd>`:

![](https://github.com/sharkdp/hyperfine/blob/master/doc/execution-order.png)


## Установка

[![Статус упаковки](https://repology.org/badge/vertical-allrepos/hyperfine.svg)](https://repology.org/project/hyperfine/versions)

### Ubuntu

Загрузите соответствующий пакет `.deb` с страницы выпуска и установите его через `dpkg`:
```sh
wget https://github.com/sharkdp/hyperfine/releases/download/v1.16.1/hyperfine_1.16.1_amd64.deb
sudo dpkg -i hyperfine_1.16.1_amd64.deb
```

### Fedora

На Fedora hyperfine можно установить из официальных репозиториев:
```sh
dnf install hyperfine
```

### Alpine Linux

На Alpine Linux hyperfine можно установить из [официальных репозиториев](https://pkgs.alpinelinux.org/packages?name=hyperfine):
```sh
apk add hyperfine
```

### Arch Linux

На Arch Linux hyperfine можно установить из [официальных репозиториев](https://www.archlinux.org/packages/community/x86_64/hyperfine/):
```sh
pacman -S hyperfine
```
### Debian Linux

На Debian Linux hyperfine можно установить из [репозиториев тестирования](https://packages.debian.org/testing/main/hyperfine):
```sh
apt install hyperfine
```

### Funtoo Linux

На Funtoo Linux hyperfine можно установить [из core-kit](https://github.com/funtoo/core-kit/tree/1.4-release/app-benchmarks/hyperfine/):
```sh
emerge app-benchmarks/hyperfine
```

### NixOS

На NixOS hyperfine можно установить из [официальных репозиториев](https://nixos.org/nixos/packages.html?query=hyperfine):

```sh
nix-env -i hyperfine
```

### Void Linux

Hyperfine можно установить через `xbps`
```sh
xbps-install -S hyperfine
```

### macOS

Hyperfine можно установить через [Homebrew](https://brew.sh):
```sh
brew install hyperfine
```

Или вы можете установить с помощью [MacPorts](https://www.macports.org):
```sh
sudo port selfupdate
sudo port install hyperfine
```

### FreeBSD

Hyperfine можно установить через `pkg`:
```sh
pkg install hyperfine
```

### OpenBSD

```sh
doas pkg_add hyperfine
```

### Windows
Hyperfine можно установить через [Chocolatey](https://community.chocolatey.org/packages/hyperfine),
[Scoop](https://scoop.sh/#/apps?q=hyperfine&s=0&d=1&o=true&id=8f7c10f75ecf5f9e42a862c615257328e2f70f61), или [Winget](https://github.com/microsoft/winget-pkgs/tree/master/manifests/s/sharkdp/hyperfine):
```sh
choco install hyperfine
```
```sh
scoop install hyperfine
```
```sh
winget install hyperfine
```

### С conda

Hyperfine можно установить через [`conda`](https://conda.io/en/latest/) из канала [`conda-forge`](https://anaconda.org/conda-forge/hyperfine):

```sh
conda install -c conda-forge hyperfine
```

### С cargo (Linux, macOS, Windows)

Hyperfine можно установить из исходного кода через cargo:

```sh
cargo install --locked hyperfine
```

Убедитесь, что вы используете Rust 1.70 или новее.

### Из бинарных файлов (Linux, macOS, Windows)

Загрузите соответствующий архив с страницы выпуска.

## Альтернативные инструменты

Hyperfine вдохновлен [bench](https://github.com/Gabriella439/bench).

### Интеграция с другими инструментами

[Chronologer](https://github.com/dandavison/chronologer) — это инструмент, который использует hyperfine для
визуализации изменений времени выполнения бенчмарков в истории Git.

Не забудьте проверить папку [`scripts`](https://github.com/sharkdp/hyperfine/tree/master/scripts) в этом репозитории

для набора инструментов для работы с результатами бенчмарка hyperfine.

## Происхождение названия

Название hyperfine было выбрано в ссылке на гипертонкие уровни цезия 133, которые играют ключевую роль в определении нашей базовой единицы времени — секунды.

## Цитирование hyperfine

Спасибо за рассмотрение цитирования hyperfine в вашей исследовательской работе. Пожалуйста, ознакомьтесь с информацией
в боковой панели о том, как правильно цитировать hyperfine.

## Лицензия

hyperfine двойной лицензии в соответствии с условиями лицензии MIT и Apache License 2.0.

См. [LICENSE-APACHE](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-APACHE) и [LICENSE-MIT](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-MIT) для подробностей.
