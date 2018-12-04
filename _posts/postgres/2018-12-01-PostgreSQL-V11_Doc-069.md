---
layout: post
title: Глава 69. Объявление и начальное содержимое системных каталогов
description: ""
tags: [PostgreSQL, PostgreSQL_Book_11]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 69. Объявление и начальное содержимое системных каталогов

В PostgreSQL используется множество разных системных каталогов для учёта информации о су-
ществовании и свойствах объектов базы, например, таблиц и функций. Физически системный ка-
талог не отличается от простой таблицы, но серверный код на C знает структуру и характеристики
каждого каталога и может работать с ним на низком уровне. Поэтому, например, не стоит пытать-
ся изменять структуру каталога «на лету»; это нарушит встроенные в код предположения о том,
как располагаются строки в каталоге. Однако структура каталога может меняться при переходе
с одной основной версии на другую.
Структуры каталогов объявляются в специально оформленных заголовочных файлах C в каталоге
src/include/catalog/ дерева исходного кода. В частности, для каждого каталога имеется заголо-
вочный файл, названный по имени каталога (например, pg_class.h для pg_class) и определяющий
набор столбцов в этом каталоге, а также другие основные свойства, например, его OID. К другим
важным файлам, задающим структуру каталога, относится indexing.h, определяющий, какие ин-
дексы присутствуют во всех системных каталогах, и toasting.h, определяющий таблицы TOAST
для каталогов, которым они нужны.
Со многими каталогами связаны исходные данные, которые должны быть загружены в них на ста-
дии «начальной загрузки» initdb, чтобы система оказалась в состоянии, когда она сможет выпол-
нять команды SQL. (Например, pg_class.h должен содержать запись, ссылающуюся на этот же
каталог, и перечисление всех остальных системных каталогов и индексов.) Эти исходные данные
задаются в редактируемой форме в файлах, которые также находятся в каталоге src/include/
catalog/. Например, в pg_proc.dat описываются все исходные строки, которые должны быть
вставлены в каталог pg_proc.
Чтобы создать файлы каталогов и загрузить в них эти исходные данные, серверный процесс, рабо-
тающий в режиме начальной загрузки, считывает файл BKI (Backend Interface, Серверный интер-
фейс), содержащий команды и исходные данные. Файл postgres.bki, используемый в этом режи-
ме, конструируется из вышеупомянутых заголовочных файлов и файлов данных при сборке дис-
трибутива PostgreSQL Perl-скриптом genbki.pl. Хотя postgres.bki привязан к определённому вы-
пуску PostgreSQL, он является платформонезависимым и устанавливается в подкаталог share де-
рева инсталляции.
Скрипт genbki.pl также генерирует производный заголовочный файл для каждого каталога, на-
пример pg_class_d.h для каталога pg_class. Этот файл содержит автоматически генерируемые
макроопределения и может содержать другие макросы, определения перечислений и т. п., кото-
рые могут быть полезны для клиентского кода, читающего определённый каталог.
Большинству разработчиков Postgres нет необходимости иметь дело непосредственно с файлом
BKI, но для практически любой нетривиальной доработки потребуется модификация заголовочных
файлов и/или файлов с исходными данными каталога. В продолжении этой главы рассказывается
об этом, а также для полноты описывается формат файла BKI.
69.1. Правила объявления системных каталогов
Ключевой частью заголовочного файла каталога является описание структуры на C, определяю-
щее вид каждой строки каталога. Оно начинается с макроса CATALOG, который, если говорить о
компиляторе C, является просто сокращённой записью typedef struct FormData_имя_каталога.
Каждое поле в этой структуре порождает столбец каталога. Поля можно дополнить макросами
свойств BKI, объявленными в genbki.h. Например, для поля можно задать значение по умолчанию
или указать, допускается ли в нём NULL. Строку CATALOG можно также дополнить некоторыми
другими макросами свойств BKI, объявленными в genbki.h и определяющими другие свойства ка-
талога в целом, например, содержит ли он OID (по умолчанию каталоги содержат OID).
Код кеша системного каталога (и в принципе почти весь код, манипулирующий каталогом) предпо-
лагает, что имеющие постоянный размер части всех кортежей системных каталогов присутствуют
2181Объявление и начальное содер-
жимое системных каталогов
фактически, так как он отображает на них объявления структуры на C. Таким образом, все поля пе-
ременной длины и поля, принимающие NULL, должны располагаться в конце, и обращаться к ним
как к полям структуры нельзя. Например, если присвоить полю pg_type.typrelid значение NULL,
обращение в каком-либо месте кода к typetup->typrelid (или, что ещё хуже, к полю typetup-
>typelem, следующему за typrelid) будет некорректным. Это приведёт к случайным ошибкам или
даже нарушениям сегментации.
В качестве частичной защиты от ошибок такого типа поля переменной длины или поля, прини-
мающие NULL, следует скрыть от компилятора C. Это реализуется посредством обёртки #ifdef
CATALOG_VARLEN ... #endif (где CATALOG_VARLEN — символ, который всегда будет неопределённым).
Это не позволяет коду на C беспрепятственно обращаться к полям, которые могут отсутствовать
или располагаться по некоторому другому смещению. В качестве дополнительной меры, препят-
ствующей созданию некорректных строк, мы требуем, чтобы все столбцы, которые не должны
принимать NULL, помечались соответствующим образом в pg_attribute. Код начальной загруз-
ки автоматически пометит столбцы каталога как NOT NULL, если они имеют фиксированную дли-
ну и перед ними нет столбцов, принимающих NULL. Там, где это правило применяется некор-
ректно, можно исправить пометку, добавив дополнительные указания BKI_FORCE_NOT_NULL или
BKI_FORCE_NULL. Но заметьте, что ограничения NOT NULL контролируются только исполнителем
запросов; на уровне кода C они не действуют, поэтому создавать или изменять строки каталога
вручную нужно так же аккуратно.
Код клиентской части не должен включать никакие заголовочные файлы каталогов pg_xxx.h, так
как эти файлы могут содержать код на C, который не будет компилироваться вне кода сервера.
(Обычно это происходит из-за того, что эти файлы также содержат объявления функций в файлах
src/backend/catalog/.) Вместо этого клиентский код может включить соответствующий сгенери-
рованный заголовок pg_xxx_d.h с определениями различных OID и другими данными, которые
могут быть полезны на стороне клиента. Если вам нужно, чтобы макросы или другой код в заго-
ловочных файлах каталогов были видимы в клиентском коде, заключите соответствущую секцию
в условие #ifdef EXPOSE_TO_CLIENT_CODE ... #endif, чтобы genbki.pl скопировал эту секцию в за-
головок pg_xxx_d.h.
Некоторые каталоги настолько основополагающие, что их нельзя создать даже командой BKI
create, которая используется для большинства каталогов, так как эта команда должна записать
информацию, описывающую новый каталог, в эти базовые каталоги. Они называются каталогами
начальной загрузки и для определения их требуется много дополнительные действий: вы долж-
ны вручную подготовить соответствующие записи для них в предварительно загружаемых данных
pg_class и pg_type, и эти записи потребуется модифицировать при последующих изменениях в
структуре каталога. (Каталогам начальной загрузки также нужны предварительно загруженные
записи в pg_attribute, но, к счастью, сейчас с этим управляется скрипт genbki.pl.) По возмож-
ности избегайте включения новых каталогов в категорию каталогов начальной загрузки.
69.2. Исходные данные системных каталогов
Для каждого каталога, с которым связаны вручную создаваемые исходные данные, (не все катало-
ги такие) имеется соответствующий файл .dat, содержащий эти данные в редактируемом формате.
69.2.1. Формат файла данных
Каждый файл .dat содержит описания структур данных Perl, в результате вычисления которых
(функцией eval) в памяти формируется структура данных, состоящая из массива хеш-ссылок, соот-
ветствующих каждой строке каталога. Немного модифицированная выдержка из pg_database.dat
иллюстрирует основные моменты:
[
# Здесь мог быть комментарий.
{ oid => '1', oid_symbol => 'TemplateDbOid',
descr => 'database\'s default template',
datname => 'template1', datdba => 'PGUID', encoding => 'ENCODING',
2182Объявление и начальное содер-
жимое системных каталогов
datcollate => 'LC_COLLATE', datctype => 'LC_CTYPE', datistemplate => 't',
datallowconn => 't', datconnlimit => '-1', datlastsysoid => '0',
datfrozenxid => '0', datminmxid => '1', dattablespace => '1663',
datacl => '_null_' },
]
Замечания:
• Общий формат файла: открывающая квадратная скобка, один или более наборов фигурных
скобок, каждый из которых представляет строку каталога, и закрывающая квадратная скобка.
После каждой закрывающей фигурной скобки должна идти запятая.
• В каждой строке каталога записываются разделённые запятыми пары ключ => значение.
В качестве ключа принимаются имена столбцов каталога, а также ключи метаданных oid,
oid_symbol и descr. (Использование oid и oid_symbol описывается в Подразделе 69.2.2. В
descr задаётся строка с описанием объекта, которое будет вставлено в pg_description или
pg_shdescription.) Ключи метаданных могут опускаться, но ключ для каждого столбца ката-
лога должен присутствовать, если только в файле .h данного каталога для столбца не задано
значение по умолчанию.
• Все значения должны заключаться в апострофы. Апострофы внутри значений экранируют-
ся обратной косой чертой. Обратные косые черты в данных могут, но не обязательно должны
дублироваться; это соответствует правилам Perl по оформлению простых строковых констант.
Заметьте, что обратные косые черты, фигурирующие в данных, будут обрабатываться скане-
ром исходных данных как символы экранирования, согласно тем же правилам записи строко-
вых констант (см. Подраздел 4.1.2.2); например, \t преобразуется в символ табуляции. Если
вы хотите получить именно обратную косую черту в окончательном значении, вам надо будет
написать четыре этих символа: Perl отбрасывает два и оставляет \\ сканеру исходных данных.
• Значения NULL представляются как _null_. (Заметьте, что создать значение с именно такой
строкой невозможно.)
• Комментарии предваряются знаком # и должны размещаться в отдельных строках.
• Для большей наглядности значения полей, выражающие OID других записей каталога, могут
быть представлены именами, а не только числовыми кодами OID. Об этом рассказывается в
Подразделе 69.2.3.
• Так как хеши являются неупорядоченной структурой данных, порядок полей и расположе-
ние строк не имеют семантической значимости. Однако для поддержания согласованного
представления мы установили несколько правил, которые применяет скрипт форматирования
reformat_dat_file.pl:
• В каждой паре фигурных скобок сначала идут поля метаданных oid, oid_symbol и descr, в
этом же порядке, а затем собственные поля каталога в определённом для них порядке.
• Переводы строк при необходимости вставляются между полями для ограничения длины
строки 80 символами, если это возможно. Перевод строки также вставляется между поля-
ми метаданных и обычными полями.
• Если в файле .h каталога задаётся значение по умолчанию для столбца и то же значение
указано в записи данных, reformat_dat_file.pl уберёт его из файла данных. Таким обра-
зом обеспечивается компактное представление данных.
• Скрипт reformat_dat_file.pl сохраняет пустые строки и строки комментариев в неизмен-
ном виде.
Скрипт reformat_dat_file.pl рекомендуется запускать перед сохранением изменений в дан-
ных каталога. Им удобно пользоваться, просто выполняя make reformat-dat-files в src/
include/catalog/.
• Если вы хотите добавить новый метод уменьшения представления данных, вы должны реали-
зовать его в reformat_dat_file.pl и также научить Catalog::ParseData() разворачивать дан-
ные в полное представление.
2183Объявление и начальное содер-
жимое системных каталогов
69.2.2. Назначение OID
Строке каталога, фигурирующей в исходных данных, можно вручную присвоить OID, добавив поле
метаданных oid => nnnn. Более того, когда строке присваивается OID, для этого OID можно создать
макрос C, добавив поле метаданных oid_symbol => имя.
Предварительно загружаемым строкам каталога должны заранее назначаться OID, если на них
по OID ссылаются другие предварительно загружаемые строки. Назначать OID также требуется,
если на OID нужно будет ссылаться из кода на C. В отсутствие этих условий поле метаданных oid
можно опустить и тогда загрузочный код назначит OID автоматически либо оставит его нулевым,
если OID в данном каталоге не используются. На практике мы обычно явно назначаем OID для
всех строк в определённом каталоге (даже если фактически присутствуют ссылки только на часть
из них) либо не назначаем их вовсе.
Указание фактического числового значения любого OID в коде на C считается крайне нежела-
тельным; вместо этого всегда следует использовать макрос. Прямые обращения к OID в pg_proc
требуются достаточно часто, поэтому был создан специальный механизм, создающий необходи-
мые макросы автоматически; см. src/backend/utils/Gen_fmgrtab.pl. С аналогичной целью преду-
смотрен (но по историческим причинам реализован по-другому) метод создания макросов для OID
в pg_type. Как следствие, записи oid_symbol в этих двух каталогах добавлять не нужно. Подобным
образом в pg_class автоматически включаются макросы для OID системных каталогов и индексов.
Для остальных системных каталогов все нужные вам макросы с oid_symbol вы должны добавлять
вручную.
Чтобы найти свободный OID для новой предварительно загружаемой строки, запустите скрипт
src/include/catalog/unused_oids. Он выводит диапазоны неиспользуемых OID, включающие гра-
ничные значения (например, выведенная строка «45-900» означает, что OID с 45 по 900 включи-
тельно ещё не задействованы). В настоящее время для назначения вручную зарезервированы зна-
чения OID 1-9999; скрипт unused_oids просто просматривает заголовки каталогов и файлы .dat
и проверяет, какие значения в них отсутствуют. Для поиска ошибок вы можете воспользоваться
скриптом duplicate_oids. (Скрипт genbki.pl также выявит дублирующиеся OID во время компи-
ляции.)
Счётчик OID начинается с 10000 при запуске начальной загрузки. Если строка каталога находится
в таблице с OID, но для неё не было явно установлено поле oid, она получит OID, равный 10000
или больше.
69.2.3. Поиск по OID
Перекрёстную ссылку из одной строки исходного каталога на другую можно записать, просто ука-
зав предопределённый OID целевой строки. Однако этот подход провоцирует ошибки и сложен
для понимания, поэтому для часто используемых каталогов в genbki.pl реализованы механизмы
записи символических ссылок. В настоящее время по символическим ссылкам можно обращаться
к методам доступа, функциям, операторам, классам и семействам операторов, а также типам. При
этом действуют следующие правила:
• Для использования символических ссылок в некотором столбце каталога требуется доба-
вить указание BKI_LOOKUP(правило_поиска) в определение этого столбца, где правилом_поис-
ка может быть pg_am, pg_proc, pg_operator, pg_opclass, pg_opfamily или pg_type. Указание
BKI_LOOKUP может быть добавлено к столбцам типа Oid, regproc, oidvector или Oid[]; в по-
следних двух случаях поиск будет выполняться для каждого элемента массива.
• В таком столбце все записи должны иметь символьный формат (исключение составляет 0,
обозначающий InvalidOid). (Если столбец объявлен как regproc, вместо 0 можно написать -.)
Скрипт genbki.pl выдаст предупреждение, встретив нераспознанное имя.
• Методы доступа, как и типы, представляются просто своими именами. Имена типов должны
соответствовать полям typname в соответствующих записях pg_type; псевдонимы типов ис-
пользовать нельзя, например, нельзя написать integer вместо int4.
2184Объявление и начальное содер-
жимое системных каталогов
• Функция может быть представлена своим значением proname, если оно уникально среди запи-
сей pg_proc.dat (это работает как ввод значения типа regproc). В противном случае её нужно
представить как proname(имя_типа_аргумента,имя_типа_аргумента,...), как в regprocedure.
Имена типов аргументов должны записываться в точности так, как они фигурируют в поле
proargtypes в pg_proc.dat. Не добавляйте в эту строку пробелы.
• Операторы представляются в виде oprname(левый_тип,правый_тип), при этом имена ти-
пов записываются в точности так, как они фигурируют в полях oprleft и oprright в
pg_operator.dat. (Вместо опущенного операнда унарного оператора записывается 0.)
• Имена классов операторов и семейств операторов уникальны только в рамках определённого
метода доступа, так что они представляются в виде имя_метода_доступа/имя_объекта.
• Ни в одном из этих случаев не поддерживается указание схемы; все объекты, создаваемые на
стадии начальной загрузки, будут принадлежать схеме pg_catalog.
Скрипт genbki.pl разрешает все символические ссылки при запуске и помещает в формируемый
файл BKI обычные числовые OID. Таким образом, при начальной загрузке отпадает необходимость
в разрешении имён.
69.2.4. Рецепты по редактированию файлов данных
Ниже приведены некоторые предложения по оптимальному решению некоторых распространён-
ных задач при изменении файлов каталогов.
Добавление в каталог нового столбца со значением по умолчанию:  Добавьте столбец в
заголовочный файл с указанием BKI_DEFAULT(значение). Файл данных потребуется редактировать,
только если в каких-либо существующих строках в добавленном поле должно быть не значение
по умолчанию.
Указание значения по умолчанию для существующего столбца, который его не
имел:  Добавьте указание BKI_DEFAULT в заголовочный файл, а затем выполните make reformat-
dat-files для удаления ставших избыточными записей поля.
Удаление столбца, со значением по умолчанию или без:  Удалите столбец из заголовочного
файла, а затем выполните make reformat-dat-files для удаления ставших избыточными записей
поля.
Изменение или удаление существующего значения по умолчанию:  Просто изменить заго-
ловочный файл недостаточно, так как при этом текущие данные будут интерпретироваться некор-
ректно. Сначала выполните make expand-dat-files, чтобы перезаписать в файлах данных все явно
заданные значения по умолчанию, затем удалите или измените указания BKI_DEFAULT, и в завер-
шение выполните make reformat-dat-files для повторного удаления избыточных полей.
Разовая массовая модификация:  Скрипт reformat_dat_file.pl можно скорректировать для
выполнения самых разных массовых модификаций. Просмотрев в нём блочные комментарии, вы
найдёте место, куда можно вставить модифицирующий код. В следующем примере мы произведём
объединение двух логических полей в pg_proc в символьном поле:
1. Добавьте новый столбец со значением по умолчанию в pg_proc.h:
+
+
/* see PROKIND_ categories below */
char
prokind BKI_DEFAULT(f);
2. Создайте на основе reformat_dat_file.pl новый скрипт, который вставит соответствующие зна-
чения «на лету»:
-
-
-
-
+
+
#
#
#
#
#
#
At this point we have the full row in memory as a hash
and can do any operations we want. As written, it only
removes default values, but this script can be adapted to
do one-off bulk-editing.
One-off change to migrate to prokind
Default has already been filled in by now, so change to other
2185Объявление и начальное содер-
жимое системных каталогов
+
+
+
+
+
+
+
+
+
# values as appropriate
if ($values{proisagg} eq 't')
{
$values{prokind} = 'a';
}
elsif ($values{proiswindow} eq 't')
{
$values{prokind} = 'w';
}
3. Запустите новый скрипт:
$ cd src/include/catalog
$ perl rewrite_dat_with_prokind.pl
pg_proc.dat
После этого в файле pg_proc.dat окажутся все три столбца, prokind, proisagg и proiswindow,
хотя они будут фигурировать только в тех строках, где им присваиваются не значения по умол-
чанию, а любые другие значения.
4. Удалите старые столбцы из pg_proc.h:
-
-
-
-
-
/* is it an aggregate? */
bool
proisagg BKI_DEFAULT(f);
/* is it a window function? */
bool
proiswindow BKI_DEFAULT(f);
5. Наконец, выполните make
pg_proc.dat.
reformat-dat-files для удаления ненужных старых записей из
Примеры
кода,
производящего
массовые
модификации,
вы
може-
те
найти
в
скриптах
convert_oid2name.pl
и
remove_pg_type_oid_symbols.pl,
вложенных
в
сообщение:
https://www.postgresql.org/message-id/CAJVSVGVX8gXnPm
+Xa=DxR7kFYprcQ1tNcCT5D0O3ShfnM6jehA@mail.gmail.com
69.3. Формат файла BKI
В этом разделе описывается, как сервер PostgreSQL интерпретирует файлы BKI. Это описание
будет легче понять, если для наглядности вы откроете файл postgres.bki.
Содержимое BKI состоит из последовательности команд. Команды образуются из нескольких ком-
понентов, в зависимости от синтаксиса конкретной команды. Компоненты команд обычно разде-
ляются пробельными символами, но это не обязательно, если не возникает неоднозначности. Спе-
циальный разделитель команд отсутствует; следующий компонент, который не может синтаксиче-
ски относиться к предыдущей команде, начинает следующую. (Обычно новая команда начинается
в отдельной строке, для структурности.) Компонентами команд могут быть определённые ключе-
вые слова, специальные символы (скобки, запятые и т. д.), числа или строки в двойных кавычках.
Все буквы в них воспринимаются с учётом регистра.
Строки, начинающиеся с #, игнорируются.
69.4. Команды BKI
create имя_таблицы oid_таблицы [bootstrap] [shared_relation] [without_oids] [rowtype_oid oid]
(имя1 = тип1 [FORCE NOT NULL | FORCE NULL] [, имя2 = тип2 [FORCE NOT NULL | FORCE NULL], ...])
Создать таблицу имя_таблицы с заданным oid_таблицы и столбцами, указанными в скобках.
Непосредственно bootstrap.c поддерживает следующие типы столбцов: bool, bytea, char
(1 байт), name, int2, int4, regproc, regclass, regtype, text, oid, tid, xid, cid, int2vector,
oidvector, _int4 (массив), _text (массив), _oid (массив), _char (массив), _aclitem (массив). Хо-
тя возможно создать таблицы, содержащие столбцы и других типов, это нельзя сделать, пока не
2186Объявление и начальное содер-
жимое системных каталогов
будет создан и заполнен соответствующими записями каталог pg_type. (По сути это означает,
что только эти типы столбцов могут быть в каталогах начальной загрузки, хотя другие каталоги
могут содержать любые встроенные типы.)
С указанием bootstrap таблица будет создана только на диске; никакие записи о ней не бу-
дут добавлены в pg_class, pg_attribute и т. д. Таким образом, таблица не будет доступна для
обычных операций SQL, пока такие записи не будут добавлены явно (командами insert). Это
указание применяется для создания самой структуры pg_class и подобных ей.
Если добавлено указание shared_relation, таблица создаётся как общая. Она будет содер-
жать столбец OID, если отсутствует указание without_oids. Дополнительным предложением
rowtype_oid может быть задан OID типа строки (OID записи в pg_type); если он не указан, OID
генерируется автоматически. (Предложение rowtype_oid бесполезно, если присутствует ука-
зание bootstrap, но его всё равно можно добавить для документирования.)
open имя_таблицы
Открыть таблицу имя_таблицы для добавления данных. Любая другая таблица, открытая в дан-
ный момент, закрывается.
close имя_таблицы
Закрыть открытую таблицу. Имя таблицы должно задаваться для перепроверки.
insert [OID = значение_oid] ( значение1 значение2 ... )
Вставить новую строку в открытую таблицу, установив значение1, значение2 и т. д. в качестве
значений столбцов и значение_oid в качестве OID. Если значение_oid равно нулю (0) или это
указание опущено, а таблица при этом содержит OID, строке назначается следующий свобод-
ный OID.
Значения NULL могут задаваться специальным ключевым словом _null_. Значения, отличные
от идентификаторов и цифровых строк, должны заключаться в двойные кавычки.
declare [unique] index имя_индекса oid_индекса on имя_таблицы using имя_метода_доступа ( клас-
с_оп1 имя1 [, ...] )
Создать индекс имя_индекса с OID, равным oid_индекса, в таблице имя_таблицы, с методом до-
ступа имя_метода_доступа. Индекс строится по полям имя1, имя2 и т. д., и для них используют-
ся соответственно классы операторов класс_оп1, класс_оп2 и т. д. Эта команда создаёт файл
индекса и добавляет соответствующие записи в каталог, но не инициализирует содержимое
индекса.
declare toast oid_таблицы_toast oid_индекса_toast on имя_таблицы
Создаёт таблицу TOAST для таблицы имя_таблицы. Таблице TOAST назначается OID, равный
oid_таблицы_toast, а её индексу назначается OID, равный oid_индекса_toast. Как и с declare
index, заполнение индекса откладывается.
build indices
Заполнить индексы, объявленные ранее.
69.5. Структура файла BKI
Команда open может применяться, только когда открываемая ей таблица существует и для неё име-
ются записи в каталогах. (Минимальный набор этих каталогов образуют pg_class, pg_attribute,
pg_proc и pg_type.) Чтобы можно было заполнить сами эти таблицы, команда create с указанием
bootstrap неявно открывает создаваемую таблицу для добавления данных.
Кроме того, команды declare index и declare toast нельзя применять, пока не будут созданы и
заполнены системные каталоги.
2187Объявление и начальное содер-
жимое системных каталогов
Таким образом, файл postgres.bki должен иметь следующую структуру:
1. create bootstrap (создание) одной из критичных таблиц
2. insert (добавление) данных, описывающих как минимум критичные таблицы
3. close
4. Повторение для других критичных таблиц.
5. create (создание) (без bootstrap) некритичной таблицы
6. open
7. insert (добавление) требуемых данных
8. close
9. Повторение для других некритичных таблиц.
10.Определение индексов и таблиц TOAST.
11. build indices
Несомненно есть и другие, недокументированные зависимости, диктующие определённый поря-
док.
69.6. Пример BKI
Следующая последовательность команд создаст таблицу test_table с OID 420, имеющую два
столбца cola и colb типа int4 и text, соответственно, и вставит две строки в эту таблицу:
create test_table 420 (cola = int4, colb = text)
open test_table
insert OID=421 ( 1 "value1" )
insert OID=422 ( 2 _null_ )
close test_table
2188