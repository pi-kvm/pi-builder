# pi-builder
pi-builder is an extendable and easy-to-use tool to build [Arch Linux ARM](https://archlinuxarm.org) for Raspberry Pi using [Docker](https://www.docker.com).

-----
# Challenge
To build an OS, distribution developers usually use a set of shell scripts, unique for each distribution. Those scripts create a chroot with necessary packages, edit configs, add users and so on. This creates the bare minimum to load, run and be further customised by the user.

However, when you create a product running on a single-board machine (a small router, IP-camera, smart home controller, etc), you might want to log all changes you made to the fresh OS to be able to repeat them without forgetting an important step like setting up `sysctl`.

A common solution is to create a large and horrifying shell script that executes all necessary actions either on the dev machine or device itself. If you're using  `chroot` and [binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc) or need to save intermediate changes, the scripts complexity grows exponentially, and over time they become impossible to support.

-----
# What is pi-builder?
It's a new approach to target OS building on embedded devices. With it, you can build images as if it was a Docker container, not a real-world device OS. Built is described using default [docker file (https://docs.docker.com/engine/reference/builder) syntax and is executed in Docker on the dev machine. The resulting image can be exported to the SD card and loaded directly to Raspberry Pi.

-----
# Benefits
* **Biults are documented and repeatable**. A docker file is virtually ready documentation listing steps needed to set up the whole system.
* **Simplicity**. Seriously, what can be easier than writing a docker file?
* **Speed and build caching**. Target OS building can consist of hundreds of complicated and long steps. Thanks to Docker and its caching you won't run all of them each time you build a new OS; execution will start from any changed command, taking previous results from the cache.
* **Real environment testing**. When you're developing software that you'll run on Raspberry Pi it makes sense to test it using the same environment it'll run on to spare future problems.

-----
# How does it work?
Arch Linux ARM developers (and other systems as well) provide [minimal root file systems] (https://mirror.yandex.ru/archlinux-arm/os/) you can install on a flash drive to run from. As those are regular roots, you can use them to create your own base Docker image using [FROM scratch](https://docs.docker.com/develop/develop-images/baseimages). This image, however, will contain executables and libraries for the `ARM` architecture, and if your machine is, eg., `x86_64`, none of the commands in this image will run.

The Linux kernel has a special way to run binaries on a different architecture. You can configure [binfmt_misc] (https://en.wikipedia.org/wiki/Binfmt_misc) to run ARM binaries using an emulator (in this case, `qemu-arm-static` for `x86_64` ). Pi-builder has a [small script](https://github.com/pikvm/pi-builder/blob/master/tools/install-binfmt) thats sets up binfmt_misc on the host system to run ARM files.

In pi-builder, OS biulding is sepatated into **_stages_**, each of them being a different element of OS configuration. Fot example, the [ro](https://github.com/pikvm/pi-builder/tree/master/stages/ro) stage includes `Dockerfile.part` with all the nesessary instructions and configs to create a read-only root. A [watchdog](https://github.com/pikvm/pi-builder/tree/master/stages/watchdog)stage has everything needed to set up a watchdog with optimal parameters on Raspberry Pi.

You can find a full list of stages pi-builder ships with [here](https://github.com/pikvm/pi-builder/tree/master/stages) or below. You can choose the stages you need to set up your system and include them to your config. Stages are basically pieces of docker file that are combined in said order and executed during biuld. You can create your own stages similar to the existing ones.

Build sequence:
1. pi-builder downloads statically compiled Debian `qemu-arm-static` and sets up `binfmt_misc` globally on your machine.
2. The Arch Linux ARM image is downloaded and loaded into Docker as a base image.
3. The container is build using the nesessary stages -- package installation, configuration, cleanup etc. 
4. You can run `docker run` (or `make shell`) in the resulting container to check that everything's fine.
5. Pi-builder's utility [docker-extract](https://github.com/pikvm/pi-builder/blob/master/tools/docker-extract) extracts the container from Docker's internal storage and moves to the directory, like an ordinary root file system.
6. You can copy the resulting file system to the SD card and use it lo load Raspberry Pi.

-----
# Usage
To biuld with pi-builder you need a fresh Docker that cat 

# Как этим пользоваться?
Для сборки системы с помощью pi-builder вам понадобится свежий докер с возможностью запуска контейнера в [привелегированном режиме](https://docs.docker.com/engine/reference/commandline/run/#full-container-capabilities---privileged) (он требуется [вспомогательному образу](https://github.com/pikvm/pi-builder/blob/master/tools/Dockerfile.root) для установки `binfmt_misc`, форматирования SD-карты и некоторых других операций).

Вся работа с pi-builder выполняется с помощью главного [Makefile](https://github.com/pikvm/pi-builder/blob/master/Makefile) в корне репозитория. В его начале перечислены параметры, доступные для переопределения, которое можно выполнить, создав файл `config.mk` с новыми значениями. Дефолтные значения таковы:

```Makefile
PROJECT ?= common  # Пространство имен для промежуточных образов, просто назовите как нравится
BOARD ?= rpi  # Целевая платформа Raspberry Pi
STAGES ?= __init__ os pikvm-repo watchdog ro ssh-root ssh-keygen __cleanup__  # Список необходимых стейжей, об этом подробнее ниже

HOSTNAME ?= pi  # Имя хоста для получившейся системы
LOCALE ?= en_US  # Локаль будущей системы (UTF-8)
TIMEZONE ?= Europe/Moscow  # Таймзона будущей системы
REPO_URL ?= http://mirror.yandex.ru/archlinux-arm  # Зеркало пакетов и всего загружаемого контента
BUILD_OPTS ?=  # Всякие дополнительные опции для docker build

CARD ?= /dev/mmcblk0  # Путь к карте памяти

QEMU_PREFIX ?=
QEMU_RM = 1
```

Самые важные параметры - это `BOARD`, определяющий, под какую плату нужно собрать систему, `STAGES`, указывающий, какие стейжи необходимо включить и `CARD`, содержащий путь к устройству SD-карты. Вы можете переопределить их, передав новые значание вместе с вызовом `make`, или создав файл `config.mk` с новыми значениями.

Стейж `__init__` всегда должен идти первым: он содержит инициализирующие инструкции для создания базового образа системы (`FROM scratch`). Далее идут стейжи для "обживания" системы - установки полезных пакетов, настройки вачдога, приведения системы к ридонли, настройки ключей SSH для рута и очистки от временных файлов.

Вы можете создать собственные стейжи и включить их в сборку наравне с комплектными. Для этого просто заведите каталог для вашего стейжа в папке `stages` и разместите в нем файл `Dockerfile.part` по аналогии с любым другим стейжем. Как вариант, можно сделать так, как это устроено в проекте [Pi-KVM](https://github.com/pikvm/pi-kvm/tree/master/os) (для которого, собственно, pi-builder и был разработан).

-----
# Комплектные стейжи
* `__init__` - главный стейж, формирующий базовый образ на основе корневой ФС Arch Linux ARM. Он ВСЕГДА должен идти первым в списке `STAGES`. 
* `os` - просто ставит некоторые пакеты и настраивает систему, чтобы в ней было несколько более комфортно жить. Просто [посмотрите его содержимое](https://github.com/pikvm/pi-builder/tree/master/stages/os).
* `ro` - превращает систему в ридонли-ось. В таком режиме Raspberry Pi можно просто выключать из розетки без предварительной подготовки, не боясь повредить файловоую систему. Для того, чтобы временно включить систему на запись (например, для обновления), используйте команду `rw`; после всех изменений выполните команду `ro` для обратного перемонтирования в ридонли.
* `pikvm-repo` - добавляет ключ и репозиторий проекта [Pi-KVM](https://pikvm.org/repos). Нужно для вачдога, содержит другие дополнительные пакеты. Подключать не обязательно.
* `watchdog` - настраивает аппаратный вачдог.
* `ssh-root` - удаляет пользователя `alarm`, блокирует пароль `root` и добавляет в его `~/.ssh/authorized_keys` ключи из каталога [stages/ssh-root/pubkeys]. **По умолчанию там лежат ключи разработчика pi-builder, так что обязательно их замените**. Кроме того, этот стейж блокирует возможность логина через UART. Если вам требуется эта возможность, напишите свой собственный стейж с аналогичными функциями.
* `ssh-keygen` - генерирует хостовые SSH-ключи. На этом стейже ВСЕГДА будет происходить пересборка системы. В обычной ситуации ручная генерация ключей не требуется, но если система загружается в ридонли, у SSH нет возможности сгенерировать ключи самостоятельно даже при первой загрузке.
* `__cleanup__` - удаляет всякий мусор во временных папках, оставшийся от сборки.

# Ограничения
-----
Некоторые файлы, такие как `/etc/host` и `/etc/hostname`, автоматически заполняются докером и все изменения, вносимые в них из докерфайла, будут потеряны. В случае с именем хоста в `Makefile` добавлен специальный костыль, которые записывает имя хоста в экспортируемую систему, или назначает это имя при запуске `make run`. Так что если вам потребуется изменить что-нибудь в подобных файлах, то придется дописать это аналогичным образом в `Makefile`.

-----
# TL;DR
Как собрать систему под Raspberry Pi 3 и поставить ее на SD-карту:
```shell
$ git clone https://github.com/pikvm/pi-builder
$ cd pi-builder
$ make rpi3
$ make install
```

Как собрать систему со своим списком стейжей:
```shell
$ make os BOARD=rpi3 STAGES="__init__ os __cleanup__"
```

Остальные команды и заданную сборочную конфигурацию можно посмотреть так:
```shell
$ make

===== Available commands  =====
    make                # Print this help
    make rpi|rpi2|rpi3  # Build Arch-ARM rootfs with pre-defined config
    make shell          # Run Arch-ARM shell
    make binfmt         # Before build
    make scan           # Find all RPi devices in the local network
    make clean          # Remove the generated rootfs
    make format         # Format /dev/mmcblk0 to /dev/mmcblk0p1 (vfat), /dev/mmcblk0p2 (ext4)
    make install        # Install rootfs to partitions on /dev/mmcblk0

===== Running configuration =====
    PROJECT = common
    BOARD   = rpi
    STAGES  = __init__ os watchdog ro ssh-root ssh-keygen __cleanup__

    BUILD_OPTS =
    HOSTNAME   = pi
    LOCALE     = en_US
    TIMEZONE   = Europe/Moscow
    REPO_URL   = http://mirror.yandex.ru/archlinux-arm

    CARD = /dev/mmcblk0
           |-- boot: /dev/mmcblk0p1
           +-- root: /dev/mmcblk0p2

    QEMU_PREFIX =
    QEMU_RM     = 1
```

* **Важно**: проверьте в Makefile путь к SD-карте в переменнjq `CARD` и отключите автомонтирование, чтобы свежеотформатированная карта памяти не прицепилась к вашей системе, и скрипт установки не зафейлился.
* **Очень важно**: положите в каталог [stages/ssh-root/pubkeys](https://github.com/pikvm/pi-builder/tree/master/stages/ssh-root/pubkeys) свой SSH-ключ, иначе не сможете потом залогиниться в систему, или не используйте стейж `ssh-root`.
* **Еще более важно**: прочитайте весь этот README целиком, чтобы понимать, что и зачем вы все-таки делаете.

-----
# Лицензия
Copyright (C) 2018 by Maxim Devaev mdevaev@gmail.com

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see https://www.gnu.org/licenses/.

