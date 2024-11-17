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

Задача работает уже 2 дня. Когда пытаюсь получить логи, получаю ошибку `500`, необходимо проверить, все ли нормально работает, не нужно ли прервать задачу.

## Сырые логи

Обычно для задач CI/CD стоить определеннный лимит по выводу в стандартный вывод. Может быть такое, что вывод нашей задачи превышает этот лимит, тогда в некоторых моментах возникает ошибка `500 Internal Server Error`, когда мы пытаемся получить сырой выввод задачи. И не важно, речь идет об интерейсе или API. Вот пример такой ошибки
```bash
$ curl --header "PRIVATE-TOKEN: ***" https://{gitlab_url}/api/v4/projects/{project_id}/jobs/{job_id}/trace
{&quot;message&quot;:&quot;500 Internal Server Error&quot;}
```
Чтобы в таких случаях все таки получить вывод команды для отладки, я одноврменно вывожу его в стандартный вывод и сохраняю его в файл утилитой `tee`. Это может выглядеть следующем образом:
```bash
cat somefile.txt 2>&1 | tee cat.log
``` 
С помощью `2>&1` я перенаправляю весь вывод из поток `stderr` в потоке `stdout`, где `2` - представляет собой файловый дискриптор `stderr`, а `1` - `stdout`.
Теперь мне достаточно войти в `GitLab runner` через `ssh` соединнение и почитать логи.

## Docker in docker `dind`

Испольнитель CI/CD задач в GitLabd часто использует принцип докер в докере `dind`. В таком случае, ваша задача будет представлять собой контайнер на испольнителе CI/CD (`runner`). Находим рабочие процессы:
```bash
docker ps
```
$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED      STATUS      PORTS     NAMES
4dc60bfce38b   59ab366372d5   "sh -c 'if [ -x /usr…"   2 days ago   Up 2 days             runner-epnvgllbs-project-2218-concurrent-0-007ce3e00a224085-build
```
Получаем `CONTAINER ID` и входим в контейнер:
```bash
$ docker exec -it 4dc60bfce38b bash
```
А дальше находим наш файл с логами
```bash
root@runner-epnvgllbs-project-2218-concurrent-0:/# ls
bin  bin.usr-is-merged  boot  builds  cache  dev  etc  home  lib  lib.usr-is-merged  lib64  media  mnt  opt  proc  root  run  sbin  sbin.usr-is-merged  srv  sys  tmp  usr  var
```
В моем случае файл с логами находился в `builds`. Теперь, после того как узнали путь к файлу, выходим из контейнера и копируем его:
```bash
root@runner-epnvgllbs-project-2218-concurrent-0:/# exit
$ docker cp  4dc60bfce38b:builds/hwservers/bmc-tools/ ./bmc-tools
```

Таким образом можно удостовериться заранее и понять, стоить ли прервать процесс или все идет своим ходом.
