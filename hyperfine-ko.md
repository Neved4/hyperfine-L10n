# hyperfine
[![CICD](https://github.com/sharkdp/hyperfine/actions/workflows/CICD.yml/badge.svg)](https://github.com/sharkdp/hyperfine/actions/workflows/CICD.yml)
[![버전 정보](https://img.shields.io/crates/v/hyperfine.svg)](https://crates.io/crates/hyperfine)
[중文](https://github.com/chinanf-boy/hyperfine-zh)
[한국어](https://github.com/Neved4/hyperfine-L10n/blob/main/hyperfine-ko.md)

커맨드 라인 벤치마킹 도구.

**데모**: [`fd`](https://github.com/sharkdp/fd) 및
[`find`](https://www.gnu.org/software/findutils/) 벤치마킹:

![hyperfine](https://i.imgur.com/z19OYxE.gif)

## 특징

- 여러 실행에서의 통계 분석.
- 임의의 셸 명령 지원.
- 벤치마크 진행 상황 및 현재 추정치에 대한 지속적인 피드백.
- 실제 벤치마크 이전에 워머프 실행 가능.
- 각 타이밍 실행 전에 설정할 수 있는 캐시 제거 명령.
- 다른 프로그램과 캐싱 효과에서의 간섭을 감지하기 위한 통계적 이상치 검출.
- 결과를 다양한 형식으로 내보내기: [AsciiDoc](https://asciidoc.org/), [CSV](https://datatracker.ietf.org/doc/html/rfc4180), [JSON](https://www.json.org/json-en.html), [Markdown](https://commonmark.org/), [Org Mode](https://orgmode.org/).
- 매개 변수화된 벤치마크 (예: 스레드 수 변경).
- 크로스 플랫폼.

## 사용법

### 기본 벤치마킹

벤치마크를 실행하려면 `hyperfine <명령>...`을 호출할 수 있습니다. 인수로는 셸 명령어를 사용할 수 있습니다. 예를 들어:
```sh
hyperfine 'sleep 0.3'
```

Hyperfine은 각 명령에 대해 실행할 실행 횟수를 자동으로 결정합니다.
기본적으로 적어도 10회의 벤치마크 실행을 수행하고 적어도 3초 동안 측정합니다.
이를 변경하려면 `-r`/`--runs` 옵션을 사용할 수 있습니다:
```sh
hyperfine --runs 5 'sleep 0.3'
```

다른 프로그램의 실행 시간을 비교하려면 여러 명령을 전달할 수 있습니다:
```sh
hyperfine 'hexdump file' 'xxd file'
```

### 예열 실행 및 준비 명령

디스크 I/O가 많이 발생하는 프로그램의 경우 벤치마킹 결과는 디스크 캐시와 디스크 캐시가 얼음이든 더운지 여부에 크게 영향을 받을 수 있습니다.

웜 캐시에서 벤치마크를 실행하려면 `-w` / `--warmup` 옵션을 사용하여 실제 벤치마크
이전에 특정 횟수의 프로그램 실행을 수행할 수 있습니다:
```sh
hyperfine --warmup 3 'grep -R TODO *'
```
반면 콜드 캐시에서 벤치마크를 실행하려면 `-p` / `--prepare` 옵션을 사용하여 각 타이밍 실행 전에 특별한 명령을 실행할 수 있습니다. 예를 들어, Linux에서 하드디스크 캐시를 지우려면 다음 명령을 실행할 수 있습니다:

```sh
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
```
이 특정 명령을 hyperfine과 함께 사용하려면 임시로 sudo 권한을 얻기 위해 `sudo -v`를 호출하고 다음과 같이 호출하십시오:
```sh
hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches' 'grep -R TODO *'
```

### 매개 변수화된 벤치마크

단일 매개 변수 (예: 스레드 수)를 변화시키는 일련의 벤치마크를 실행하려면 `-P` / `--parameter-scan` 옵션을 사용하고 다음과 같이 호출하십시오:
```sh
hyperfine --prepare 'make clean' --parameter-scan num_threads 1 12 'make -j {num_threads}'
```
이는 십진수로도 작동합니다. `-D` / `--parameter-step-size` 옵션을 사용하여 단계 크기를 제어할 수 있습니다:
```
hyperfine --parameter-scan delay 0.3 0.7 -D 0.2 'sleep {delay}'
```
이렇게 하면 `sleep 0.3`, `sleep 0.5` 및 `sleep 0.7`이 실행됩니다.

숫자가 아닌 매개 변수에 대해서도 `-L` / `--parameter-list` 옵션으로 값을 리스트로 제공할 수 있습니다:

```sh
hyperfine -L compiler gcc,clang '{compiler} -O2 main.cpp'
```

### 중간 셸

기본적으로 명령은 미리 정의된 셸 (`/bin/sh`에서 Unix, Windows의 `cmd.exe`)을 사용하여 실행됩니다. 다른 셸을 사용하려면 `-S, --shell <SHELL>` 옵션을 사용할 수 있습니다:
```sh
hyperfine --shell zsh 'for i in {1..10000}; do echo test; done'
```

hyperfine은 항상 셸 생성 시간을 보정합니다.
이를 위해 빈 명령을 사용하여 셸을 실행하는 보정 프로시저를 수행하여 셸의 시작 시간을 측정합니다.
그런 다음 이 시간을 해당 명령에 의한 실제 시간에서 뺍니다.

보정되지 않은 셸에서 벤치마크를 실행하려면 `-N` 또는 `--shell=none` 옵션을 사용할 수 있습니다.
이것은 셸 시작 오버헤드 보정이 큰 소음 양을 생성하는 매우 빠른 명령 (< 5 ms)에 유용합니다.
이 경우에는 `*` 또는 `~`와 같은 셸 구문을 사용할 수 없습니다.
```sh
hyperfine -N 'grep TODO /home/user'
```











### 셸 함수 및 별칭

bash를 사용하는 경우 셸 함수를 내보내어 hyperfine으로 직접 벤치마킹할 수 있습니다:

```sh
my_function() { sleep 1; }
export -f my_function
hyperfine --shell=bash my_function
```

그렇지 않으면 벤치마킹 대상 프로그램에 내장하거나 그 프로그램에서 가져올 수 있습니다:

```sh
hyperfine 'my_function() { sleep 1; }; my_function'
echo 'alias my_alias="sleep 1"' > /tmp/my_alias.sh
hyperfine '. /tmp/my_alias.sh; my_alias'
```

### 결과 내보내기

Hyperfine은 CSV, JSON, Markdown 및 기타 형식으로 벤치마크 결과를 내보내기 위한 여러 옵션을 제공합니다 (자세한 내용은 `--help` 텍스트를 참조하십시오).

#### Markdown

`--export-markdown` <파일> 옵션을 사용하여 다음과 같은 테이블을 만들 수 있습니다:

| 명령                             |      평균 [s] | 최소 [s] | 최대 [s] |      상대적 |
| :------------------------------- | ------------: | -------: | -------: | ----------: |
| `find . -iregex '.*[0-9]\.jpg$'` | 2.275 ± 0.046 |    2.243 |    2.397 | 9.79 ± 0.22 |
| `find . -iname '*[0-9].jpg'`     | 1.427 ± 0.026 |    1.405 |    1.468 | 6.14 ± 0.13 |
| `fd -HI '.*[0-9]\.jpg$'`         | 0.232 ± 0.002 |    0.230 |    0.236 |        1.00 |

#### JSON

JSON 출력은 벤치마크 결과를 더 자세히 분석하려는 경우 유용합니다.
[`scripts/`](https://github.com/sharkdp/hyperfine/tree/master/scripts) 폴더에는 벤치마크 결과를 더 자세히 분석하고 도움이 되는 시각화를 생성하는 데 도움이 되는 다양한 파이썬 프로그램이 많이 포함되어 있습니다. 예를 들어 런타임의 히스토그램이나 여러 벤치마크를 비교하기 위한 Whisker Plot과 같은 것입니다:

| ![](doc/histogram.png) | ![](doc/whisker.png) |
| ---------------------: | -------------------: |


### 상세한 벤치마크 플로차트

다음 차트는 `--warmup`, `--prepare <cmd>`, `--setup <cmd>` 또는 `--cleanup <cmd>`와
같은 옵션을 사용할 때 다양한 타이밍 실행의 실행 순서를 설명합니다:

![](doc/execution-order.png)

## 설치

[![패키징 상태](https://repology.org/badge/vertical-allrepos/hyperfine.svg)](https://repology.org/project/hyperfine/versions)

### Ubuntu

릴리스 페이지에서 해당하는 .deb 패키지를 다운로드하고 dpkg를 사용하여 설치할 수 있습니다:
```sh
wget https://github.com/sharkdp/hyperfine/releases/download/v1.16.1/hyperfine_1.16.1_amd64.deb
sudo dpkg -i hyperfine_1.16.1_amd64.deb
```

### Fedora

Fedora에서 hyperfine은 공식 저장소에서 설치할 수 있습니다:

```sh
dnf install hyperfine
```

### Alpine Linux

Alpine Linux에서 hyperfine을 [공식 저장소로부터](https://pkgs.alpinelinux.org/packages?name=hyperfine) 설치할 수 있습니다.:
```sh
apk add hyperfine
```

### Arch Linux

Arch Linux에서 hyperfine을 [공식 저장소로부터](https://www.archlinux.org/packages/community/x86_64/hyperfine/) 설치할 수 있습니다:
```sh
pacman -S hyperfine
```

### Debian Linux

Debian Linux에서 hyperfine을 [공식 저장소로부터](https://packages.debian.org/testing/main/hyperfine) 설치할 수 있습니다:
```sh
apt install hyperfine
```

### Funtoo Linux

Funtoo Linux에서 hyperfine을 [공식 core-kit](https://github.com/funtoo/core-kit/tree/1.4-release/app-benchmarks/hyperfine/) 설치할 수 있습니다:
```sh
emerge app-benchmarks/hyperfine
```

### NixOS

NixOS에서 hyperfine을 [공식 저장소로부터](https://nixos.org/nixos/packages.html?query=hyperfine) 설치할 수 있습니다:
```sh
nix-env -i hyperfine
```

### Void Linux

Hyperfine은 xbps를 통해 설치할 수 있습니다:

```sh
xbps-install -S hyperfine
```

### macOS

hyperfine은 [Homebrew](https://brew.sh)를 통해 설치할 수 있습니다:
```sh
brew install hyperfine
```

또는 [MacPorts](https://www.macports.org)를 사용하여 설치할 수 있습니다:
```sh
sudo port selfupdate
sudo port install hyperfine
```

### FreeBSD

hyperfine은 pkg를 통해 설치할 수 있습니다:
```sh
pkg install hyperfine
```

### OpenBSD

```sh
doas pkg_add hyperfine
```

### Windows

hyperfine은 [Chocolatey](https://community.chocolatey.org/packages/hyperfine),
[Scoop](https://scoop.sh/#/apps?q=hyperfine&s=0&d=1&o=true&id=8f7c10f75ecf5f9e42a862c615257328e2f70f61)
또는 [Winget](https://github.com/microsoft/winget-pkgs/tree/master/manifests/s/sharkdp/hyperfine) 를 통해 설치할 수 있습니다:
```sh
choco install hyperfine
```
```sh
scoop install hyperfine
```
```sh
winget install hyperfine
```

### conda로 설치

Hyperfine은 [`conda`](https://conda.io/en/latest/)를 통해
[`conda-forge`](https://anaconda.org/conda-forge/hyperfine) 채널에서 설치할 수 있습니다:
```sh
conda install -c conda-forge hyperfine
```

### cargo로 설치 (Linux, macOS, Windows)

Hyperfine은 [cargo](https://doc.rust-lang.org/cargo/)를 통해 소스로부터 설치할 수 있습니다:
```sh
cargo install --locked hyperfine
```

Rust 1.70 이상을 사용해야 합니다.

### 바이너리에서 설치 (Linux, macOS, Windows)

릴리스 페이지에서 해당하는
아카이브를 다운로드하십시오.

## 대안 도구

Hyperfine은 [bench](https://github.com/Gabriella439/bench)에서 영감을 받았습니다.

## 다른 도구와의 통합

[Chronologer](https://github.com/dandavison/chronologer)는 깃 히스토리에서 벤치마크 시간의 변경을
시각화하기 위해 hyperfine을 사용하는 도구입니다.

hyperfine 벤치마크 결과와 작업하기 위한 도구 세트를 확인하려면 이 저장소의 [`scripts` folder](https://github.com/sharkdp/hyperfine/tree/master/scripts)
폴더를 확인하십시오.

## 이름의 유래

hyperfine이라는 이름은 시간의 기본 단위를 정의하는
데 중요한 역할을 하는 세슘133의 초미세 구조 준위에
착안하여 선택되었습니다.

## hyperfine 인용

hyperfine을 연구 작업에서 인용해 주셔서 감사합니다.
hyperfine을 올바르게 인용하는 방법에 대한 정보는 사이드바에서 확인하십시오.

## 라이선스

hyperfine은 MIT 라이선스와 Apache License 2.0의 조건에 따라 이중 라이선스가 부여됩니다.

세부 정보는 [LICENSE-APACHE][] 및 [LICENSE-MIT][] 파일을 참조하십시오.
