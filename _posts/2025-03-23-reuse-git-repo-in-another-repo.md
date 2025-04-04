---
title: "Как переиспользовать код из одного репозитория в другом"
categories:
  - blog
tags:
  - ci/cd
  - devops
  - gitlab
  - git
  - repo
  - submodules

---

## Git submodules

С ростом количества репозиториев растет и количество дублированного кода, который решает одну и ту же задачу в разных репозиториях: форматирование вывода, перебор паролей, получение данных по сети, файлы с конфигурациями и т.д.

Основная проблема заключается в том, что одни и те же правки приходится вносить во всех репозиториях, и это не обходится без человеческого фактора. Поэтому нужно минимизировать количество изменяемого кода. Сделать это можно, если перенести весь дублированный код в отдельные утилиты или библиотеки, к которым будут обращаться основные программы. Затем следует создать для них отдельный репозиторий, который будет включать в себя все инструменты разработки.

## Инициализация

После того как мы перенесли все инструменты в отдельный репозиторий, мы можем клонировать его как вложенный репозиторий (подмодуль, submodule) в наш основной. Для этого выполняем:
```bash
git submodule add git@gitlab.ru:hwservers/bmc-tools.git
```
Затем:
```bash
git submodule update --init
```
или:
```bash 
git submodule init
git submodule update
```
В итоге у нас появится вложенный репозиторий и файл `.gitmodules`, у которого следующее содержание:
```
[submodule "bmc-tools"]
  path = bmc-tools
  url = git@gitlab.ru:hwservers/bmc-tools.git
```

## Подтягивание изменений для submodule

Репозиторий с инструментами разработки (библиотеки, скрипты, программы, конфигурации и т.д.) со временем будет обновляться. Чтобы подтянуть эти изменения в основной репозиторий, нужно выполнить следующую команду:
```bash
git submodule update --remote --recursive
```
Флаг `remote` указывает, что нужно обновить подмодули до версий, которые находятся в их удалённых репозиториях, а не до версий, зафиксированных в локальном вложенном репозитории (например, если переключились на другую ветку). Флаг `recursive` нужен, чтобы обновить все подмодули, включая подмодули внутри подмодулей.

Далее необходимо зафиксировать версию вложенного репозитория, чтобы при клонировании основного репозитория Git подтянул нужную версию вложенного. Сделать так, чтобы автоматически всегда подтягивалась последняя версия вложенного репозитория, не получится. Остается только держать нужную версию в коммитах основного репозитория:

```bash
git add <submodule>
git commit -m "<msg>"
git push
```

## Клонирование репозитория с вложенным репозиторием

После клонирования основного репозитория вложенный репозиторий будет пустым. Чтобы скачать его содержимое, необходимо отдельно инициализировать его изнутри основного:
```bash
cd <repo name>
git submodule update --init --remote --recursive 
Submodule 'bmc-tools' (git@gitlab.ru:hwservers/bmc-tools.git) registered for path 'bmc-tools'
Cloning into '/my/repo/path/bmc-tools'...
Submodule path 'bmc-tools': checked out 'dd425af34b26faa938fe899a3e751bb5b71b8af5'
```

## CI/CD

Во время выполнения pipeline'а CI/CD Git клонирует наш репозиторий. Если в нем содержится вложенный репозиторий, его необходимо отдельно инициализировать. Это можно сделать, указав нужную команду в списке команд, выполняемых pipeline'ом. Или можно воспользоваться встроенным функционалом.

В GitLab можно настроить инициализацию вложенного репозитория для pipeline'а CI/CD, как указано в [документации](https://docs.gitlab.com/ci/runners/git_submodules/#use-git-submodules-in-cicd-jobs):

1. GitLab CI/CD использует специальный токен `CI_JOB_TOKEN`, который действует как персональный токен. Он будет использоваться для клонирования вложенного репозитория через HTTPS внутри основного.  
2. В настройках CI/CD вложенного репозитория через веб-интерфейс найдите вкладку `Token Access` и отключите ограничение на клонирование репозитория CI-задачами из других репозиториев или добавьте эти репозитории в список разрешённых.  
3. В вашем репозитории в файле `.gitlab-ci.yml` добавьте следующий код:
```yml
variables:
  GIT_SUBMODULE_STRATEGY: recursive # git clone <x>.git --recurse-submodules
  GIT_SUBMODULE_FORCE_HTTPS: "true" # git@<x>.git -> https://<x>.git
  GIT_SUBMODULE_DEPTH: 1 # git clone --depth <n> https://<x>.git
```
