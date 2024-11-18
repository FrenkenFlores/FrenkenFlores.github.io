---
title: "Получение логов от задачи GitLab CI/CD, ошибка 500"
categories:
  - blog
tags:
  - ci/cd
  - devops
  - gitlab
---

## Проблема

Задача работает уже два дня. Когда пытаюсь получить логи, получаю ошибку с кодом `500`. Необходимо проверить, всё ли нормально работает, не нужно ли прервать задачу.

## Сырые логи

Обычно для задач CI/CD установлен определённый лимит по выводу в стандартный поток. Может быть такое, что вывод нашей задачи превышает этот лимит, тогда в некоторых случаях возникает ошибка `500 Internal Server Error`, когда мы пытаемся получить сырой вывод задачи. И неважно, речь идёт об интерфейсе или API. Вот пример такой ошибки.
```bash
$ curl --header "PRIVATE-TOKEN: ***" https://{gitlab_url}/api/v4/projects/{project_id}/jobs/{job_id}/trace
{&quot;message&quot;:&quot;500 Internal Server Error&quot;}
```
Чтобы в таких случаях всё-таки получить вывод команды для отладки, я одновременно вывожу его в стандартный поток и сохраняю в файл с помощью утилиты tee. Это может выглядеть следующим образом:
```bash
cat somefile.txt 2>&1 | tee cat.log
``` 
С помощью `2>&1` я перенаправляю весь вывод из потока `stderr` в поток `stdout`, где `2` - представляет собой файловый дескриптор `stderr`, а `1` - `stdout`.
Теперь мне достаточно войти в `GitLab runner` через `ssh` соединнение и почитать логи.

## Docker in docker `dind`

Испольнитель CI/CD задач в GitLab часто использует принцип докер в докере `dind`. В таком случае, ваша задача будет представлять собой контейнер на испольнителе CI/CD. Находим рабочие процессы:
```bash
docker ps
$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED      STATUS      PORTS     NAMES
4dc60bfce38b   59ab366372d5   "sh -c 'if [ -x /usr…"   2 days ago   Up 2 days             runner-epnvgllbs-project-2218-concurrent-0-007ce3e00a224085-build
```
Получаем `CONTAINER ID` и подключаемся к контейнеру:
```bash
$ docker exec -it 4dc60bfce38b bash
```
А дальше находим наш файл с логами
```bash
root@runner-epnvgllbs-project-2218-concurrent-0:/# ls
bin  bin.usr-is-merged  boot  builds  cache  dev  etc  home  lib  lib.usr-is-merged  lib64  media  mnt  opt  proc  root  run  sbin  sbin.usr-is-merged  srv  sys  tmp  usr  var
```
В моем случае файл с логами находился в дириктории `builds`. Теперь, после того как узнали путь к файлу, выходим из контейнера и копируем его:
```bash
root@runner-epnvgllbs-project-2218-concurrent-0:/# exit
$ docker cp  4dc60bfce38b:builds/hwservers/bmc-tools/ ./bmc-tools
```

Таким образом, можно заранее удостовериться и понять, стоит ли прервать процесс или всё идёт своим чередом.
