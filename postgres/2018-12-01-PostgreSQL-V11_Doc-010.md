---
layout: page
title: Глава 10. Преобразование типов
description: ""
tags: [PostgreSQL]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 10. Преобразование типов

SQL-операторы, намеренно или нет, требуют совмещать данные разных типов в одном выражении.
Для вычисления подобных выражений со смешанными типами PostgreSQL предоставляет широкий
набор возможностей.
Очень часто пользователю не нужно понимать все тонкости механизма преобразования. Однако
следует учитывать, что неявные преобразования, производимые PostgreSQL, могут влиять на ре-
зультат запроса. Поэтому при необходимости нужные результаты можно получить, применив яв-
ное преобразование типов.
В этой главе описываются общие механизмы преобразования типов и соглашения, принятые в
PostgreSQL. За дополнительной информацией о конкретных типах данных и разрешённых для них
функциях и операторах обратитесь к соответствующим разделам в Главе 8 и Главе 9.
10.1. Обзор
SQL — язык со строгой типизацией. То есть каждый элемент данных в нём имеет некоторый тип,
определяющий его поведение и допустимое использование. PostgreSQL наделён расширяемой си-
стемой типов, более универсальной и гибкой по сравнению с другими реализациями SQL. При этом
преобразования типов в PostgreSQL в основном подчиняются определённым общим правилам, для
их понимания не нужен эвристический анализ. Благодаря этому в выражениях со смешанными
типами можно использовать даже типы, определённые пользователями.
Анализатор выражений PostgreSQL разделяет их лексические элементы на пять основных кате-
горий: целые числа, другие числовые значения, текстовые строки, идентификаторы и ключевые
слова. Константы большинства не числовых типов сначала классифицируются как строки. В опре-
делении языка SQL допускается указывать имена типов в строках и это можно использовать в
PostgreSQL, чтобы направить анализатор по верному пути. Например, запрос:
SELECT text 'Origin' AS "label", point '(0,0)' AS "value";
label | value
--------+-------
Origin | (0,0)
(1 row)
содержит две строковых константы, типа text и типа point. Если для такой константы не указан
тип, для неё первоначально предполагается тип unknown, который затем может быть уточнён, как
описано ниже.
В SQL есть четыре фундаментальных фактора, определяющих правила преобразования типов для
анализатора выражений PostgreSQL:
Вызовы функций
Система типов PostgreSQL во многом построена как дополнение к богатым возможностям функ-
ций. Функции могут иметь один или несколько аргументов, и при этом PostgreSQL разрешает
перегружать имена функций, так что имя функции само по себе не идентифицирует вызывае-
мую функцию; анализатор выбирает правильную функцию в зависимости от типов переданных
аргументов.
Операторы
PostgreSQL позволяет использовать в выражениях префиксные и постфиксные операторы с од-
ним аргументом, а также операторы с двумя аргументами. Как и функции, операторы можно
перегружать, так что и с ними существует проблема выбора правильного оператора.
Сохранение значений
SQL-операторы INSERT и UPDATE помещают результаты выражений в таблицы. При этом полу-
чаемые значения должны соответствовать типам целевых столбцов или, возможно, приводить-
ся к ним.
344Преобразование типов
UNION, CASE и связанные конструкции
Так как все результаты запроса объединяющего оператора SELECT должны оказаться в одном
наборе столбцов, результаты каждого подзапроса SELECT должны приводиться к одному набору
типов. Подобным образом, результирующие выражения конструкции CASE должны приводиться
к общему типу, так как выражение CASE в целом должно иметь определённый выходной тип. То
же справедливо в отношении конструкций ARRAY и функций GREATEST и LEAST.
Информация о существующих преобразованиях или приведениях типов, для каких типов они опре-
делены и как их выполнять, хранится в системных каталогах. Пользователь также может добавить
дополнительные преобразования с помощью команды CREATE CAST. (Обычно это делается, когда
определяются новые типы данных. Набор приведений для встроенных типов достаточно хорошо
проработан, так что его лучше не менять.)
Дополнительная логика анализа помогает выбрать оптимальное приведение в группах типов,
допускающих неявные преобразования. Для этого типы данных разделяются на несколько ба-
зовых категорий, которые включают: boolean, numeric, string, bitstring, datetime, timespan,
geometric, network и пользовательские типы. (Полный список категорий приведён в Таблице 52.63;
хотя его тоже можно расширить, определив свои категории.) В каждой категории могут быть вы-
браны один или несколько предпочитаемых типов, которые будут считаться наиболее подходя-
щими при рассмотрении нескольких вариантов. Аккуратно выбирая предпочитаемые типы и до-
пустимые неявные преобразования, можно добиться того, что выражения с неоднозначностями
(в которых возможны разные решения задачи преобразования) будут разрешаться наилучшим об-
разом.
Все правила преобразования типов разработаны с учётом следующих принципов:
• Результат неявных преобразованиях всегда должен быть предсказуемым и понятным.
• Если в неявном преобразовании нет нужды, анализатор и исполнитель запроса не должны
тратить лишнее время на это. То есть, если запрос хорошо сформулирован и типы значений
совпадают, он должен выполняться без дополнительной обработки в анализаторе и без лиш-
них вызовов неявных преобразований.
• Кроме того, если запрос изначально требовал неявного преобразования для функции, а поль-
зователь определил новую функцию с точно совпадающими типами аргументов, анализатор
должен переключиться на новую функцию и больше не выполнять преобразование для вызова
старой.
10.2. Операторы
При выборе конкретного оператора, задействованного в выражении, PostgreSQL следует описан-
ному ниже алгоритму. Заметьте, что на этот выбор могут неявно влиять приоритеты остальных
операторов в данном выражении, так как они определяют, какие подвыражения будут аргумента-
ми операторов. Подробнее об этом рассказывается в Подразделе 4.1.6.
Выбор оператора по типу
1.
Выбрать операторы для рассмотрения из системного каталога pg_operator. Если имя операто-
ра не дополнено именем схемы (обычно это так), будут рассматриваться все операторы с под-
ходящим именем и числом аргументов, видимые в текущем пути поиска (см. Подраздел 5.8.3).
Если имя оператора определено полностью, в рассмотрение принимаются только операторы
из указанной схемы.
•
2.
(Optional) Если в пути поиска оказывается несколько операторов с одинаковыми типами
аргументов, учитываются только те из них, которые находятся в пути раньше. Операторы
с разными типами аргументов рассматриваются на равных правах вне зависимости от их
положения в пути поиска.
Проверить, нет ли среди них оператора с точно совпадающими типами аргументов. Если такой
оператор есть (он может быть только одним в отобранном ранее наборе), использовать его.
345Преобразование типов
1
Отсутствие точного совпадения создаёт угрозу вызова с указанием полного имени (нетипич-
ным) любого оператора, который может оказаться в схеме, где могут создавать объекты недо-
веренные пользователи. В таких ситуациях приведите типы аргументов для получения точного
совпадения.
3.
a. (Optional) Если один аргумент при вызове бинарного оператора имеет тип unknown, для
данной проверки предполагается, что он имеет тот же тип, что и второй его аргумент. При
вызове бинарного оператора с двумя аргументами unknown или унарного с одним unknown,
оператор не будет выбран на этом шаге.
b. (Optional) Если один аргумент при вызове бинарного оператора имеет тип unknown, а другой
— домен, проверить, есть ли оператор, принимающий базовый тип домена с обеих сторон;
если таковой находится, использовать его.
Найти самый подходящий.
a. Отбросить кандидаты, для которых входные типы не совпадают и не могут быть преобразо-
ваны (неявным образом) так, чтобы они совпали. В данном случае считается, что констан-
ты типа unknown можно преобразовать во что угодно. Если остаётся только один кандидат,
использовать его, в противном случае перейти к следующему шагу.
b. Если один из аргументов имеет тип домен, далее считать его типом базовый тип домена.
Благодаря этому при поиске неоднозначно заданного оператора домены будут подобны
свои базовым типам.
c. Просмотреть все кандидаты и оставить только те, для которых точно совпадают как мож-
но больше типов аргументов. Оставить все кандидаты, если точных совпадений нет. Если
остаётся только один кандидат, использовать его, в противном случае перейти к следую-
щему шагу.
d. Просмотреть все кандидаты и оставить только те, которые принимают предпочитаемые
типы (из категории типов входных значений) в наибольшем числе позиций, где требуется
преобразование типов. Оставить все кандидаты, если ни один не принимает предпочитае-
мые типы. Если остаётся только один кандидат, использовать его, в противном случае пе-
рейти к следующему шагу.
e. Если какие-либо значения имеют тип unknown, проверить категории типов, принимаемых
в данных позициях аргументов оставшимися кандидатами. Для каждой позиции выбрать
категорию string, если какой-либо кандидат принимает эту категорию. (Эта склонность
к строкам объясняется тем, что константа типа unknown выглядит как строка.) Если эта
категория не подходит, но все оставшиеся кандидаты принимают одну категорию, выбрать
её; в противном случае констатировать неудачу — сделать правильный выбор без дополни-
тельных подсказок нельзя. Затем отбросить кандидаты, которые не принимают типы вы-
бранной категории. Далее, если какой-либо кандидат принимает предпочитаемый тип из
этой категории, отбросить кандидаты, принимающие другие, не предпочитаемые типы для
данного аргумента. Оставить все кандидаты, если эти проверки не прошёл ни один. Если
остаётся только один кандидат, использовать его, в противном случае перейти к следую-
щему шагу.
f. Если в списке аргументов есть аргументы и типа unknown, и известного типа, и этот из-
вестный тип один для всех аргументов, предположить, что аргументы типа unknown также
имеют этот тип, и проверить, какие кандидаты могут принимать этот тип в позиции аргу-
мента unknown. Если остаётся только один кандидат, использовать его, в противном случае
констатировать неудачу.
Ниже это проиллюстрировано на примерах.
1
Эта угроза неактуальна для имён без схемы, так как путь поиска, содержащий схемы, в которых недоверенные пользователи могут создавать объекты, не
соответствует шаблону безопасного использования схем.
346Преобразование типов
Пример 10.1. Разрешение оператора факториала
В стандартном каталоге определён только один оператор факториала (постфиксный !) и он при-
нимает аргумент типа bigint. При просмотре следующего выражения его аргументу изначально
назначается тип integer:
SELECT 40 ! AS "40 factorial";
40 factorial
--------------------------------------------------
815915283247897734345611269596115894272000000000
(1 row)
Анализатор выполняет преобразование типа для этого операнда и запрос становится равносиль-
ным:
SELECT CAST(40 AS bigint) ! AS "40 factorial";
Пример 10.2. Разрешение оператора конкатенации строк
Синтаксис текстовых строк используется как для записи строковых типов, так и для сложных ти-
пов расширений. Если тип не указан явно, такие строки сопоставляются по тому же алгоритму с
наиболее подходящими операторами.
Пример с одним неопределённым аргументом:
SELECT text 'abc' || 'def' AS "text and unknown";
text and unknown
------------------
abcdef
(1 row)
В этом случае анализатор смотрит, есть ли оператор, у которого оба аргумента имеют тип text.
Такой оператор находится, поэтому предполагается, что второй аргумент следует воспринимать
как аргумент типа text.
Конкатенация двух значений неопределённых типов:
SELECT 'abc' || 'def' AS "unspecified";
unspecified
-------------
abcdef
(1 row)
В данном случае нет подсказки для выбора типа, так как в данном запросе никакие типы не ука-
заны. Поэтому анализатор просматривает все возможные операторы и находит в них кандидаты,
принимающие аргументы категорий string и bit-string. Так как категория string является предпо-
чтительной, выбирается она, а затем для разрешения типа не типизированной константы выбира-
ется предпочтительный тип этой категории, text.
Пример 10.3. Разрешение оператора абсолютного значения и отрицания
В каталоге операторов PostgreSQL для префиксного оператора @ есть несколько записей, описы-
вающих операции получения абсолютного значения для различных числовых типов данных. Одна
из записей соответствует типу float8, предпочтительного в категории числовых типов. Таким об-
разом, столкнувшись со значением типа unknown, PostgreSQL выберет эту запись:
SELECT @ '-4.5' AS "abs";
abs
-----
347Преобразование типов
4.5
(1 row)
Здесь система неявно привела константу неизвестного типа к типу float8, прежде чем применять
выбранный оператор. Можно убедиться в том, что выбран именно тип float8, а не какой-то другой:
SELECT @ '-4.5e500' AS "abs";
ОШИБКА:
"-4.5e500" вне диапазона для типа double precision
С другой стороны, префиксный оператор ~ (побитовое отрицание) определён только для целочис-
ленных типов данных, но не для float8. Поэтому, если попытаться выполнить похожий запрос с
~, мы получаем:
SELECT ~ '20' AS "negation";
ОШИБКА: оператор не уникален: ~ "unknown"
ПОДСКАЗКА: Не удалось выбрать лучшую кандидатуру оператора. Возможно, вам следует
добавить явные преобразования типов.
Это происходит оттого, что система не может решить, какой оператор предпочесть из нескольких
возможных вариантов ~. Мы можем облегчить её задачу, добавив явное преобразование:
SELECT ~ CAST('20' AS int8) AS "negation";
negation
----------
-21
(1 row)
Пример 10.4. Разрешение оператора включения в массив
Ещё один пример разрешения оператора с одним аргументом известного типа и другим неизвест-
ного:
SELECT array[1,2] <@ '{1,2,3}' as "is subset";
is subset
-----------
t
(1 row)
В каталоге операторов PostgreSQL есть несколько записей для инфиксного оператора <@, но
только два из них могут принять целочисленный массива слева: оператор включения массива
(anyarray<@anyarray) и оператор включения диапазона (anyelement<@anyrange). Так как ни один
из этих полиморфных псевдотипов (см. Раздел 8.21) не считается предпочтительным, анализатор
не может избавиться от неоднозначности на данном этапе. Однако, в Шаг 3.f говорится, что кон-
станта неизвестного типа должна рассматриваться как значение типа другого аргумента, в дан-
ном случае это целочисленный массив. После этого подходящим считается только один из двух
операторов, так что выбирается оператор с целочисленными массивами. (Если бы был выбран опе-
ратор включения диапазона, мы получили бы ошибку, так как значение в строке не соответствует
формату значений диапазона.)
Пример 10.5. Нестандартный оператор с доменом
Иногда пользователи пытаются ввести операторы, применимые только к определённому домену.
Это возможно, но вовсе не так полезно, как может показаться, ведь правила разрешения операто-
ров применяются к базовому типу домена. Взгляните на этот пример:
CREATE DOMAIN mytext AS text CHECK(...);
CREATE FUNCTION mytext_eq_text (mytext, text) RETURNS boolean AS ...;
CREATE OPERATOR = (procedure=mytext_eq_text, leftarg=mytext, rightarg=text);
348Преобразование типов
CREATE TABLE mytable (val mytext);
SELECT * FROM mytable WHERE val = 'foo';
В этом запросе не будет использоваться нововведённый оператор. При разборе запроса сначала
будет проверено, есть ли оператор mytext = mytext (см. Шаг 2.a), но это не так; затем будет рас-
смотрен базовый тип домена (text) и проверено наличие оператора text = text (см. Шаг 2.b), и
таковой действительно есть; в итоге строковое значение типа unknown будет воспринято как text
и будет применён оператор text = text. Единственный вариант задействовать нововведённый опе-
ратор — добавить явное приведение:
SELECT * FROM mytable WHERE val = text 'foo';
так, чтобы оператор mytext = text был найден сразу, согласно правилу точного совпадения. Ес-
ли дело доходит до правил наибольшего соответствия, они активно дискредитируют операторы
доменных типов. Если бы они этого не делали, с таким оператором возникало бы слишком много
ошибок разрешения операторов, потому что правила приведения всегда считают домен приводи-
мым к базовому типу и наоборот, так что доменный оператор применялся бы во всех случаях, где
применяется одноимённый оператор с базовым типом.
10.3. Функции
При выборе конкретной функции, задействованной в выражении, PostgreSQL следует описанному
ниже алгоритму.
Разрешение функции по типу
1.
Выбрать функции для рассмотрения из системного каталога pg_proc. Если имя функции не
дополнено именем схемы, будут рассматриваться все функции с подходящим именем и числом
аргументов, видимые в текущем пути поиска (см. Подраздел 5.8.3). Если имя функции опреде-
лено полностью, в рассмотрение принимаются только функции из указанной схемы.
a. (Optional) Если в пути поиска оказывается несколько функций с одинаковыми типами ар-
гументов, учитываются только те из них, которые находятся в пути раньше. Функции с раз-
ными типами аргументов рассматриваются на равных правах вне зависимости от их поло-
жения в пути поиска.
b. (Optional) Если в числе параметров функции есть массив VARIADIC и при вызове не указы-
вается ключевое слово VARIADIC, функция обрабатывается, как если бы этот параметр был
заменён одним или несколькими параметрами типа элементов массива, по числу аргумен-
тов при вызове. После такого расширения по фактическим типам аргументов она может
совпасть с некоторой функцией с постоянным числом аргументов. В этом случае исполь-
зуется функция, которая находится в пути раньше, а если они оказываются в одной схеме,
предпочитается вариант с постоянными аргументами.
2
Это создаёт угрозу безопасности при вызове с полным именем функции с переменным
числом аргументов, которая может оказаться в схеме, где могут создавать объекты недо-
веренные пользователи. Злонамеренный пользователь может перехватывать управление и
выполнять произвольные SQL-функции, как будто их выполняете вы. Запись вызова с клю-
чевым словом VARIADIC устраняет эту угрозу. Однако для вызовов с передачей параметров
VARIADIC "any" часто не существует необходимой формулировки с ключом VARIADIC. Чтобы
такие вызовы были безопасными, создание объектов в схеме функции должно разрешаться
только доверенным пользователям.
c.
(Optional) Функции, для которых определены значения параметров по умолчанию, счита-
ются совпадающими с вызовом, в котором опущено ноль или более параметров в соответ-
ствующих позициях. Если для вызова подходят несколько функций, используется та, что
обнаруживается в пути поиска раньше. Если в одной схеме оказываются несколько функ-
ций с одинаковыми типами в позициях обязательных параметров (что возможно, если в них
2
Эта угроза неактуальна для имён без схемы, так как путь поиска, содержащий схемы, в которых недоверенные пользователи могут создавать объекты, не
соответствует шаблону безопасного использования схем.
349Преобразование типов
определены разные наборы пропускаемых параметров), система не сможет выбрать опти-
мальную, и выдаст ошибку «неоднозначный вызов функции», если лучшее соответствие
для вызова не будет найдено.
2
Это создаёт угрозу при вызове с полным именем любой функции, которая может оказать-
ся в схеме, где могут создавать объекты недоверенные пользователи. Злонамеренный поль-
зователь может создать функцию с именем уже существующей, продублировав параметры
исходной и добавив дополнительные со значениями по умолчанию. В результате при по-
следующих вызовах будет выполняться не исходная функция. Для ликвидации этой угрозы
помещайте функции в схемы, в которых создавать объекты могут только доверенные объ-
екты.
2. Проверить, нет ли функции, принимающей в точности типы входных аргументов. Если такая
функция есть (она может быть только одной в отобранном ранее наборе), использовать её. От-
2
сутствие точного совпадения создаёт угрозу вызова с полным именем функции в схеме, где
могут создавать объекты недоверенные пользователи. В таких ситуациях приведите типы ар-
гументов для получения точного соответствия. (В случаях с unknown совпадения на этом этапе
не будет никогда.)
3. Если точное совпадение не найдено, проверить, не похож ли вызов функции на особую форму
преобразования типов. Это имеет место, когда при вызове функции передаётся всего один ар-
гумент и имя функции совпадает с именем (внутренним) некоторого типа данных. Более того,
аргументом функции должна быть либо строка неопределённого типа, либо значение типа,
двоично-совместимого с указанным или приводимого к нему с помощью функций ввода/выво-
да типа (то есть, преобразований в стандартный строковый тип и обратно). Если эти условия
3
выполняются, вызов функции воспринимается как особая форма конструкции CAST.
4. Найти самый подходящий.
a. Отбросить кандидаты, для которых входные типы не совпадают и не могут быть преобразо-
ваны (неявным образом) так, чтобы они совпали. В данном случае считается, что констан-
ты типа unknown можно преобразовать во что угодно. Если остаётся только один кандидат,
использовать его, в противном случае перейти к следующему шагу.
b. Если один из аргументов имеет тип домен, далее считать его типом базовый тип домена.
Благодаря этому при поиске неоднозначно заданной функции домены будут подобны свои
базовым типам.
c. Просмотреть все кандидаты и оставить только те, для которых точно совпадают как мож-
но больше типов аргументов. Оставить все кандидаты, если точных совпадений нет. Если
остаётся только один кандидат, использовать его, в противном случае перейти к следую-
щему шагу.
d. Просмотреть все кандидаты и оставить только те, которые принимают предпочитаемые
типы (из категории типов входных значений) в наибольшем числе позиций, где требуется
преобразование типов. Оставить все кандидаты, если ни один не принимает предпочитае-
мые типы. Если остаётся только один кандидат, использовать его, в противном случае пе-
рейти к следующему шагу.
e. Если какие-либо значения имеют тип unknown, проверить категории типов, принимаемых
в данных позициях аргументов оставшимися кандидатами. Для каждой позиции выбрать
категорию string, если какой-либо кандидат принимает эту категорию. (Эта склонность
к строкам объясняется тем, что константа типа unknown выглядит как строка.) Если эта
категория не подходит, но все оставшиеся кандидаты принимают одну категорию, выбрать
её; в противном случае констатировать неудачу — сделать правильный выбор без дополни-
тельных подсказок нельзя. Затем отбросить кандидаты, которые не принимают типы вы-
бранной категории. Далее, если какой-либо кандидат принимает предпочитаемый тип из
этой категории, отбросить кандидаты, принимающие другие, не предпочитаемые типы для
данного аргумента. Оставить все кандидаты, если эти проверки не прошёл ни один. Если
3
Этот шаг нужен для поддержки приведений типов в стиле вызова функции, когда на самом деле соответствующей функции приведения нет. Если такая
функция приведения есть, она обычно называется именем выходного типа и необходимости в особом подходе нет. За дополнительными комментариями
обратитесь к CREATE CAST.
350Преобразование типов
остаётся только один кандидат, использовать его, в противном случае перейти к следую-
щему шагу.
f.
Если в списке аргументов есть аргументы и типа unknown, и известного типа, и этот из-
вестный тип один для всех аргументов, предположить, что аргументы типа unknown также
имеют этот тип, и проверить, какие кандидаты могут принимать этот тип в позиции аргу-
мента unknown. Если остаётся только один кандидат, использовать его, в противном случае
констатировать неудачу.
Заметьте, что для функций действуют те же правила «оптимального соответствия», что и для опе-
раторов. Они проиллюстрированы следующими примерами.
Пример 10.6. Разрешение функции округления по типам аргументов
В PostgreSQL есть только одна функция round, принимающая два аргумента: первый типа numeric,
а второй — integer. Поэтому в следующем запросе первый аргумент integer автоматически при-
водится к типу numeric:
SELECT round(4, 4);
round
--------
4.0000
(1 row)
Таким образом, анализатор преобразует этот запрос в:
SELECT round(CAST (4 AS numeric), 4);
Так как числовые константы с десятичными точками изначально относятся к типу numeric, для
следующего запроса преобразование типов не потребуется, так что он немного эффективнее:
SELECT round(4.0, 4);
Пример 10.7. Разрешение функций с переменными параметрами
CREATE FUNCTION public.variadic_example(VARIADIC numeric[]) RETURNS int
LANGUAGE sql AS 'SELECT 1';
CREATE FUNCTION
Эта функция принимает в аргументах ключевое слово VARIADIC, но может вызываться и без него.
Ей можно передавать и целочисленные, и любые числовые аргументы:
SELECT public.variadic_example(0),
public.variadic_example(0.0),
public.variadic_example(VARIADIC array[0.0]);
variadic_example | variadic_example | variadic_example
------------------+------------------+------------------
1 |
1 |
1
(1 row)
Однако для первого и второго вызова предпочтительнее окажутся специализированные функции,
если таковые есть:
CREATE FUNCTION public.variadic_example(numeric) RETURNS int
LANGUAGE sql AS 'SELECT 2';
CREATE FUNCTION
CREATE FUNCTION public.variadic_example(int) RETURNS int
LANGUAGE sql AS 'SELECT 3';
CREATE FUNCTION
SELECT public.variadic_example(0),
public.variadic_example(0.0),
351Преобразование типов
public.variadic_example(VARIADIC array[0.0]);
variadic_example | variadic_example | variadic_example
------------------+------------------+------------------
3 |
2 |
1
(1 row)
Если используется конфигурация по умолчанию и существует только первая функция, первый и
второй вызовы будут небезопасными. Любой пользователь может перехватить их, создав вторую
или третью функцию. Безопасным будет третий вызов, в котором тип аргумента соответствует в
точности и используется ключевое слово VARIADIC.
Пример 10.8. Разрешение функции извлечения подстроки
В PostgreSQL есть несколько вариантов функции substr, и один из них принимает аргументы типов
text и integer. Если эта функция вызывается со строковой константой неопределённого типа, си-
стема выбирает функцию, принимающую аргумент предпочитаемой категории string (а конкрет-
нее, типа text).
SELECT substr('1234', 3);
substr
--------
34
(1 row)
Если текстовая строка имеет тип varchar, например когда данные поступают из таблицы, анали-
затор попытается привести её к типу text:
SELECT substr (varchar '1234', 3);
substr
--------
34
(1 row)
Этот запрос анализатор фактически преобразует в:
SELECT substr(CAST (varchar '1234' AS text), 3);
Примечание
Анализатор узнаёт из каталога pg_cast, что типы text и varchar двоично-совместимы,
что означает, что один тип можно передать функции, принимающей другой, не выпол-
няя физического преобразования. Таким образом, в данном случае операция преобра-
зования на самом не добавляется.
И если функция вызывается с аргументом типа integer, анализатор попытается преобразовать
его в тип text:
SELECT substr(1234, 3);
ОШИБКА: функция substr(integer, integer) не существует
ПОДСКАЗКА: Функция с данными именем и типами аргументов не найдена. Возможно, вам
следует добавить явные преобразования типов.
Этот вариант не работает, так как integer нельзя неявно преобразовать в text. Однако с явным
преобразованием запрос выполняется:
SELECT substr(CAST (1234 AS text), 3);
substr
--------
352Преобразование типов
34
(1 row)
10.4. Хранимое значение
Значения, вставляемые в таблицу, преобразуется в тип данных целевого столбца по следующему
алгоритму.
Преобразование по типу хранения
1. Проверить точное совпадение с целевым типом.
2. Если типы не совпадают, попытаться привести тип к целевому. Это возможно, если в каталоге
pg_cast (см. CREATE CAST) зарегистрировано приведение присваивания между двумя типами.
Если же результат выражения — строка неизвестного типа, содержимое этой строки будет
подано на вход процедуре ввода целевого типа.
3. Проверить, не требуется ли приведение размера для целевого типа. Приведение размера — это
преобразование типа к такому же. Если это приведение описано в каталоге pg_cast, приме-
нить к его к результату выражения, прежде чем сохранить в целевом столбце. Функция, реали-
зующая такое приведение, всегда принимает дополнительный параметр типа integer, в кото-
ром передаётся значение atttypmod для целевого столбца (обычно это её объявленный размер,
хотя интерпретироваться значение atttypmod для разных типов данных может по-разному), и
третий параметр типа boolean, передающий признак явное/неявное преобразование. Функция
приведения отвечает за все операции с длиной, включая её проверку и усечение данных.
Пример 10.9. Преобразование для типа хранения character
Следующие запросы показывают, что сохраняемое значение подгоняется под размер целевого
столбца, объявленного как character(20):
CREATE TABLE vv (v character(20));
INSERT INTO vv SELECT 'abc' || 'def';
SELECT v, octet_length(v) FROM vv;
v
| octet_length
----------------------+--------------
abcdef
|
20
(1 row)
Суть происходящего здесь в том, что две константы неизвестного типа по умолчанию воспринима-
ются как значения text, что позволяет применить к ним оператор || как оператор конкатенации
значений text. Затем результат оператора, имеющий тип text, приводится к типу bpchar («blank-
padded char» (символы, дополненные пробелами), внутреннее имя типа character) в соответствии
с типом целевого столбца. (Так как типы text и bpchar двоично-совместимы, при этом преобразо-
вании реальный вызов функции не добавляется.) Наконец, в системном каталоге находится функ-
ция изменения размера bpchar(bpchar, integer, boolean) и применяется для результата опера-
тора и длины столбца. Эта связанная с типом функция проверяет длину данных и добавляет недо-
стающие пробелы.
10.5. UNION, CASE и связанные конструкции
SQL-конструкция UNION взаимодействует с системой типов, так как ей приходится объединять зна-
чения возможно различных типов в единый результирующий набор. Алгоритм разрешения типов
при этом применяется независимо к каждому отдельному столбцу запроса. Подобным образом
различные типы сопоставляются при выполнении INTERSECT и EXCEPT сопоставляют различные
типы подобно UNION. По такому же алгоритму сопоставляют типы выражений и определяют тип
своего результата конструкции CASE, ARRAY, VALUES, GREATEST и LEAST.
Разрешение типов для UNION, CASE и связанных конструкций
1.
Если все данные одного типа и это не тип unknown, выбрать его.
353Преобразование типов
4
2. Если тип данных — домен, далее считать их типом базовый тип домена.
3. Если все данные типа unknown, выбрать для результата тип text (предпочитаемый для катего-
рии string). В противном случае значения unknown для остальных правил игнорируются.
4. Если известные типы входных данных оказываются не из одной категории, констатировать
неудачу.
5. Выбрать первый известный предпочитаемый тип из этой категории, если такой есть.
6. В противном случае выбрать последний известный тип, в который можно неявно преобразовать
все данные предшествующих известных типов. (Такой тип есть всегда, в крайнем случае этому
условию удовлетворяет первый тип.)
7. Привести все данные к выбранном типу. Констатировать неудачу, если для каких-либо данных
преобразование в этот тип невозможно.
Ниже это проиллюстрировано на примерах.
Пример 10.10. Разрешение типов с частичным определением в Union
SELECT text 'a' AS "text" UNION SELECT 'b';
text
------
a
b
(2 rows)
В данном случае константа 'b' неизвестного типа будет преобразована в тип text.
Пример 10.11. Разрешение типов в простом объединении
SELECT 1.2 AS "numeric" UNION SELECT 1;
numeric
---------
1
1.2
(2 rows)
Константа 1.2 имеет тип numeric и целочисленное значение 1 может быть неявно приведено к
типу numeric, так что используется этот тип.
Пример 10.12. Разрешение типов в противоположном объединении
SELECT 1 AS "real" UNION SELECT CAST('2.2' AS REAL);
real
------
1
2.2
(2 rows)
Здесь значение типа real нельзя неявно привести к integer, но integer можно неявно привести
к real, поэтому типом результата объединения будет real.
Пример 10.13. Разрешение типов во вложенном объединении
SELECT NULL UNION SELECT NULL UNION SELECT 1;
ERROR:
UNION types text and integer cannot be matched
4
Так же, как домены воспринимаются при выборе операторов и функций, доменные типы могут сохраняться в конструкции UNION или подобной, если
пользователь позаботится о том, чтобы все входные данные приводились к этому типу явно или неявно. В противном случае предпочтение будет отдано
базовому типу домена.
354Преобразование типов
Эта ошибка возникает из-за того, что PostgreSQL воспринимает множественные UNION как пары с
вложенными операциями, то есть как запись
(SELECT NULL UNION SELECT NULL) UNION SELECT 1;
Внутренний UNION разрешается как выдающий тип text, согласно правилам, приведённым выше.
Затем внешний UNION получает на вход типы text и integer, что и приводит к показанной ошибке.
Эту проблему можно устранить, сделав так, чтобы у самого левого UNION минимум с одной стороны
были данные желаемого типа результата.
Операции INTERSECT и EXCEPT также разрешаются по парам. Однако остальные конструкции, опи-
санные в этом разделе, рассматривают все входные данные сразу.
10.6. Выходные столбцы SELECT
Правила, описанные в предыдущих разделах, распространяются на присвоение типов данных, кро-
ме unknown, во всех выражениях в запросах SQL, за исключением бестиповых буквальных значе-
ний, принимающих вид простых выходных столбцов команды SELECT. Например, в запросе
SELECT 'Hello World';
ничто не говорит о том, какой тип должно принимать строковое буквальное значение. В этой си-
туации PostgreSQL разрешит тип такого значения как text.
Когда SELECT является одной из ветвей конструкции UNION (или INTERSECT/EXCEPT) или когда он
находится внутри INSERT ... SELECT, это правило не действует, так как более высокий приоритет
имеют правила, описанные в предыдущих разделах. В первом случае тип бестипового буквального
значения может быть получен из другой ветви UNION, а во втором — из целевого столбца.
Списки RETURNING в данном контексте воспринимаются так же, как выходные списки SELECT.
Примечание
До PostgreSQL 10 этого правила не было и бестиповые буквальные значения в выход-
ном списке SELECT оставались с типом unknown. Это имело различные негативные по-
следствия, так что было решено это изменить.