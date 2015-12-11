.. sectionauthor:: Иван Ковалев <ivan.kovalev@nextgis.ru>
.. sectionauthor:: Артём Светлов <artem.svetlov@nextgis.ru>
.. sectionauthor:: Дмитрий Барышников <dmitry.baryshnikov@nextgis.ru>

.. _ngb_install:

Установка
=========

Данная инструкция проверена и работает в Ubuntu Server 12.04 LTS и выше.
Для установки системы необходим Python 2.7.

Подготовка базы данных
----------------------

Подключить репозиторий ubuntugis (см. `поддерживаемые
дистрибутивы <http://trac.osgeo.org/ubuntugis/wiki/SupportedDistributions>`_):

**Для ubuntu server 12.04**

.. code:: bash

    sudo apt-get install software-properties-common python-software-properties
    sudo apt-add-repository ppa:ubuntugis/ppa

**Для ubuntu server 14.04**

.. code:: bash

    sudo apt-add-repository ppa:ubuntugis/ubuntugis-unstable
    sudo apt-get update
    sudo apt-get upgrade

Установить PostgreSQL:

.. code:: bash

    sudo apt-get install postgresql

Создаем пользователя:

.. code:: bash

    sudo -u postgres createuser ngbio_admin -P -e

после ввода пароля три раза говорим 'n'.

Создаем базу, в которую будет развернута NextGIS Bio:

.. code:: bash

    sudo -u postgres createdb -O ngbio_admin --encoding=UTF8 db_ngbio
    sudo nano /etc/postgresql/9.3/main/pg_hba.conf

Отредактируем строку ``local   all   all   peer`` и приведём её к виду:
``local   all   all   md5``

Не забудьте перезапустить сервис базы:

.. code:: bash

    sudo service postgresql restart

Установить PostGIS:

.. code:: bash

    sudo apt-cache search postgis

В полученном списке найдите пакет, подходящий для вашей версии
PostgreSQL, его имя должно иметь вид
postgresql-{version}-postgis-{version} и установите его:

.. code:: bash

    sudo apt-get install postgresql-9.3-postgis-2.1
    sudo -u postgres psql -d db_ngw -c 'CREATE EXTENSION postgis;'
    sudo -u postgres psql -d db_ngw -c 'ALTER TABLE geometry_columns OWNER TO ngw_admin;'
    sudo -u postgres psql -d db_ngw -c 'ALTER TABLE spatial_ref_sys OWNER TO ngw_admin;'
    sudo -u postgres psql -d db_ngw -c 'ALTER TABLE geography_columns OWNER TO ngw_admin;'

После этих операций будут созданы БД PostgreSQL с установленным в ней
:term:`PostGIS` и пользователь :abbr:`БД (база данных)`, который станет ее владельцем, а также 
таблиц ``geometry_columns``, ``georgaphy_columns``, ``spatial_ref_sys``.

Убедитесь, что функции PostGIS появились в базе:

.. code:: bash

    psql -d db_ngw -U ngw_admin -c "SELECT PostGIS_Full_Version();"

Если вы разворачиваете систему на чистом сервере, и вам надо сделать ещё
одну базу PostGIS для хранения данных, то включаем доступ к ней из сети

.. code:: bash

    sudo su - postgres
    nano /etc/postgresql/9.3/main/pg_hba.conf
    делаем строку host    all    all    127.0.0.1/32    md5

    nano /etc/postgresql/9.3/main/postgresql.conf
    делаем строку listen_addresses='*', и расскоментируем её.

.. code:: bash

    sudo service postgresql restart

Подготовка базового ПО
----------------------

Установить pip:

.. code:: bash

    sudo apt-get install python-pip

Установить virtualenv:

.. code:: bash

    sudo pip install virtualenv

Установить дополнительные инструменты:

.. code:: bash

    sudo apt-get install python-mapscript git libgdal-dev python-dev g++ \
    libxml2-dev libxslt1-dev gdal-bin

В случае разработки NextGIS Bio может понадобится регистрация ключей.
**Для большинства случаев ключи генерировать не нужно!** Это необходимо при
разработке.

Генерируем ключи для работы с GitHub (копируем и вставляем ключ в
настройки пользователя GitHub в `разделе SSH keys <https://github.com/settings/ssh>`_):

.. code:: bash

    mkdir ~/.ssh
    cd ~/.ssh
    ssh-keygen -t rsa -C "your@email.com"
    ssh-add ~/.ssh/id_rsa
    cat id_rsa.pub
    cd ~

Если включена двух-факторная авторизация, понадобится еще:

* `Закэшировать пароль <https://help.github.com/articles/caching-your-github-password-in-git/#platform-linux>`_
* `Сгенерировать access token <https://github.com/settings/applications#personal-access-tokens>`_
  и использовать его вместо пароля

Подготовка к установке NextGIS Bio
----------------------------------

Создаём необходимые директории:

.. code:: bash

    mkdir -p ~/ngbio
    cd ~/ngw

Клонируем репозиторий:

.. code:: bash

    git clone https://github.com/nextgis/nextgisbio.git

Создаем виртуальное окружение virtualenv в папке ``~/ngbio/env`` (папка
создастся сама после выполнения команды):

.. code:: bash

    virtualenv --no-site-packages env

Установка NextGIS Bio
---------------------

Устанавливаем пакет NextGIS Bio в режиме разработки, при этом будут
установлены все необходимые пакеты:

.. code:: bash

    env/bin/pip install -e ./nextgisbio


Конфигурационный файл NextGIS Web
---------------------------------

Пример конфигурационного файла доступен в репозитории `здесь <https://github.com/nextgis/nextgisbio/blob/master/development.example.ini>`_). В этот
текстовый файл нужно внести изменения в соответствии со своим
окружением. Основное изменение касается строки подключения к базе данных:

.. code:: python

    sqlalchemy.url = postgresql+psycopg2://{USER_NAME}:{USER_PASSWORD}@0.0.0.0/{DATABASE_NAME}

где вместо {USER_NAME} надо подставить имя пользователя базы данных,
{USER_PASSWORD} - пароль пользователя, {DATABASE_NAME} - название базы данных.

Инициализация БД
----------------

Инициализация БД выполняется следующим образом:

.. code:: bash

    env/bin/initialize_ngbio_db development.ini

Следует отметить, что эта команда удаляет таблицы при их наличии в БД.

Обновление ПО
-------------

Для обновления NextGIS Bio необходимо выполнить команду:

.. code:: bash

    cd ~/ngbio/nextgisbio
    git pull

Если в файле setup.py добавились зависимости, то следует выполнить:

.. code:: bash

    env/bin/pip install -e ./nextgisbio

Если изменилась структура БД то следует выполнить:

.. code:: bash

    # Внимание! Существующие таблицы удалятся!
    env/bin/initialize_ngbio_db development.ini

После выполнения команд необходимо перезапустить NextGIS Bio либо перезапуском
pserve, либо веб-сервера с модулем uWSGI.
