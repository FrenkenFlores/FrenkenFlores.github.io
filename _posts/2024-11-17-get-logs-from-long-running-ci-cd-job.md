---
title: "Получение логов от длительно выполняющейся задачи GitLab CI/CD. Ошибка 500 (Internal Server Error)"
categories:
  - blog
tags:
  - ci/cd
  - devops
  - gitlab
  - troubleshooting
  - runner
  - executor
  - dind
  - docker
---

При выполнении длительной задачи GitLab CI/CD, чей размер логов превышает стандартный лимит, иногда возникает проблема с просмотром логов - при попытке их получить появляется ошибка с кодом `500`. Необходимо проверить статус выполнения задачи и определить, нужно ли её прервать.


## Сырые логи

Обычно для задач CI/CD установлен определённый лимит по выводу в стандартный поток. Может быть такое, что вывод нашей задачи превышает этот лимит, тогда в некоторых случаях возникает ошибка `500 Internal Server Error`, когда мы пытаемся получить сырой вывод задачи. И неважно, речь идёт об интерфейсе или API. Вот пример такой ошибки.
```bash
>$ curl --header "PRIVATE-TOKEN: ***" https://{gitlab_url}/api/v4/projects/{project_id}/jobs/{job_id}/trace
{&quot;message&quot;:&quot;500 Internal Server Error&quot;}
```
Для начала нам нужен доступ к выводу исполняемой задачи. По умолчанию это `stdout` и `stderr`, но проблема потоков вывода `stdout` и `stderr` в том, что они не сохраняют свое содержимое - как только в них выводятся строки, тут же они и исчезают. То есть мы таким образом не сможем получить все логи. Чтобы в таких случаях всё-таки получить вывод команды для отладки, я одновременно вывожу и сохраняю его в файл с помощью утилиты [`tee`](https://www.man7.org/linux/man-pages/man1/tee.1.html). Это может выглядеть следующим образом:
```bash
>$ cat somefile.txt 2>&1 | tee cat.log
```
Утилита `tee` дублирует только поток вывода `stdout`. Чтобы не терять поток вывода ошибки `stderr`, с помощью `2>&1` я перенаправляю весь вывод из потока `stderr` в поток `stdout`, где `2` - представляет собой файловый дескриптор `stderr`, а `1` - `stdout`.
Таким образом мы не будем блокировать вывод задачи в веб-интерфейсе.

Мои задачи в `.gitlab-ci.yml` будут описаны следующим образом:

```yml
some_task:
  stage: some_stage
  tags: [some_gitlab_runner]
  script:
    - some_task.sh 2>&1 | tee some_log.log
  artifacts:
    paths:
      - ./some_log.log
```
Как вы заметили, такой способ еще дает возможность сохранять логи в качестве артифактов, чтобы можно было по окончании их скачать и более детально изучать через IDE.

Далее нам нужен доступ, например по протоколу `SSH`, к GitLab runner, который выполняет наши CI/CD задачи, чтобы посмотреть содержимое нашего репозитория. Обычно прежде чем начать выполнение задачи, GitLab runner должен склонировать репозиторий. Он как раз и будет нашей текущей директорией, поэтому в ней будут сохранен наш файл с логами, созданный утилитой `tee`.

Теперь нам достаточно войти в `GitLab runner` через `ssh` соединнение, чтобы почитать логи, если у вас GitLab runner использует [`Shell executor`](https://docs.gitlab.com/runner/executors/shell.html).
```bash
>$ ssh user@gitlab-runner-host
gitlab-runner-host:/>$ ls /home/gitlab-runner/builds/
```

## Docker in docker `dind`

Исполнитель (`runner`) CI/CD задач в GitLab часто использует принцип докер в докере `dind`. В таком случае ваша задача будет представлять собой контейнер на исполнителе CI/CD. Находим рабочие процессы:
```bash
>$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED      STATUS      PORTS     NAMES
4dc60bfce38b   59ab366372d5   "sh -c 'if [ -x /usr…"   2 days ago   Up 2 days             runner-epnvgllbs-project
```
Получаем `CONTAINER ID` и подключаемся к контейнеру:
```bash
gitlab-runner-host:/>$ docker exec -it 4dc60bfce38b bash
```
А дальше находим наш файл с логами
```bash
root@runner-epnvgllbs-project:/>$ ls
bin  bin.usr-is-merged  boot  builds  cache  dev  etc  home  lib  lib.usr-is-merged  lib64  media  mnt  opt  proc  root  run  sbin  sbin.usr-is-merged  srv  sys  tmp  usr  var
```
В моем случае файл с логами находился в директории `builds`. Теперь, после того как узнали путь к файлу, выходим из контейнера и копируем его:
```bash
root@runner-epnvgllbs-project:/>$ exit
$ docker cp  4dc60bfce38b:builds/hwservers/bmc-tools/ ./bmc-tools
```
Затем через `scp` скачиваем его из исполнителя GitLab.
```bash
>$ scp user@gitlab-runner-host:~/bmc-tools local_dir/bmc-tools
```

Таким образом, можно заранее удостовериться и понять, стоит ли прервать процесс или всё идёт своим чередом.
