---
layout: page
title: Глава 33. Регрессионные тесты
description: ""
tags: [PostgreSQL]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 33. Регрессионные тесты

Регрессионные тесты представляют собой полный набор проверок реализации SQL в PostgreSQL.
Они тестируют как стандартные SQL-операции, так и расширенные возможности PostgreSQL
33.1. Выполнение тестов
Регрессионное тестирование можно выполнять как на уже установленном и работающем серве-
ре, так и используя временную инсталляцию внутри дерева сборки. Более того, существуют «па-
раллельный» и «последовательный » режимы тестирования. Последовательный метод выполняет
каждый сценарий теста отдельно, тогда как параллельный метод запускает несколько процессов
на сервере с тем, чтобы выполнить определённый набор тестов параллельно. Параллельное тести-
рование позволяет с уверенностью утверждать, что межпроцессное взаимодействие и блокировки
работают корректно.
33.1.1. Запуск тестов на временной инсталляции
Чтобы запустить параллельное регрессионное тестирование после сборки, но до инсталляции, на-
берите:
make check
в каталоге верхнего уровня. (Или, как вариант, вы можете перейти в src/test/regress и выполнить
команду там.) По завершении процесса вы должны увидеть нечто вроде:
=======================
All 115 tests passed.
=======================
или список тестов, не пройденных успешно. Прочитайте Раздел 33.2, прежде чем делать вывод о
серьёзности выявленных «проблем».
Поскольку данный метод тестирования выполняется на временном сервере, он не будет работать,
если вы выполняете сборку под пользователем root, сервер просто не запустится из под root. Ре-
комендуется не делать сборку под пользователем root, если только вы не собираетесь проводить
тестирование после завершения инсталляции.
Если вы сконфигурировали PostgreSQL для инсталляции в месте, где уже установлена предыдущая
версия PostgreSQL, и вы выполняете make check до инсталляции новой версии, вы можете столк-
нуться с тем, что тестирование завершится со сбоем, поскольку новая программа будет пытаться
использовать уже установленные общие библиотеки. (Типичные симптомы - жалобы на неопреде-
лённые символы.) Если вы хотите провести тестирование до перезаписи старой инсталляции, вам
необходимо проводить построение с configure --disable-rpath. Однако этот вариант не рекомен-
дуется для окончательной инсталляции.
Параллельное регрессионное тестирование запускает довольно много процессов под вашим име-
нем пользователя. В настоящее время возможный максимум параллельной обработки составляет
двадцать параллельных тестовых сценариев, а это означает сорок процессов: это и серверный про-
цесс, и psql процесс для каждого тестового сценария. Поэтому если ваша система устанавливает
ограничения на количество процессов для каждого пользователя, имеет смысл уточнить, что ваш
лимит составляет не меньше пятидесяти процессов или около того. В противном случае вы столк-
нетесь с кажущимися случайными сбоями в параллельном тестировании. Если же вы не имеете
права увеличить свой лимит процессов, вы можете снизить степень параллелизма установкой па-
раметра MAX_CONNECTIONS. Например:
make MAX_CONNECTIONS=10 check
выполняет не больше десяти тестов параллельно.
745Регрессионные тесты
33.1.2. Запуск тестов для существующей инсталляции
Чтобы запустить тестирование после инсталляции (см. Главу 16), инициализируйте кластер баз
данных и запустите сервер, как показано в Главе 18, потом введите:
make installcheck
или для параллельного тестирования:
make installcheck-parallel
Тестовые сценарии будут соединяться с сервером на локальном компьютере с номером порта по
умолчанию, если иное не задано переменными среды PGHOST и PGPORT. Тестирование будет прове-
дено в базе данных regression; любая существующая база с таким именем будет удалена.
Также тесты будут создавать случайные объекты общие для кластера, такие как роли и пользо-
ватели. Имена этих объектов будут начинаться с regress_. Опасайтесь использования режима
installcheck там, где пользователи или табличные пространства могут иметь такие имена.
33.1.3. Дополнительные пакеты тестов
Команды make check и make installcheck запускают только «основные» регрессионные тесты,
которые проверяют встроенную функциональность сервера PostgreSQL. Исходный дистрибутив
также содержит дополнительные возможности тестирования, большая часть которых имеет дело с
дополнительной функциональностью, такой, как, например, дополнительные процедурные языки.
Чтобы запустить пакет тестов (включая основные) применительно к модулям, выбранным для по-
строения, наберите одну из этих команд в каталоге верхнего уровня дерева сборки:
make check-world
make installcheck-world
Эти команды запускают тестирование используя временный сервер или уже установленный сер-
вер, в соответствии с данным выше описанием для команд make check и make installcheck.
Остальные детали соответствуют ранее изложенным объяснениям для каждого метода. Необхо-
димо иметь в виду, что команда make check-world выстраивает отдельное дерево временной ин-
сталляции для каждого тестируемого модуля, а это требует гораздо больше времени и дискового
пространства, нежели команда make installcheck-world.
В качестве альтернативного пути можно запустить индивидуальный набор тестов, набрав make
check или make installcheck в подходящем подкаталоге дерева сборки. Имейте в виду, что make
installcheck предполагает, что вы уже установили соответствующие модули, а не только основ-
ной сервер.
Дополнительные тесты, которые можно активизировать таким способом:
• Регрессионные тесты для дополнительных процедурных языков (отличных от PL/pgSQL, кото-
рый тестируется в рамках основного тестирования). Эти тесты расположены в каталоге src/
pl.
• Регрессионные тесты для модулей contrib, расположенные в каталоге contrib. Не для всех
модулей из contrib существуют тесты.
• Регрессионные тесты для библиотеки ECPG, расположенные в src/interfaces/ecpg/test.
• Тесты, для проверки работы одновременного доступа параллельными сессиями, расположен-
ные в src/test/isolation.
• Тесты клиентских программ из src/bin. См. также Раздел 33.4.
При использовании режима installcheck эти тесты удалят все существующие базы дан-
ных с именами pl_regression, contrib_regression, isolation_regression, ecpg1_regression,
ecpg2_regression, а также regression.
746Регрессионные тесты
Тесты TAP выполняются только когда PostgreSQL был сконфигурирован с ключом --enable-tap-
tests. Это рекомендуется для разработки, но если подходящей инсталляции Perl нет, этот ключ
можно опустить.
Некоторые комплекты не запускаются по умолчанию — одни потому, что выполнять их в много-
пользовательской системе небезопасно, а другие потому, что им требуется специальное программ-
ное обеспечение. Эти комплекты тестов можно включить дополнительно, присвоив переменной
PG_TEST_EXTRA (это может быть переменная окружения или скрипта make) строку с их списком
через пробел, например:
make check-world PG_TEST_EXTRA='kerberos ldap ssl'
В настоящее время поддерживаются следующие значения:
kerberos
Запускает комплект тестов в подкаталоге src/test/kerberos. Эти тесты требуют наличия ин-
сталляции MIT Kerberos и открывают сокеты TCP/IP для приёма соединений.
ldap
Запускает комплект тестов в подкаталоге src/test/ldap. Эти тесты требуют наличия инстал-
ляции OpenLDAP и открывают сокеты TCP/IP для приёма соединений.
ssl
Запускает комплект тестов в подкаталоге src/test/ssl. Эти тесты открывают сокеты TCP/IP
для приёма соединений.
Тесты функциональности, которая не поддерживается в текущей конфигурации сборки, не запус-
каются, даже если они указаны в PG_TEST_EXTRA.
33.1.4. Локаль и кодировка
По умолчанию, тесты, работающие на временной инсталляции, используют локаль, определённую
в текущей среде и кодировку базы данных, заданную при выполнении initdb. Для тестирования
различных локалей может оказаться полезным установить подходящие переменные среды, напри-
мер:
make check LANG=C
make check LC_COLLATE=en_US.utf8 LC_CTYPE=ru_RU.utf8
Поддержка переменной LC_ALL в этом случае не реализована; все остальные переменные среды,
относящиеся к локали, работают.
При тестировании на существующей инсталляции, локаль определяется имеющимся кластером
базы данных и не может быть изменена для выполнения теста.
Вы можете задать кодировку базы данных в переменной ENCODING, например:
make check LANG=C ENCODING=EUC_JP
Установка кодировки базы данных таким образом имеет смысл только для локали C; в противном
случае кодировка определяется автоматически из локали, и установка кодировки, не соответству-
ющей локали, приведёт к ошибке.
Кодировка базы данных может быть установлена как для тестирования на временной, так и на
существующей инсталляции, хотя в последнем случае она должна быть совместимой с локалью
этой инсталляции.
33.1.5. Специальные тесты
Пакет основных регрессионных тестов содержит несколько тестовых файлов, которые не запуска-
ются по умолчанию, поскольку они могут зависеть от платформы или выполняться слишком долго.
747Регрессионные тесты
Вы можете запустить те или иные дополнительные тесты, задав переменную EXTRA_TESTS. Напри-
мер, запустить тест numeric_big:
make check EXTRA_TESTS=numeric_big
Запустить тесты сортировки:
make check EXTRA_TESTS='collate.icu.utf8 collate.linux.utf8' LANG=en_US.utf8
Тест collate.linux.utf8 работает только на платформе Linux/glibc. Тест collate.icu.utf8 рабо-
тает только в сборках с поддержкой ICU. Оба теста будут выполнены успешно только в базе данных
с кодировкой UTF-8.
33.1.6. Тестирование сервера горячего резерва
Исходный дистрибутив также содержит регрессионные тесты для статического поведения серве-
ра горячего резерва. Для выполнения тестов требуется работающий ведущий сервер и работаю-
щий резервный, принимающий новые записи WAL от ведущего (с использованием либо трансля-
ции файлов журналов, либо потоковой репликации). Эти серверы не создаются автоматически,
также как и настройка репликации здесь не описана. Пожалуйста, сверьтесь с соответствующими
разделами документации.
Для запуска тестов сервера горячего резерва необходимо создать базу данных regression на ве-
дущем сервере:
psql -h primary -c "CREATE DATABASE regression"
Затем, на ведущем сервере в базе данных regression запустите предварительный скрипт src/test/
regress/sql/hs_primary_setup.sql Например:
psql -h primary -f src/test/regress/sql/hs_primary_setup.sql regression
Убедитесь, что эти изменения распространились на резервный сервер.
Теперь, для выполнения теста, настройте, чтобы подключение по умолчанию выполнялось к ре-
зервному серверу (например, задав переменные среды PGHOST и PGPORT). И, наконец, запустите
make standbycheck в каталоге регрессионных тестов:
cd src/test/regress
make standbycheck
Чтобы протестировать работу резервного сервера в некоторых экстремальных условиях, эти
условия можно получить на главном, воспользовавшись скриптом src/test/regress/sql/
hs_primary_extremes.sql.
33.2. Оценка результатов тестирования
Некоторые правильно установленные и полностью функциональные PostgreSQL инсталляции мо-
гут «давать сбой» при прохождении некоторых регрессионных тестов из-за особенностей, при-
сущих той или иной платформе, таких как различное представление чисел с плавающей запя-
той и формулировкой сообщений. В настоящее время результаты тестов оцениваются простым
diff сравнением с выводом, сделанным в эталонной системе, поэтому результаты чувствительны
к небольшим отличиям между системами. Когда тест завершается со «сбоем», всегда исследуйте
разницу между ожидаемым и полученным результатом; возможно, вы обнаружите, что разница
не столь уж существенна. Тем не менее, мы стремимся поддерживать эталонные файлы на всех
поддерживаемых платформах, чтобы можно было ожидать прохождения всех тестов.
Актуальные итоговые результаты регрессионного тестирования хранятся в каталоге src/test/
regress/results. Тестовый скрипт использует команду diff, чтобы сравнить каждый итоговый
файл с ожидаемыми результатами, которые хранятся в каталоге src/test/regress/expected. Все
различия сохраняются в src/test/regress/regression.diffs для последующей проверки. (Если
проводился тест не из основного пакета, то его результаты появятся в соответствующем подката-
логе, а не в src/test/regress.)
748Регрессионные тесты
Если вам не нравится используемая по умолчанию команда diff, установите переменную среды
PG_REGRESS_DIFF_OPTS, например PG_REGRESS_DIFF_OPTS='-u'. (Или, если хотите, запустите diff
самостоятельно.)
Если по какой-то причине какая-то конкретная платформа генерирует «сбой» для отдельного те-
ста, но изучение его результата убеждает вас, что результат правильный, вы можете добавить но-
вый файл сравнения, чтобы замаскировать отчёт об ошибке для последующего прохождения теста.
За подробностями обратитесь к Разделу 33.3.
33.2.1. Различия в сообщениях об ошибке
Некоторые регрессионные тесты подставляют заведомо неверные входные значения. Сообщения
об ошибке могут выдаваться как PostgreSQL, так и самой операционной системой. В последнем
случае, форма сообщений может отличаться в зависимости от платформы, но отражают они одну
и ту же информацию. Вот эта разница в сообщениях и приводит к «сбоям» регрессионного теста,
которые можно устранить при проверке.
33.2.2. Разница локалей
Если вы проводите тестирование на сервере, который был установлен с локалью, имеющей поря-
док сопоставления, отличный от С, вы можете столкнуться с различиями в порядке сортировки
и, как следствие, с последующими сбоями. Пакет регрессионных тестов решает эту проблему пу-
тём наличия альтернативных файлов результата, которые способны справиться с большим коли-
чеством локалей.
Если вы используете метод тестирования на временной инсталляции, то чтобы запустить тестиро-
вание на другой локали, используйте подходящую переменную среды, относящуюся к локали, в
командной строке make, например:
make check LANG=de_DE.utf8
(Драйвер регрессионного теста обнуляет LC_ALL, поэтому выбор локали посредством данной пере-
менной не работает.) Чтобы не использовать локаль, либо обнулите все переменные среды, отно-
сящиеся к локали, либо установите их в C) или используйте следующую специальную команду:
make check NO_LOCALE=1
Когда тест проходит на существующей инсталляции, установки локали определяются этой инстал-
ляцией. Чтобы это изменить, инициализируйте кластер базы данных с иной локалью, передав со-
ответствующие параметры initdb.
В целом, рекомендуется по возможности проводить регрессионные тесты при таких установках
локали, которые будут использованы в работе, тогда в результате тестирования будут проверены
актуальные участки кода, относящиеся к локали и кодировке. В зависимости от окружения опера-
ционной системы, вы можете столкнуться со сбоями, но вы хотя бы будете знать, какого поведения
локали можно ожидать при работе с реальными приложениями.
33.2.3. Разница в дате и времени
Большая часть результатов проверки даты и времени зависит от часового пояса окружения. Эта-
лонные файлы созданы для пояса PST8PDT (Беркли, Калифорния), поэтому если проводить тесты
не с этим часовым поясом, проявятся мнимые ошибки. Драйвер регрессионного теста задаёт пе-
ременную среды PGTZ как PST8PDT, что позволяет получить корректный результат.
33.2.4. Разница в числах с плавающей запятой
Некоторые тесты применяют 64-битное вычисление чисел с плавающей запятой (double
precision) из столбцов таблицы. Наблюдаются различия в результатах при использовании мате-
матических функций для столбцов double precision. Тесты float8 и geometry особенно чувстви-
тельны к небольшим различиям между платформами и даже режимами оптимизации компиля-
тора. Чтобы понять реальную значимость этих различий, нужно сравнить их глазами, поскольку
обычно они располагаются с десятого разряда справа от десятичной точки.
749Регрессионные тесты
Некоторые системы показывают минус ноль как -0, тогда как другие показывают просто 0.
Некоторые системы сигнализируют об ошибках в pow() и exp() не так, как ожидает текущий код
PostgreSQL.
33.2.5. Разница в сортировке строк
Иногда наблюдаются различия в том, что одни и те же строки выводятся в ином порядке, неже-
ли в контрольном файле. В большинстве случаев это не является, строго говоря, ошибкой. Основ-
ная часть скриптов регрессионных тестов не столь педантична, чтобы использовать ORDER BY для
каждого SELECT, и поэтому в результате порядок строк не гарантирован согласно спецификации
SQL. На практике мы видим, как одинаковые запросы, выполняемые для одних и тех же данных
на одном и том же программном обеспечении, выдают результаты в одинаковом порядке для всех
платформ, в связи с чем отсутствие ORDER BY не является проблемой. Однако некоторые запросы
выявляют межплатформенные различия в сортировке. Когда тестирование идет на уже установ-
ленном сервере, различия в сортировке могут быть следствием того, что локаль установлена в от-
личное от С значение, или некоторые параметры заданы не по умолчанию, такие как work_mem
или стоимостные параметры планировщика.
Поэтому, если вы видите различия в сортировке строк, не стоит волноваться, если только запрос
не использует ORDER BY. Тем не менее, сообщайте нам о таких случаях, чтобы мы могли добавить
ORDER BY в конкретный запрос, чтобы исключить возможность ошибки в будущих релизах.
Вы можете задать вопрос, почему мы явно не добавили ORDER BY во все запросы регрессионных
тестов, чтобы избавиться от таких ошибок раз и навсегда. Причина в том, что это снизит полез-
ность регрессионных тестов, поскольку они будут иметь тенденцию к проверке планов запросов
использующих сортировку, за счёт исключения запросов без сортировки.
33.2.6. Недостаточная глубина стека
Если ошибки
теста приводят к поломке сервера при выполнении команды select
infinite_recurse(), это означает, что предел платформы для размера стека меньше, чем показы-
вает параметр max_stack_depth. Проблема может быть решена запуском сервера с большим разме-
ром стека (рекомендованное значение max_stack_depth по умолчанию - 4 Мб). Если вы не можете
этого сделать, в качестве альтернативы уменьшите значение max_stack_depth.
На платформах, поддерживающих функцию getrlimit(), сервер должен автоматически выбирать
значение переменной max_stack_depth; поэтому, если вы не переписывали это значение вручную,
сбой такого типа — просто дефект, который нужно зарегистрировать.
33.2.7. Тест «случайных значений»
Тестовый скрипт random подразумевает получение случайных значений. В очень редких случаях
это приводит к сбоям в регрессионном тестировании. Выполнение
diff results/random.out expected/random.out
должно выводить одну или несколько строк различий. Нет причин для беспокойства, до тех пор
пока сбои в этом тесте не повторяются постоянно.
33.2.8. Параметры конфигурации
Когда тестирование проходит на существующей инсталляции, некоторые нестандартные значения
параметров могут привести к сбоям в тесте. Например, изменение таких параметров конфигура-
ции, как enable_seqscan или enable_indexscan могут привести к такому изменению системы, ко-
торое сможет воздействовать на результаты тестов, использующих EXPLAIN.
33.3. Вариативные сравнительные файлы
Поскольку некоторые тесты по сути выдают результаты, зависящие от окружения, мы предлагаем
несколько вариантов «ожидаемых» файлов результата. Каждый регрессионный тест может иметь
750Регрессионные тесты
несколько сравнительных файлов, показывающих возможные результаты на разных платформах.
Существует два независимых механизма для определения, какой именно сравнительный файл бу-
дет выбран для каждого теста.
Первый механизм позволяет выбирать сравнительный файл для конкретной платформы. Есть файл
сопоставления src/test/regress/resultmap, в котором определяется, какой сравнительный файл
выбирать для каждой платформы. Чтобы устранить ложные «сбои» тестирования для конкретной
платформы, для начала вы должны выбрать или создать вариант сравнительного файла, а потом
добавить строку в файл resultmap.
Каждая строка в файле сопоставления выглядит как
testname:output:platformpattern=comparisonfilename
Имя теста (testname) здесь - просто название конкретного модуля регрессионного теста. Значе-
ние output показывает, вывод какого файла проверять. Для стандартного регрессионного теста
это всегда out. Значение соответствует расширению выходного файла. platformpattern представ-
ляет собой шаблон в стиле Unix-утилиты expr (т. е. регулярное выражение с неявным ^ якорем
в начале). Этот шаблон сравнивается с именем платформы, которое выводится из config.guess.
comparisonfilename это имя сравнительного файла, который будет использован.
Например: некоторые системы интерпретируют очень маленькие числа с плавающей запятой как
ноль, а не как ошибку потери значимости. Это приводит к разночтениям в регрессионных тестах
для float8. Поэтому мы предлагаем вариант сравнительного файла float8-small-is-zero.out,
который включает в себя результат, ожидаемый для таких систем. Чтобы замаскировать сообще-
ние о ложном «сбое» на платформе OpenBSD, файл resultmap включает в себя:
float8:out:i.86-.*-openbsd=float8-small-is-zero.out
который сработает на любой машине, где выходное значение config.guess соответствует i.86-.*-
openbsd. Другие строки в resultmap выбирают вариант сравнительного файла для других плат-
форм, если это целесообразно.
Второй механизм выбора более автоматический: он просто выбирает «подходящую пару» из
нескольких предлагаемых сравнительных файлов. Драйвер скрипта регрессионного теста рас-
сматривает стандартный сравнительный файл для теста, testname.out, вариативный файл
testname_digit.out (где digit любое одиночное число от 0 до 9). Если какой-нибудь из этих фай-
лов полностью совпадает, тест считается пройденным. В противном случае, для отчёта об ошибке
выбирается файл с наименьшим различием. (Если resultmap включает вводные для конкретного
теста, то в этом случае testname подменное имя, взятое из файла resultmap.)
Например, для теста char сравнительный файл char.out содержит результаты, ожидаемые для
локалей C и POSIX, тогда как файл char_1.out содержит результаты, характерные для многих дру-
гих локалей.
Механизм "лучшей пары" был разработан, чтобы справляться с результатами, зависящими от ло-
кали, но он может применяться в любой ситуации, когда сложно предсказать результаты, исходя
только из названия платформы. Ограниченность этого метода проявляется лишь в том, что драй-
вер теста не может сказать, какой вариант правилен для данного окружения; просто выбирает-
ся вариант, который кажется наиболее подходящим. Поэтому безопаснее всего использовать этот
метод только для вариативных результатов, которые вы хотели бы видеть одинаково надёжными
для любого контекста.
33.4. Тесты TAP
Различные тесты, особенно тесты клиентских программ в src/bin, используют инструменты Perl
TAP и запускаются программой тестирования Perl prove. Вы можете передать аргументы команд-
ной строки команде prove, установив для make переменную PROVE_FLAGS, например:
make -C src/bin check PROVE_FLAGS='--timer'
За дополнительными сведениями обратитесь к странице руководства по prove.
751Регрессионные тесты
В переменной PROVE_TESTS, которую воспринимает make, может быть указан список разделённых
пробелами путей, заданных относительно расположения Makefile, вызывающего prove. Этот спи-
сок определяет подмножество тестов для выполнения, вместо всех по умолчанию (t/*.pl). Напри-
мер:
make check PROVE_TESTS='t/001_test1.pl t/003_test3.pl'
Тесты на базе TAP требуют модуля Perl IPC::Run. Этот модуль доступен из CPAN или операционной
системы.
33.5. Проверка покрытия теста
Исходный код PostgreSQL может быть скомпилирован с инструментарием для теста покрытия, так
что можно проверить, какие части кода покрывает регрессионное тестирование или любое другое
тестирование, запускаемое относительно кода. В настоящее время эта возможность поддержива-
ется в сочетании с компиляцией с GCC и требует наличия gcov и lcov программ.
Типичный рабочий процесс выглядит так:
./configure --enable-coverage ... OTHER OPTIONS ...
make
make check # или другой комплект тестов
make coverage-html
Затем откройте в своём HTML-браузере страницу coverage/index.html. Команды make работают
и в подкаталогах.
Если у вас нет программы lcov или вы предпочитаете HTML-отчёту текстовый формат, вы можете
также выполнить
make coverage
вместо make coverage-html и получить выходные файлы .gcov для каждого исходного файла, от-
носящегося к тесту. (Команды make coverage и make coverage-html перезаписывают файлы друг
друга, поэтому при одновременном их использовании может возникнуть путаница.)
Чтобы обнулить подсчёт выполнений между тестами, запустите:
make coverage-clean