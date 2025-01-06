# Автоматизация развертывания и эксплуатации программного обеспечения - DevOps

[Я в Телеграм](https://t.me/amunra2) <img src="https://img.icons8.com/external-tal-revivo-shadow-tal-revivo/344/external-telegram-is-a-cloud-based-instant-messaging-and-voice-over-ip-service-logo-shadow-tal-revivo.png" alt="Telegram" width=15>


# Лабораторная работа №2

## Условие

[Условие лабораторной работы](./task.pdf)

## Выполнение

> 1. Выполнялась лаба с имеющимся проектом с курса по [Web-разработке](https://github.com/amunra2/web-bmstu-iu7). Проект был выполнен на C#/TS, поэтому далее будут устанавливаться необходимые для его работы пакеты - dotnet, npm (уже не помню, но вроде нельзя было сложить проект в Docker, но утверждать не могу)
> 2. Для выполнения нужно 2 виртуальные машины (их установка описана в [лабораторной работе 1](../lab_01/README.md))

1. Создать репозиторий в GitLab для проекта
2. Загрузить в него проект
3. Если в проекте используется база, то развернуть ее можно на этом [сайте](https://customer.elephantsql.com/instance)

1. Первая виртуальная машина
    1. Подключение через ssh, поэтому в базисе нужно в `port forwarding` добавить новый порт с праметрами: внешние — одинаковые и какие захотите, а внутренний — 22
    2. Подключаемся по ssh:
    ```
    ssh user@<ip вашей ВМ1> -p <ваш внешний порт>
    ```
    3. Установить gitlab-runner
    ```
    curl -L "[https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh](https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh)" | sudo bash 
    sudo apt-get install gitlab-runner
    ```
    4. Зарегистрировать gitlab-runner
        1. Команда `sudo gitlab-runner register`
        2. В url: [https://git.iu7.bmstu.ru/](https://git.iu7.bmstu.ru/)
        3. Токен: в репозитории → settings → CI/CD → runners (expand). Там лежит ваш токен
        4. Описание: любое
        5. Теги: build,test,deploy
        6. Следующий шаг скипаем
        7. Исполнитель: shell
        8. Готово (можно в репе проверить, что он есть)
        
    5. Установить dotnet (для бекенда C# приложений): `sudo apt-get update &&   sudo apt-get install -y dotnet-sdk-6.0`
    6. Установить npm (для фронтенда): `sudo apt install npm`
    7. Установить sshpass: `sudo apt install sshpass`
    8. В корне проекта создать файл `.gitlab-ci.yml` и заполнить его. Мой файл представлен ниже (нужно добавить еще в стадию деплоя прокидывание конфига `nginx.conf`, у меня это не сделано). При этом добавить в веб версии нужные переменные.
        
        ```yaml
        stages:
          - build
          - test
          - deploy
        
        build-backend:
          stage: build
          tags:
            - build
          script:
            - cd backend/
            - dotnet publish -f net6.0 -c Release -o ./build
          artifacts:
            untracked: false
            when: on_success
            expire_in: "1 hour"
            paths:
              - backend/build
        
        build-frontend:
          stage: build
          tags:
            - build
          script:
            - cd frontend/
            - npm i
            - npm run build
            - mv dist build
          artifacts:
            untracked: false
            when: on_success
            expire_in: "1 hour"
            paths:
              - frontend/build
        
        test-backend-unit-bl:
          stage: test
          tags:
            - test
          script:
            - cd test/UnitBL/
            - dotnet test
        
        test-backend-unit-db:
          stage: test
          tags:
            - test
          script:
            - cd test/UnitDB/
            - dotnet test 
        
        deploy:
          stage: deploy
          tags:
            - test
          script:
            - sshpass -p $PROD_PASSWORD rsync -e "ssh -o StrictHostKeyChecking=no" -a backend/build/ $PROD_USER@$PROD_HOST:/var/opt/app
            - sshpass -p $PROD_PASSWORD rsync -e "ssh -o StrictHostKeyChecking=no" -a --delete frontend/build/ $PROD_USER@$PROD_HOST:/var/www/site
            - echo $PROD_PASSWORD | sshpass -p $PROD_PASSWORD ssh $PROD_USER@$PROD_HOST sudo -S systemctl restart servering-backend.service
          rules:
            - if: '$CI_COMMIT_REF_NAME == "dev" || $CI_COMMIT_REF_NAME == "main"'
              when: manual
            - when: never
        ```
        
    9. Готово
2. Вторая виртуальная машина
    1. Подключение через ssh, поэтому в базисе нужно в `port forwarding` добавить новый порт с праметрами: внешние — одинаковые и какие захотите, а внутренний — 22
    2. Подключаемся по ssh:
    ```
    ssh user@<ip вашей ВМ2> -p <ваш внешний порт>
    ```
    3. Установить dotnet (для бекенда C# приложений): `sudo apt-get update &&   sudo apt-get install -y dotnet-sdk-6.0`
    4. Создать папку для фронтенда: `sudo mkdir /var/www/site` и выдать на нее права пользователю: `sudo chown -R $USER /var/www/site`
    5. Создать папку для бекенда: `sudo mkdir /var/opt/app` и выдать на нее права пользователю: `sudo chown -R $USER /var/opt/app`
    6. Установить nginx: `sudo apt install nginx`
    7. Прописать конфиг  `nginx.conf`. Мой конфиг:
        
        ```bash
        worker_processes  1;
        pid /var/run/nginx.pid;
        events {
          worker_connections  1024;
        }
        
        http {
          include       /etc/nginx/mime.types;
                    
          sendfile        on;
          keepalive_timeout  65;
        
          server {
            listen 80;
            server_name localhost;
        
            root /var/www/site;
            index index.html index.htm;
        
            location / {
              try_files $uri $uri/ /index.html;
            }
        
            location /api/v1/ {
              proxy_no_cache 1;
              proxy_pass 'http://localhost:5555/api/v1/';
            }
        
            error_log /var/log/nginx/front-error.log;
            access_log /var/log/nginx/front-access.log;
          }
        }
        ```
        
    8. Запускать бэкенд как сервис
        1. Создаем файл: `sudo nano /etc/systemd/system/servering-backend.service`
        2. Текст в моем случае такой
            
            ```bash
            [Unit]
            Description=Backend for website ServerING
            
            [Service]
            ExecStart=/var/opt/app/ServerING
            Restart=always
            
            [Install]
            WantedBy=multi-user.target
            ```
            
        3. Перезапустить сервис бекенда: `sudo systemctl restart servering-backend.service`
3. Прокинуть пакетики на ВМ2 через ВМ1 
    1. Добавить в port forwarding в базисе для ВМ1 порт с внутренним 80, а внешним любым
    2. Ввести следующие команды со своими и внутренними айпишниками  машин
        
        ```bash
        sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward”
        sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination <ваш внутренний айпи ВМ2>
        sudo iptables -t nat -A POSTROUTING -p tcp -d <ваш внутренний айпи ВМ2> --dport 80 -j SNAT --to-source <ваш внутренний айпи ВМ1>
        ```
        
4. Лаба готова

## Возможные проблемы


> При проблеме на деплое поднобного вида: **Failed to add the host to the list of known hosts**
>
> Необходимо прописать следующую команду на ВМ1 `sudo chown -v gitlab-runner ./gitlab-runner/.ssh/known_hosts` 

> **Пересылка портов**  (сохранить изменения (я не стал))
>
> 1. Сохранить **iptables** ([источник](https://askubuntu.com/questions/119393/how-to-save-rules-of-the-iptables))
> 2. Сохранить **ip-forward**: 
`sudo nano /etc/sysctl.conf` и в нем расскоментировать строку
`net.ipv4.ip_forward=1`
>

_@amunra2 (2023г.)_
