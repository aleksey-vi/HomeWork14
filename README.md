# OTUS ДЗ 14 Ansible

## Домашнее задание

Первые шаги с Ansible
Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:
- необходимо использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible

### Установка Ansible

Все последующие действия выполняются на хостовой машине!!!


Схема выглядит так, что на хосте работает Ansible, который с помощью Vagrant открывает Vagrantfile и уже после запускается CentOS.

Версия Ansible =>2.4 требует для своей работы Python 2.6 или выше
Убедитесь что у Вас установлена нужная версия:

`` python -V ``

Далее следует сама установка:

```sudo apt install ansible```
После чего создаем каталог ``mkdir Ansible`` и переходим в него ``cd Ansible`` затем клонируем Vagrantfile
```git clone https://gist.github.com/lalbrekht/f811ce9a921570b1d95e07a7dbebeb1e```

Поднимите управляемый хост командой ``vagrant up`` и убедитесь что все
прошло успешно и есть доступ по ssh ``vagrant ssh nginx``

### Подготовка окружения

Для подключения к хосту nginx нам необходимо будет передать множество параметров - это особенность Vagrant. Узнать эти параметры можно с помощью ```vagrant ssh-config```

Создадим отедльную директорию под inventory-файл, и сам файл hosts.

    mkdir staging
    cd staging
    vi hosts

Содержимое файла hosts:

    [web]
    nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=/home/strayer/OTUS/dz14/Ansible/.vagrant/machines/nginx/virtualbox/private_key

И наконец убедимся, что Ansible может управлять нашим хостом. Сделать это можно с помощью команды:

    ansible -i /home/ed/otus/10/staging/hosts all -m ping  


### Настройка Ansible

Как видим нам придется каждый раз точно указывать наш инвентори файл и вписывать в него много информации. Это можно обойти используя ``ansible.cfg`` файл - прописав конфигурацию в нем. Для этого в текущем каталоге создадим файл ``ansible.cfg`` со следующим содержанием:

    nano ansible.cfg

 и вписать

    [defaults]
    inventory = /home/strayer/OTUS/dz14/Ansible/staging/hosts
    remote_user = vagrant
    host_key_checking = False
    retry_files_enabled = False

После этого убедимся еще раз, что Ansible может управлять нашим хостом. Сделать это можно с помощью команды:

    ansible nginx -m ping

Теперь, когда убедились, что у нас все подготовлено - установлен Ansible, поднят хост для теста и Ansible имеет к нему доступ, мы можем конфигурировать наш хост. Для начала воспользуемся Ad-Hoc командами и выполним некоторые удаленные команды на нашем хосте:

    ansible nginx  -m command -a "uname -r"

    ansible nginx -m systemd -a name=firewalld

Вывод будет большим но нас интересует конкретная строка

    "status": {
        ...
        "ActiveState": "inactive",
        ...

Установим пакет epel-release на хосты

    ansible nginx -m yum -a "name=epel-release state=present" -b

Напишем простой Playbook, который будет устанавливать пакет epel-release. Создадим файл epel.yml со следующим содержимым

    ---
    - name: Install EPEL Repo
    hosts: all
    become: true
    tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
      name: epel-release
      state: present

Проверим работу

    ansible-playbook epel.yml


### Домашка

За основу будущего playbook возьмем уже созданный нами файл ``epel.yml``, переименовав его в ``nginx.yml``. Добавим в него установку пакета NGINX. Измененный файл будет выглядеть так:
```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages

  ```

Обратите внимание - добавили Tags. Теперь можно вывести в консоль список тегов и выполнить, например, только установку NGINX. В нашем случае так, например, можно осуществлять его обновление.

Выведем в консоль все теги:

    ansible-playbook nginx.yml --list-tags

Запустим только установку NGINX:

    ansible-playbook nginx.yml -t nginx-package

Далее добавим шаблон для конфига NGINX и модуль, который будет
    копировать этот шаблон на хост:

      - name: NGINX | Create NGINX config file from template
      template:
      src: templates/nginx.conf.j2
      dest: /tmp/nginx.conf
      tags:
      - nginx-configuration
Сразу же пропишем в Playbook необходимую нам переменную. Нам нужно чтобы NGINX слушал на порту 8080:

    - name: NGINX | Install and configure NGINX
     hosts: nginx
     become: true
     vars:
     nginx_listen_port: 8080

Для самого шаблона нам необходимо создать папку `templates`

    mkdir templates

И файл `nginx.conf.j2`

    nano nginx.conf.j2

В него мы пропишем:
```
# {{ ansible_managed }}
events {
worker_connections 1024;
}

http {
server {
    listen       {{ nginx_listen_port }} default_server;
    server_name  default_server;
    root         /usr/share/nginx/html;

    location / {
    }
}
}
```
Результирующий файл `nginx.yml` будет выглядеть следующим образом:
```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```
Запускаем измененный playbook.

    ansible-playbook playbooks/nginx.yml

проверяем, что работает

http://192.168.11.150:8080/
