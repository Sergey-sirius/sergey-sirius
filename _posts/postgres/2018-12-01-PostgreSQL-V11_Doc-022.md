---
layout: post
title: Глава 22. Управление базами данных
description: ""
tags: [PostgreSQL, PostgreSQL_Book_11]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 22. Управление базами данных


Каждый работающий экземпляр сервера PostgreSQL обслуживает одну или несколько баз данных.
Поэтому базы данных представляют собой вершину иерархии SQL-объектов («объектов базы дан-
ных»). Данная глава описывает свойства баз данных, процессы создания, управления и удаления.
22.1. Обзор
База данных — именованная коллекция объектов SQL («объектов базы данных»). В целом, каждый
объект базы данных (таблицы, функции и т. д.) принадлежит одной и только одной базе данных.
(Правда есть несколько системных каталогов, например, pg_database, которые принадлежат всему
кластеру и доступны для каждой базы данных этого кластера.) Если точнее, база данных это набор
схем, которые включают в себя таблицы, функции и т. д. Таким образом, полная иерархия включает
в себя: сервер, базу данных, схему, таблицу (или иные типы объектов, к примеру, функции).
При подключении к серверу базы данных, клиент должен указать в параметрах подключения имя
базы данных, с которой нужно соединиться. Одно соединение не может иметь доступ более чем
к одной базе данных. Однако приложение не ограничено в количестве соединений к одной и той
же или разным базам данных. Базы данных разделены физически и контроль доступа осуществ-
ляется на уровне соединения. В случае, когда один экземпляр сервера PostgreSQL обслуживает
проекты или пользователей, которых необходимо изолировать друг от друга, рекомендуется раз-
мещать их в раздельных базах данных. В случае, когда проекты или пользователи взаимосвязаны
и должны иметь возможность использовать общие ресурсы, они должны размещаться в одной базе
данных, но, возможно, в раздельных схемах. Схемы — в чистом виде логическая структура, и кто
к чему может получить доступ управляется системой привилегий. Более подробная информация
по управлению схемами приведена в Разделе 5.8.
Базы данных создаются командой CREATE DATABASE (см. Раздел 22.2), а удаляются командой DROP
DATABASE (см. Раздел 22.5). Список существующих баз данных можно посмотреть в системном ка-
талоге pg_database, например,
SELECT datname FROM pg_database;
Метакоманда \l или ключ -l командной строки приложения psql также позволяют вывести список
существующих баз данных.
Примечание
Стандарт SQL называет базы данных «каталогами», но на практике у них нет отличий.
22.2. Создание базы данных
Для создания базы данных сервер PostgreSQL должен быть развёрнут и запущен (см. Раздел 18.3).
База данных создаётся SQL-командой CREATE DATABASE:
CREATE DATABASE имя;
где имя подчиняется правилам именования идентификаторов SQL. Текущий пользователь автома-
тически назначается владельцем. Владелец может удалить свою базу, что также приведёт к уда-
лению всех её объектов, в том числе, имеющих других владельцев.
Создание баз данных это привилегированная операция. Как предоставить права доступа, описано
в Разделе 21.2.
Поскольку для выполнения команды CREATE DATABASE необходимо подключение к серверу базы
данных, возникает вопрос как создать самую первую базу данных. Первая база данных всегда со-
здаётся командой initdb при инициализации пространства хранения данных (см. Раздел  18.2.)
598Управление базами данных
Эта база данных называется postgres.Далее для создания первой «обычной» базы данных можно
подключиться к postgres.
Вторая база данных template1,также создаётся во время инициализации кластера. При каждом
создании новой базы данных в рамках кластера по факту производится клонирование шаблона
template1. При этом любые изменения сделанные в template1 распространяются на все создан-
ные впоследствии базы данных. Следует избегать создания объектов в template1, за исключением
ситуации, когда их необходимо автоматически добавлять в новые базы. Более подробно в Разде-
ле 22.3.
Для удобства, есть утилита командной строки для создания баз данных, createdb.
createdb dbname
Утилита createdb не делает ничего волшебного, она просто подключается к базе данных postgres
и выполняет ранее описанную SQL-команду CREATE DATABASE. Подробнее о её вызове можно узнать
в createdb. Обратите внимание, что команда createdb без параметров создаст базу данных с име-
нем текущего пользователя.
Примечание
Глава 20 содержит информацию о том, как ограничить права на подключение к задан-
ной базе данных.
Иногда необходимо создать базу данных для другого пользователя и назначить его владельцем,
чтобы он мог конфигурировать и управлять ею. Для этого используйте одну из следующих команд:
CREATE DATABASE имя_базы OWNER имя_роли;
из среды SQL, или:
createdb -O имя_роли имя_базы
из командной строки ОС. Лишь суперпользователь может создавать базы данных для других (для
ролей, членом которых он не является).
22.3. Шаблоны баз данных
По факту команда CREATE DATABASE выполняет копирование существующей базы данных. По умол-
чанию копируется стандартная системная база template1. Таким образом, template1 это шаблон,
на основе которого создаются новые базы. Если добавить объекты в template1, то впоследствии
они будут копироваться в новые базы данных. Это позволяет внести изменения в стандартный на-
бор объектов. Например, если в template1 установить процедурный язык PL/Perl, то он будет до-
ступен в новых базах без дополнительных действий.
Также существует вторая системная база template0.При инициализации она содержит те же са-
мые объекты, что и template1, предопределённые в рамках устанавливаемой версии PostgreSQL.
Не нужно вносить никаких изменений в template0 после инициализации кластера. Если в команде
CREATE DATABASE указать на необходимость копирования template0 вместо template1, то на выходе
можно получить «чистую» пользовательскую базу данных без изменений, внесённых в template1.
Это удобно, когда производится восстановление из дампа данных с помощью утилиты pg_dump:
скрипт дампа лучше выполнять в чистую базу, во избежание каких-либо конфликтов с объектами,
которые могли быть добавлены в template1.
Другая причина, для копирования template0 вместо template1 заключается в том, что можно ука-
зать новые параметры локали и кодировку при копировании template0, в то время как для копий
template1 они не должны меняться. Это связано с тем, что template1 может содержать данные в
специфических кодировках и локалях, в отличие от template0.
Для создания базы данных на основе template0, используйте:
599Управление базами данных
CREATE DATABASE dbname TEMPLATE template0;
из среды SQL, или:
createdb -T template0 dbname
из командной строки ОС.
Можно создавать дополнительные шаблоны баз данных, и, более того, можно копировать любую
базу данных кластера, если указать её имя в качестве шаблона в команде CREATE DATABASE. Важно
понимать, что это (пока) не рассматривается в качестве основного инструмента для реализации
возможности «COPY DATABASE». Важным является то, что при копировании все сессии к копиру-
емой базе данных должны быть закрыты. CREATE DATABASE выдаст ошибку, если есть другие под-
ключения; во время операции копирования новые подключения к этой базе данных не разрешены.
В таблице pg_databaseесть два полезных флага для каждой базы данных: столбцы datistemplate
и datallowconn. datistemplate указывает на факт того, что база данных может выступать в каче-
стве шаблона в команде CREATE DATABASE. Если флаг установлен, то для пользователей с правом
CREATEDB клонирование доступно; если флаг не установлен, то лишь суперпользователь и владелец
базы данных могут её клонировать. Если datallowconn не установлен, то новые подключения к
этой базе не допустимы (однако текущие сессии не закрываются при сбросе этого флага). База
template0 обычно помечена как datallowconn = false для избежания любых её модификаций. И
template0, и template1 всегда должны быть помечены флагом datistemplate = true.
Примечание
template1 и template0 не выделены как-то особенно, кроме того факта, что template1
используется по умолчанию в команде CREATE DATABASE. Например, можно удалить
template1 и безболезненно создать заново из template0. Это можно посоветовать в слу-
чае, если template1 был замусорен. (Чтобы удалить template1, необходимо сбросить
флаг pg_database.datistemplate = false.)
База данных postgres также создаётся при инициализации кластера. Она использу-
ется пользователями и приложениями для подключения по умолчанию. Представля-
ет собой всего лишь копию template1, и может быть удалена и повторно создана при
необходимости.
22.4. Конфигурирование баз данных
Обратившись к Главе  19 можно выяснить, что сервер PostgreSQL имеет множество параметров
конфигурации времени исполнения. Можно выставить специфичные для базы данных значения
по умолчанию.
Например, если по какой-то причине необходимо выключить GEQO оптимизатор в какой-то из
баз, то можно, либо выключить его для всех баз данных одновременно, либо убедиться, что все
клиенты заботятся об этом, выполняя команду SET geqo TO off. Для того чтобы это действовало
по умолчанию в конкретной базе данных, необходимо выполнить команду:
ALTER DATABASE mydb SET geqo TO off;
Установка сохраняется, но не применяется тотчас. В последующих подключениях к этой базе дан-
ных, эффект будет таким, будто перед началом сессии была выполнена команда SET geqo TO off;.
Стоит обратить внимание, что пользователь по-прежнему может изменять этот параметр во время
сессии; ведь это просто значение по умолчанию. Чтобы сбросить такое установленное значение,
используйте ALTER DATABASE dbname RESET varname.
22.5. Удаление базы данных
Базы данных удаляются командой DROP DATABASE:
600Управление базами данных
DROP DATABASE имя;
Лишь владелец базы данных или суперпользователь могут удалить базу. При удалении также уда-
ляются все её объекты. Удаление базы данных это необратимая операция.
Невозможно выполнить команду DROP DATABASE пока существует хоть одно подключение к задан-
ной базе. Однако можно подключиться к любой другой, в том числе и template1. template1 мо-
жет быть единственной возможностью при удалении последней пользовательской базы данных
кластера.
Также существует утилита командной строки для удаления баз данных dropdb:
dropdb dbname
(В отличие от команды createdb утилита не использует имя текущего пользователя по умолча-
нию).
22.6. Табличные пространства
Табличные пространства в PostgreSQL позволяют администраторам организовать логику разме-
щения файлов объектов базы данных в файловой системе. К однажды созданному табличному про-
странству можно обращаться по имени на этапе создания объектов.
Табличные пространства позволяют администратору управлять дисковым пространством для ин-
сталляции PostgreSQL. Это полезно минимум по двум причинам. Во-первых, это нехватка места в
разделе, на котором был инициализирован кластер и невозможность его расширения. Табличное
пространство можно создать в другом разделе и использовать его до тех пор, пока не появится
возможность переконфигурирования системы.
Во-вторых, табличные пространства позволяют администраторам оптимизировать производитель-
ность согласно бизнес-процессам, связанным с объектами базы данных. Например, часто исполь-
зуемый индекс можно разместить на очень быстром и надёжном, но дорогом SSD-диске. В то же
время таблица с архивными данными, которые редко используются и скорость к доступа к ним не
важна, может быть размещена в более дешёвом и медленном хранилище.
Предупреждение
Несмотря на внешнее размещение относительно основного каталога хранения данных
PostgreSQL, табличные пространства являются неотъемлемой частью кластера и не
могут трактоваться, как самостоятельная коллекция файлов данных. Они зависят от
метаданных, расположенных в главном каталоге, и потому не могут быть подключены
к другому кластеру, или копироваться по отдельности. Также, в случае потери таблич-
ного пространства (при удалении файлов, сбое диска и т. п.), кластер может оказаться
недоступным или не сможет запуститься. Таким образом, при размещении таблично-
го пространства во временной файловой системе, например, в RAM-диске, возникает
угроза надёжности всего кластера.
Для создания табличного пространства используется команда CREATE TABLESPACE, например::
CREATE TABLESPACE fastspace LOCATION '/ssd1/postgresql/data';
Каталог должен существовать, быть пустым и принадлежать пользователю ОС, под которым за-
пущен PostgreSQL. Все созданные впоследствии объекты, принадлежащие целевому табличному
пространству, будут храниться в файлах расположенных в этом каталоге. Каталог не должен раз-
мещаться на съёмных или устройствах временного хранения, так как кластер может перестать
функционировать из-за потери этого пространства.
Примечание
Обычно нет смысла создавать более одного пространства на одну логическую файло-
вую систему, так как нет возможности контролировать расположение отдельных фай-
601Управление базами данных
лов в файловой системе. Однако PostgreSQL не накладывает никаких ограничений в
этом отношении, и более того, напрямую не заботится о точках монтирования файло-
вой системы. Просто осуществляется хранение файлов в указанных каталогах.
Создавать табличное пространство должен суперпользователь базы данных, но после этого можно
разрешить обычным пользователям его использовать. Для этого необходимо предоставить приви-
легию CREATE на табличное пространство.
Таблицы, индексы и целые базы данных могут храниться в отдельных табличных пространствах.
Для этого пользователь с правом CREATE на табличное пространство должен указать его имя в
качестве параметра соответствующей команды. Например, далее создаётся таблица в табличном
пространстве space1:
CREATE TABLE foo(i int) TABLESPACE space1;
Как вариант, используйте параметр default_tablespace:
SET default_tablespace = space1;
CREATE TABLE foo(i int);
Когда default_tablespace имеет значение отличное от пустой строки, он будет использоваться
неявно в качестве значения параметра TABLESPACE в командах CREATE TABLE и CREATE INDEX, если
в самой команде не задано иное.
Существует параметр temp_tablespaces, который указывает на размещение временных таблиц и
индексов, а также файлов, создаваемых, например, при операциях сортировки больших наборов
данных. Предпочтительнее, в качестве значения этого параметра, указывать не одно имя, а спи-
сок из нескольких табличных пространств. Это поможет распределить нагрузку, связанную с вре-
менными объектами, по различным табличным пространствам. При каждом создании временного
объекта будет случайным образом выбираться имя из указанного списка табличных пространств.
Табличное пространство, связанное с базой данных, также используется для хранения её систем-
ных каталогов. Более того, это табличное пространство используется по умолчанию для таблиц,
индексов и временных файлов, создаваемых в базе данных, если не указано иное в выражении
TABLESPACE, или переменной default_tablespace, или temp_tablespaces (соответственно). Если
база данных создана без указания конкретного табличного пространства, то используется про-
странство, к которому принадлежит копируемый шаблон.
При инициализации кластера автоматически создаются два табличных пространства. Табличное
пространство pg_global используется для общих системных каталогов. Табличное пространство
pg_default используется по умолчанию для баз данных template1 и template0 (в свою очередь,
также является пространством по умолчанию для других баз данных, пока не будет явно указано
иное в выражении TABLESPACE команды CREATE DATABASE).
После создания, табличное пространство можно использовать в рамках любой базы данных, при
условии, что у пользователя имеются необходимые права. Это означает, что табличное простран-
ство невозможно удалить до тех пор, пока не будут удалены все объекты баз данных, использую-
щих это пространство.
Для удаления пустого табличного пространства используйте команду DROP TABLESPACE.
Чтобы получить список табличных пространств можно сделать запрос к системному каталогу
pg_tablespace, например,
SELECT spcname FROM pg_tablespace;
Метакоманда \db утилиты psql также позволяет отобразить список существующих табличных про-
странств.
PostgreSQL использует символические ссылки для упрощения реализации табличных про-
странств. Это означает, что табличные пространства могут использоваться только в системах,
поддерживающих символические ссылки.
602Управление базами данных
Каталог $PGDATA/pg_tblspc содержит символические ссылки, которые указывают на внешние таб-
личные пространства кластера. Хоть и не рекомендуется, но возможно регулировать табличные
пространства вручную, переопределяя эти ссылки. Ни при каких обстоятельствах эти операции
нельзя проводить, пока запущен сервер баз данных. Обратите внимание, что в версии PostgreSQL
9.1 и более ранних также необходимо обновить информацию в pg_tablespace о новых расположе-
ниях. (Если это не сделать, то pg_dump будет продолжать выводить старые расположения таблич-
ных пространств.)
