# Один день из жизни DevOps

Этот проект демонстрирует эволюцию простого веб-приложения от базового скрипта до развертывания в Kubernetes, проходя через ключевые практики и инструменты DevOps.

## Обзор

Мы начинаем с простого Python-скрипта и последовательно добавляем:
*   Контроль версий (Git)
*   Веб-серверы (Apache, Nginx, uWSGI)
*   Автоматизацию развертывания (Ansible)
*   Контейнеризацию (Docker, Docker Compose)
*   Оркестрацию (Kubernetes)

Цель — показать, как эти инструменты интегрируются в жизненный цикл разработки и доставки ПО.

## Начало работы: Простое приложение

Создаем каталог и простое веб-приложение на Python.

```bash
mkdir hello
cd hello
```

Создаем файл приложения `app.py`:

```python
#!/usr/bin/env python3

import datetime

def do_magic():
    now = datetime.datetime.now()
    return "Hello! {0}".format(now)

if __name__ == "__main__":
    print(do_magic())
```

Делаем скрипт исполняемым и запускаем его:

```bash
chmod +x ./app.py
./app.py
```

## Этап 1: Контроль версий (Git)

Устанавливаем Git и инициализируем репозиторий.

```bash
sudo apt install git
git init
```

Настраиваем пользователя для коммитов:

```bash
git config --global user.email "alm15@mail.ru"
git config --global user.name "Almaskhan Oskenbayev"
```

Добавляем файл приложения в репозиторий и создаем первый коммит.

```bash
git add app.py
git commit -m 'first version'
```

Просматриваем историю коммитов:

```bash
git log
```

## Этап 2: Веб-сервер Apache2 + CGI

Устанавливаем и настраиваем Apache для работы с нашим скриптом через CGI.

```bash
sudo apt install apache2
```

Настраиваем права и символическую ссылку для доступа к нашему каталогу.

```bash
sudo usermod www-data -a -G user
sudo systemctl restart apache2
cd /var/www
sudo mv html html2
sudo ln -s ~/hello/ html
```

Включаем модуль CGI и настраиваем виртуальный хост.

```bash
sudo a2enmod cgi
sudo systemctl restart apache2
```

Создаем файл `.htaccess` в каталоге `~/hello`:

```
AddHandler cgi-script .py
Options +ExecCGI
DirectoryIndex app.py
```

Модифицируем `app.py` для работы в качестве CGI-скрипта:

```python
#!/usr/bin/env python3
import datetime
import os

def do_magic():
    now = datetime.datetime.now()
    return "Hello! {0}".format(now)

if __name__ == "__main__":
    if 'REQUEST_URI' in os.environ:
        print("Content-type: text/html\n\n")
    print(do_magic())
```

Фиксируем изменения в Git и создаем тег `v1.0`.

```bash
git add .htaccess app.py
git commit -m 'apache2 cgi version'
git tag v1.0
```

## Этап 3: Переход на uWSGI

Создаем новую ветку для разработки с uWSGI.

```bash
git checkout -b uwsgi
```

Модифицируем `app.py` для поддержки стандарта WSGI.

```python
#!/usr/bin/env python3
import datetime
import os

def do_magic():
    now = datetime.datetime.now()
    return "Hello! {0}".format(now)

def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [do_magic().encode()]

if __name__ == "__main__":
    if 'REQUEST_URI' in os.environ:
        print("Content-type: text/html\n\n")
    print(do_magic())
```

Устанавливаем uWSGI и плагин для Python3.

```bash
sudo apt install uwsgi-plugin-python3
```

Создаем конфигурационный файл `dev.ini` для uWSGI:

```ini
[uwsgi]
plugin=python3
http-socket=:9090
wsgi-file=app.py
```

Запускаем приложение через uWSGI:

```bash
uwsgi dev.ini
```

Фиксируем изменения и сливаем ветку `uwsgi` с `master`.

```bash
git add app.py dev.ini
git commit -m 'uwsgi version'
git checkout master
git merge uwsgi
```

## Этап 4: Nginx в качестве обратного прокси

Заменяем Apache на Nginx.

```bash
sudo systemctl disable apache2
sudo systemctl stop apache2
sudo apt install nginx -y
```

Создаем конфигурационный файл для Nginx `/etc/nginx/sites-available/hello.conf`:

```nginx
server {
    listen 80;
    root /var/www/html;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:9000;
        uwsgi_param Host $host;
        uwsgi_param X-Real-IP $remote_addr;
        uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
        uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;
    }
}
```

Активируем конфигурацию и перезапускаем Nginx.

```bash
sudo ln -s /etc/nginx/sites-available/hello.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

Создаем конфигурационный файл `prod.ini` для uWSGI, который будет работать с Nginx.

```ini
[uwsgi]
plugin=python3
socket=127.0.0.1:9000
wsgi-file=app.py
```

Запускаем uWSGI с новой конфигурацией.

```bash
uwsgi prod.ini
```

## Этап 5: Подготовка к автоматическому развертыванию (Ansible)

Создаем структуру каталогов для хранения конфигурационных файлов развертывания.

```bash
sudo mkdir -p deploy/{apache2,uwsgi,nginx,systemd}
sudo mv .htaccess deploy/apache2/
sudo mv *.ini deploy/uwsgi/
sudo cp /etc/nginx/sites-available/hello.conf deploy/nginx/
```

Фиксируем изменения в Git.

```bash
git add deploy/
git commit -m 'prepare to deploy'
```

Создаем и настраиваем LXD-контейнер для тестирования развертывания.

```bash
sudo lxd init
# На все вопросы отвечаем по умолчанию, кроме выбора storage backend (выбираем dir)
```

Настраиваем профиль LXD для автоматической установки SSH-ключа.

```bash
lxc profile edit default
# Добавляем в конфигурацию:
# config:
#   cloud-init.user-data: |
#     #cloud-config
#     packages:
#       - openssh-server
#     ssh_authorized_keys:
#       - "ssh-rsa AAA... ваш_публичный_ключ"
```

Запускаем контейнер.

```bash
lxc launch ubuntu: test-container
```

Устанавливаем Ansible.

```bash
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

Создаем структуру для плейбуков Ansible.

```bash
mkdir ansible
cd ansible
```

Создаем файл конфигурации `ansible.cfg`:

```ini
[defaults]
inventory = hosts
host_key_checking = False
interpreter_python=auto_silent
```

Создаем файл инвентаря `hosts`:

```ini
test ansible_host=10.169.189.127 ansible_user=ubuntu
```

Проверяем подключение к контейнеру.

```bash
ansible -m ping test
```

Создаем плейбук для развертывания `deploy.yml`:

```yaml
---
- name: deploy hello app
  gather_facts: false
  hosts: test
  become: true
  vars:
    repo: https://github.com/almsys/hello
    repo_dir: '/opt/hello'
    packages:
      - nginx
      - python3
      - uwsgi-plugin-python3

  tasks:
    - name: update apt cache
      apt:
        update_cache: yes

    - name: install packages
      package:
        name: "{{ item }}"
        state: latest
      with_items: "{{ packages }}"

    - name: checkout repo
      git:
        repo: "{{ repo }}"
        dest: "{{ repo_dir }}"
        version: v2.0

    - name: copy systemd config
      copy:
        remote_src: yes
        src: "{{ repo_dir }}/deploy/systemd/hello.service"
        dest: "/etc/systemd/system/hello.service"

    - name: enable and start service
      systemd:
        name: hello
        daemon_reload: yes
        enabled: yes
        state: started

    - name: copy nginx config
      copy:
        remote_src: yes
        src: "{{ repo_dir }}/deploy/nginx/hello.conf"
        dest: "/etc/nginx/sites-available/hello.conf"

    - name: disable default nginx config
      file:
        state: absent
        path: /etc/nginx/sites-enabled/default
      notify: restart nginx

    - name: enable our nginx config
      file:
        src: /etc/nginx/sites-available/hello.conf
        dest: /etc/nginx/sites-enabled/hello.conf
        state: link
      notify: restart nginx

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

Запускаем плейбук.

```bash
ansible-playbook deploy.yml
```

Фиксируем плейбуки в Git.

```bash
git add ansible/
git commit -m 'ansible deploy'
git push origin master
```

## Этап 6: Контейнеризация (Docker)

Создаем `Dockerfile` для сборки образа нашего приложения.

```dockerfile
FROM tiangolo/uwsgi-nginx:python3.7

COPY ./app.py /app/app.py
COPY ./deploy/uwsgi/prod.ini /app/uwsgi.ini

WORKDIR /app
```

Собираем Docker-образ.

```bash
docker build -t almsys/hello:1.0 .
```

Запускаем контейнер.

```bash
docker container run --publish 80:80 --detach almsys/hello:1.0
```

Публикуем образ в Docker Hub.

```bash
docker login --username almsys
docker push almsys/hello:1.0
```

## Этап 7: Оркестрация (Docker Compose и Kubernetes)

Создаем файл `docker-compose.yml` для упрощенного запуска.

```yaml
version: "3"
services:
  hello:
    image: almsys/hello:1.0
    ports:
      - "80:80"
    restart: always
```

Запускаем приложение с помощью Docker Compose.

```bash
docker-compose up
```

Устанавливаем Minikube для создания локального кластера Kubernetes.

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo install minikube /usr/local/bin/
sudo snap install kubectl --classic
minikube start
```

Создаем манифест для развертывания в Kubernetes `k8s.yml`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
  replicas: 4
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: almsys/hello:1.0
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

Применяем манифест и проверяем развертывание.

```bash
kubectl apply -f k8s.yml
kubectl get pods
kubectl get svc
```

Тестируем приложение.

```bash
curl http://<CLUSTER-IP>:<PORT>
```

## Заключение

Этот проект прошел полный путь от простого скрипта до приложения, развернутого в оркестрируемом кластере, демонстрируя ключевые инструменты и практики современного DevOps.
```

Этот `README.md` теперь готов для размещения на GitHub. Он хорошо структурирован, код и команды правильно оформлены, что делает его удобным для чтения и изучения.
