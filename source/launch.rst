.. sectionauthor:: Иван Ковалев <ivan.kovalev@nextgis.ru>
.. sectionauthor:: Артём Светлов <artem.svetlov@nextgis.ru>

.. _ngb_launch:
    
Запуск
======

Запуск через Pserve
-------------------

Для запуска NextGIS Bio через Pserve необходимо выполнить команду:

.. code:: bash

    env/bin/pserve development.ini

Для автоматического запуска NextGIS Bio при загрузке операционной системы
необходимо отредактировать пользовательский скрипт автозапуска:

.. code:: bash

    sudo nano /etc/rc.local

и добавить в него строку:

.. code:: bash

    {ABSOLUTE_PATH_TO_ENV}/env/bin/pserve --daemon  /home/zadmin/ngw/production.ini

где {ABSOLUTE_PATH_TO_ENV} - составляющая абсолютного пути до вашего виртуального окружения.

В промышленной эксплуатации нужно использовать не pserve, а :ref:`uWSGI <ngb_uwsgi>`.

Для проверки работоспособности необходимо в веб-браузере набрать:

::

    http://0.0.0.0:6543

Должно открыться окно авторизации.

.. note: При запуске pserve через supervisor необходимо добавить настройку environment=LANG=ru_RU.UTF-8 для поддержки русскихимен в названии загружаемых файлов.

.. _ngb_uwsgi:

Запуск через uWSGI
------------------

Для начал необходимо установить uWSGI:

.. code:: bash

   user@ubuntu:~/ngw$ source env/bin/activate
   (env)user@ubuntu:~/ngw$ pip install uwsgi
    
или через системный сервис:

.. code:: bash

   apt-get install uwsgi uwsgi-plugin-python uwsgi-emperor
 
К существующему конфигурационном ini-файлу paste добавляем секцию
``uwsgi``

::

    [uwsgi]
    module = nextgisbio.uwsgiapp
    env = PASTE_CONFIG=%p

При использовании FreeBSD может потребоваться отключить WSGI file
wrapper, так как он иногда работает некорректно. Для этого в этой же
секции:

::

    env = WSGI_FILE_WRAPPER=no
    
Для запуска uWSGI через unix socet секция должна иметь следующий вид:
    
::
    
    [uwsgi]
    home = /home/ngbio/ngbio/env
    socket = /home/ngbio/uwsgi/ngbio
    protocol=uwsgi
    chmod-socket=777
    master = true
    processes = 8
    threads = 4
    logto = /home/ngbio/logs/ngbio.log
    log-slow = 1000
    paste = config:%p
    paste-logger = %p
    env=LANG=ru_RU.UTF-8

.. note:: 
   Соответсвующие папки должны быть созданы. Для работы локали 
   (LANG=ru_RU.UTF-8) необходимо что бы в системе имелись соответсвующие файлы 
   (locale -a). Если локали нет, то ее необходимо добавить (locale-gen 
   ru_RU.utf8). Так же рекомендуется установить локаль системной (update-locale 
   LANG=ru_RU.UTF-8).

Далее в зависимости от того, какой интерфейс требуется на выходе от
uwsgi. Тут есть некоторая путаница, связаная с тем, что uwsgi - это
одновременно и протокол и программа. Ниже речь идет именно о протоколе.

HTTP:

::

    socket = host:port | :port
    protocol = http

uWSGI:

::

    socket = host:port | :port | /path/to/socket
    protocol = uwsgi

FastCGI:

::

    socket = host:port | :port | /path/to/socket
    protocol = fastcgi

Знака \| в конфиге быть не должно, надо написать например так:

::

    socket =  :6543    

При использовании сокета в файловой системе права на него могут быть
выставлены через параметр chmod:

::

    chmod = 777

Количество процессов задается параметром ``workers``, а количество
потоков в процессе - параметром ``thread``. В примере ниже будет
запущено 2 процесса с 4 потоками в каждом:

::

    workers = 2
    threads = 4

Вариант с отдельным процессами более безопасный, но и более
ресурсоемкий.

Запуск uwsgi осуществляется командой ``uwsgi file.ini``, причем все
переменные могут быть так же переопределены из командной строки,
например так: ``uwsgi --workers=8 file.ini``. В таком же виде uwsgi
можно запускать и через supervisor, например так:

::

    [program:nextgisweb]
    command = /path/to/uwsgi /path/to/file.ini
    
supervisor + uwsgi
~~~~~~~~~~~~~~~~~~

Для запуска через supervisor + uWSGI без использования веб-сервера конфигурация 
должна иметь следующий вид:
    
::    

   [uwsgi]
   module = nextgisbio.uwsgiapp
   lazy = yes
   env = PASTE_CONFIG=%p
   env = PATH=/home/ngw_admin/ngw/env/bin:/bin:/usr/sbin:/usr/bin
   env = LANG=ru_RU.UTF-8
   virtualenv = /home/ngw_admin/ngw/env
   protocol = http
   socket = :8080
   workers = 4 # количество потоков обработки подключений
   limit-post = 4831838208 # максимальный размер файла

Конфигурация supervisor может иметь следующий вид:
    
::
    
    [program:ngw]
    command = /home/ngbio/ngbio/env/bin/uwsgi /home/ngbio/ngbio/production.ini
    user = ngw_admin
    environment=LANG=ru_RU.UTF-8

nginx + uwsgi
~~~~~~~~~~~~~

Для запуска при помощи nginx в файл конфигурации сервера необходимо добавить 
следующие строки.

В случае запуска uWSGI на TCP порту:    

:: 

    location /path_to_ngw_instance/ {
        include uwsgi_params;
	    uwsgi_pass 127.0.0.1:6543;
    }
    
    
В случае запуска uWSGI на unix порту:    

:: 

    location /path_to_ngbio_instance/ {
        include uwsgi_params;
        uwsgi_pass unix:///home/ngbio/uwsgi/ngbio;
    }


nginx + uwsgi (вариант 2)
~~~~~~~~~~~~~~~~~~~~~~~~~

Создаем файл с настройками:  

::

	sudo touch /etc/nginx/sites-available/ngbio.conf

содержание:  

::

     server {
          listen                 6555;
          
          location / {
            uwsgi_read_timeout 600s; #для больших файлов необходимо поставить большее время
            uwsgi_send_timeout 600s;

            include            uwsgi_params;
            uwsgi_pass         unix:/tmp/ngbio.socket;

            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
            
            proxy_buffer_size 64k; # для больших файлов увеличиваем буфер
            proxy_max_temp_file_size 0; # и размер временного файла ставим без огранчиений
            proxy_buffers 8 32k;
        }
    }
