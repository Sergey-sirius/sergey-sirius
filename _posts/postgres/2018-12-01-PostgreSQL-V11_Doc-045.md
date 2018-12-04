---
layout: post
title: Глава 45. PL/Perl — процедурный язык Perl
description: ""
tags: [PostgreSQL, PostgreSQL_Book_11]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 45. PL/Perl — процедурный язык Perl

PL/Perl — это загружаемый процедурный язык, позволяющий реализовывать функции PostgreSQL
на языке программирования Perl.
Основным преимуществом PL/Perl является то, что он позволяет применять в сохранённых функци-
ях множество функций и операторов «перемалывания строк», имеющихся в Perl. Разобрать слож-
ные строки на языке Perl может быть гораздо проще, чем используя строковые функции и управ-
ляющие структуры в PL/pgSQL.
Чтобы установить PL/Perl в определённую базу данных, выполните команду CREATE EXTENSION
plperl.
Подсказка
Если язык устанавливается в template1, он будет автоматически установлен во все
создаваемые впоследствии базы данных.
Примечание
Пользователи, имеющие дело с исходным кодом, должны явно включить сборку PL/
Perl в процессе установки. (За дополнительными сведениями обратитесь к Главе 16.)
Пользователи двоичных пакетов могут найти PL/Perl в отдельном модуле.
45.1. Функции на PL/Perl и их аргументы
Чтобы создать функцию на языке PL/Perl, используйте стандартный синтаксис CREATE FUNCTION:
CREATE FUNCTION имя_функции (типы-аргументов) RETURNS тип-результата AS $$
# Тело функции на PL/Perl
$$ LANGUAGE plperl;
Тело функции содержит обычный код Perl. Фактически, код обвязки PL/Perl помещает этот код
в подпрограмму Perl. Функция PL/Perl вызывается в скалярном контексте, так что она не может
вернуть список. Не скалярные значения (массивы, записи и множества) можно вернуть по ссылке,
как описывается ниже.
В процедуре PL/Perl возвращаемое из кода Perl значение игнорируется.
PL/Perl также поддерживает анонимные блоки кода, которые выполняются оператором DO:
DO $$
# Код PL/Perl
$$ LANGUAGE plperl;
Анонимный блок кода не принимает аргументы, а любое значение, которое он мог бы вернуть,
отбрасывается. В остальном он работает подобно коду функции.
Примечание
Использовать вложенные именованные подпрограммы в Perl опасно, особенно если они
обращаются к лексическим переменным в окружающей области. Так как функция PL/
Perl оборачивается в подпрограмму, любая именованная функция внутри неё будет вло-
женной. Вообще гораздо безопаснее создавать анонимные подпрограммы и вызывать
их по ссылке на код. Дополнительную информацию вы можете получить на странице
руководства man perldiag, в описании ошибок Variable "%s" will not stay shared
(Переменная "%s" не останется разделяемой) и Variable "%s" is not available (Пе-
1195PL/Perl — процедурный язык Perl
ременная "%s" недоступна), либо найти в Интернете по ключевым словам «perl nested
named subroutine» (perl вложенная именованная подпрограмма).
Синтаксис команды CREATE FUNCTION требует, чтобы тело функции было записано как строковая
константа. Обычно для этого удобнее всего заключать строковую константу в доллары (см. Под-
раздел 4.1.2.4). Если вы решите применять синтаксис спецпоследовательностей E'', вам придётся
дублировать апострофы (') и обратную косую черту (\) в теле функции (см. Подраздел 4.1.2.1).
Аргументы и результат обрабатываются как и в любой другой подпрограмме на Perl: аргументы
передаются в @_, а результирующим значением будет указанное в return или полученное в по-
следнем выражении, вычисленном в функции.
Например, функцию, возвращающую большее из двух целых чисел, можно определить так:
CREATE FUNCTION perl_max (integer, integer) RETURNS integer AS $$
if ($_[0] > $_[1]) { return $_[0]; }
return $_[1];
$$ LANGUAGE plperl;
Примечание
Аргументы будут преобразованы из кодировки базы данных в UTF-8 для использования
в PL/Perl, а при выходе снова будут преобразованы из UTF-8 в кодировку базы данных.
Если функции передаётся NULL-значение SQL, значением аргумента в Perl станет «undefined».
Показанное выше определение функции будет не очень хорошо обрабатывать значения NULL (в
действительности они будут восприняты как нули). Мы могли бы добавить указание STRICT в это
определение, чтобы PostgreSQL поступал немного разумнее: при передаче значения NULL функ-
ция вовсе не будет вызываться, будет сразу возвращён результат NULL. С другой стороны, мы
могли бы проверить значения undefined в теле функции. Например, предположим, что нам нужна
функция perl_max, которая с одним аргументом NULL и вторым аргументом не NULL должна воз-
вращать не NULL, а второй аргумент:
CREATE FUNCTION perl_max (integer, integer) RETURNS integer AS $$
my ($x, $y) = @_;
if (not defined $x) {
return undef if not defined $y;
return $y;
}
return $x if not defined $y;
return $x if $x > $y;
return $y;
$$ LANGUAGE plperl;
Как показано выше, чтобы выдать значение SQL NULL, нужно вернуть значение undefined. Это
можно сделать и в строгой, и в нестрогой функции.
Всё в аргументах функции, что не является ссылкой, является строкой, то есть стандартным для
PostgreSQL внешним текстовым представлением соответствующего типа данных. В случае с обыч-
ными числовыми или текстовыми типами, Perl просто воспринимает их должным образом, и про-
граммист, как правило, может об этом не думать. Однако в более сложных случаях может потре-
боваться преобразовать аргумент в форму, подходящую для использования в Perl. Например, для
преобразования типа bytea в двоичное значение можно использовать функцию decode_bytea.
Аналогично, значения, передаваемые в PostgreSQL, должны быть в формате внешнего текстового
представления. Например, для подготовки двоичных данных к возврату в значении bytea можно
воспользоваться функцией encode_bytea.
Perl может возвращать массивы PostgreSQL как ссылки на массивы Perl. Например, так:
1196PL/Perl — процедурный язык Perl
CREATE OR REPLACE function returns_array()
RETURNS text[][] AS $$
return [['a"b','c,d'],['e\\f','g']];
$$ LANGUAGE plperl;
select returns_array();
Perl передаёт массивы PostgreSQL как объект, сопоставленный с PostgreSQL::InServer::ARRAY. С
этим объектом можно работать как со ссылкой на массив или строкой, что допускает обратную
совместимость с кодом Perl, написанным для PostgreSQL версии до 9.1. Например:
CREATE OR REPLACE FUNCTION concat_array_elements(text[]) RETURNS TEXT AS $$
my $arg = shift;
my $result = "";
return undef if (!defined $arg);
# в качестве ссылки на массив
for (@$arg) {
$result .= $_;
}
# также работает со строкой
$result .= $arg;
return $result;
$$ LANGUAGE plperl;
SELECT concat_array_elements(ARRAY['PL','/','Perl']);
Примечание
Многомерные массивы представляются как ссылки на массивы меньшей размерности
со ссылками — этот способ хорошо знаком каждому программисту на Perl.
Аргументы составного типа передаются функции как ссылки на хеши. Ключами хеша являются
имена атрибутов составного типа. Например:
CREATE TABLE employee (
name text,
basesalary integer,
bonus integer
);
CREATE FUNCTION empcomp(employee) RETURNS integer AS $$
my ($emp) = @_;
return $emp->{basesalary} + $emp->{bonus};
$$ LANGUAGE plperl;
SELECT name, empcomp(employee.*) FROM employee;
Функция на PL/Perl может вернуть результат составного типа, применяя тот же подход: возвратить
ссылку на хеш с требуемыми атрибутами. Например, так:
CREATE TYPE testrowperl AS (f1 integer, f2 text, f3 text);
CREATE OR REPLACE FUNCTION perl_row() RETURNS testrowperl AS $$
return {f2 => 'hello', f1 => 1, f3 => 'world'};
$$ LANGUAGE plperl;
1197PL/Perl — процедурный язык Perl
SELECT * FROM perl_row();
Столбцы объявленного типа результата, отсутствующие в хеше, будут возвращены как значения
NULL.
Подобным образом в виде ссылки на хеш могут быть возвращены выходные аргументы процедуры:
CREATE PROCEDURE perl_triple(INOUT a integer, INOUT b integer) AS $$
my ($a, $b) = @_;
return {a => $a * 3, b => $b * 3};
$$ LANGUAGE plperl;
CALL perl_triple(5, 10);
Функции на PL/Perl могут также возвращать множества со скалярными или составными типами.
Обычно желательно возвращать результат по одной строке, чтобы сократить время подготовки
с одной стороны, и чтобы не потребовалось накапливать весь набор данных в памяти, с другой.
Это можно реализовать с помощью функции return_next, как показано ниже. Заметьте, что после
последнего вызова return_next, нужно поместить return или (что лучше) return undef.
CREATE OR REPLACE FUNCTION perl_set_int(int)
RETURNS SETOF INTEGER AS $$
foreach (0..$_[0]) {
return_next($_);
}
return undef;
$$ LANGUAGE plperl;
SELECT * FROM perl_set_int(5);
CREATE OR REPLACE FUNCTION perl_set()
RETURNS SETOF testrowperl AS $$
return_next({ f1 => 1, f2 => 'Hello', f3 => 'World' });
return_next({ f1 => 2, f2 => 'Hello', f3 => 'PostgreSQL' });
return_next({ f1 => 3, f2 => 'Hello', f3 => 'PL/Perl' });
return undef;
$$ LANGUAGE plperl;
Для небольших наборов данных можно также вернуть ссылку на массив, содержащий скаляры,
ссылки на массивы, либо ссылки на хеши для простых типов, типов массивов и составных типов,
соответственно. Ниже приведена пара простых примеров, показывающих, как возвратить весь на-
бор данных в виде ссылки на массив:
CREATE OR REPLACE FUNCTION perl_set_int(int) RETURNS SETOF INTEGER AS $$
return [0..$_[0]];
$$ LANGUAGE plperl;
SELECT * FROM perl_set_int(5);
CREATE OR REPLACE FUNCTION perl_set() RETURNS SETOF testrowperl AS $$
return [
{ f1 => 1, f2 => 'Hello', f3 => 'World' },
{ f1 => 2, f2 => 'Hello', f3 => 'PostgreSQL' },
{ f1 => 3, f2 => 'Hello', f3 => 'PL/Perl' }
];
$$ LANGUAGE plperl;
SELECT * FROM perl_set();
Если вы хотите использовать в своём коде strict, у вас есть несколько вариантов. Для временного
глобального использования вы можете задать для plperl.use_strict значение true командой SET.
1198PL/Perl — процедурный язык Perl
Это повлияет на компилируемые впоследствии функции PL/Perl, но не на функции, уже скомпили-
рованные в текущем сеансе. Для постоянного глобального использования вы можете присвоить
параметру plperl.use_strict значение true в файле postgresql.conf.
Для постоянного использования strict в опредёлённых функциях вы можете просто написать:
use strict;
в начале тела этих функций.
Вы также можете использовать указания feature в use, если используете Perl версии 5.10.0 или
новее.
45.2. Значения в PL/Perl
Значения аргументов, передаваемые в код функции PL/Perl, представляют собой просто входные
аргументы, преобразованные в текстовый вид (так же, как при выводе оператором SELECT). И на-
оборот, команды return и return_next могут принять любую строку, соответствующую формату
ввода для объявленного типа результата функции.
45.3. Встроенные функции
45.3.1. Обращение к базе данных из PL/Perl
Обращаться к самой базе данных из кода Perl можно, используя следующие функции:
spi_exec_query(запрос [, макс-строк])
spi_exec_query выполняет команду SQL и возвращает весь набор строк в виде ссылки на мас-
сив хешей. Эту функцию следует использовать, только если вы знаете, что набор будет
относительно небольшим. Так выглядит пример запроса (SELECT) с дополнительно заданным
максимальным числом строк:
$rv = spi_exec_query('SELECT * FROM my_table', 5);
Этот запрос возвращает не больше 5 строк из таблицы my_table. Если в my_table есть столбец
my_column, получить его значение из строки $i результата можно следующим образом:
$foo = $rv->{rows}[$i]->{my_column};
Общее число строк, возвращённых запросом SELECT, можно получить так:
$nrows = $rv->{processed}
Так можно выполнить команду другого типа:
$query = "INSERT INTO my_table VALUES (1, 'test')";
$rv = spi_exec_query($query);
Затем можно получить статус команды (например, SPI_OK_INSERT) следующим образом:
$res = $rv->{status};
Чтобы получить число затронутых строк, выполните:
$nrows = $rv->{processed};
Полный пример:
CREATE TABLE test (
i int,
v varchar
);
1199PL/Perl — процедурный язык Perl
INSERT
INSERT
INSERT
INSERT
INTO
INTO
INTO
INTO
test
test
test
test
(i,
(i,
(i,
(i,
v)
v)
v)
v)
VALUES
VALUES
VALUES
VALUES
(1,
(2,
(3,
(4,
'first line');
'second line');
'third line');
'immortal');
CREATE OR REPLACE FUNCTION test_munge() RETURNS SETOF test AS $$
my $rv = spi_exec_query('select i, v from test;');
my $status = $rv->{status};
my $nrows = $rv->{processed};
foreach my $rn (0 .. $nrows - 1) {
my $row = $rv->{rows}[$rn];
$row->{i} += 200 if defined($row->{i});
$row->{v} =~ tr/A-Za-z/a-zA-Z/ if (defined($row->{v}));
return_next($row);
}
return undef;
$$ LANGUAGE plperl;
SELECT * FROM test_munge();
spi_query(команда)
spi_fetchrow(cursor)
spi_cursor_close(cursor)
Функции spi_query и spi_fetchrow применяются в паре, когда набор строк может быть очень
большим или когда нужно возвращать строки по мере их поступления. Функция spi_fetchrow
работает только с spi_query. Следующий пример показывает, как использовать их вместе:
CREATE TYPE foo_type AS (the_num INTEGER, the_text TEXT);
CREATE OR REPLACE FUNCTION lotsa_md5 (INTEGER) RETURNS SETOF foo_type AS $$
use Digest::MD5 qw(md5_hex);
my $file = '/usr/share/dict/words';
my $t = localtime;
elog(NOTICE, "opening file $file at $t" );
open my $fh, '<', $file # здесь мы обращаемся к файлу!
or elog(ERROR, "cannot open $file for reading: $!");
my @words = <$fh>;
close $fh;
$t = localtime;
elog(NOTICE, "closed file $file at $t");
chomp(@words);
my $row;
my $sth = spi_query("SELECT * FROM generate_series(1,$_[0]) AS b(a)");
while (defined ($row = spi_fetchrow($sth))) {
return_next({
the_num => $row->{a},
the_text => md5_hex($words[rand @words])
});
}
return;
$$ LANGUAGE plperlu;
SELECT * from lotsa_md5(500);
Обычно вызов spi_fetchrow нужно повторять, пока не будет получен результат undef, показы-
вающий, что все строки уже прочитаны. Курсор, возвращаемый функцией spi_query, автома-
тически освобождается, когда spi_fetchrow возвращает undef. Если вы не хотите читать все
строки, освободите курсор, выполнив spi_cursor_close, чтобы не допустить утечки памяти.
1200PL/Perl — процедурный язык Perl
spi_prepare(команда, типы аргументов)
spi_query_prepared(план, аргументы)
spi_exec_prepared(план [, атрибуты], аргументы)
spi_freeplan(план)
Функции spi_prepare, spi_query_prepared, spi_exec_prepared и spi_freeplan реализуют ту
же функциональность, но для подготовленных запросов. Функция spi_prepare принимает стро-
ку запроса с нумерованными местозаполнителями аргументов ($1, $2 и т. д.) и список строк
с типами аргументов:
$plan = spi_prepare('SELECT * FROM test WHERE id > $1 AND name = $2',
'INTEGER', 'TEXT');
План запроса, подготовленный вызовом spi_prepare, можно использовать вместо строки за-
проса либо в spi_exec_prepared, возвращающей тот же результат, что и spi_exec_query, ли-
бо в spi_query_prepared, возвращающей курсор так же, как spi_query, который затем можно
передать в spi_fetchrow. В необязательном втором параметре spi_exec_prepared можно пере-
дать хеш с атрибутами; в настоящее время поддерживается только атрибут limit, задающий
максимальное число строк, которое может вернуть запрос.
Подготовленные запросы хороши тем, что позволяют использовать единожды подготовленный
план для неоднократного выполнения запроса. Когда план оказывается не нужен, его можно
освободить, вызвав spi_freeplan:
CREATE OR REPLACE FUNCTION init() RETURNS VOID AS $$
$_SHARED{my_plan} = spi_prepare('SELECT (now() + $1)::date AS now',
'INTERVAL');
$$ LANGUAGE plperl;
CREATE OR REPLACE FUNCTION add_time( INTERVAL ) RETURNS TEXT AS $$
return spi_exec_prepared(
$_SHARED{my_plan},
$_[0]
)->{rows}->[0]->{now};
$$ LANGUAGE plperl;
CREATE OR REPLACE FUNCTION done() RETURNS VOID AS $$
spi_freeplan( $_SHARED{my_plan});
undef $_SHARED{my_plan};
$$ LANGUAGE plperl;
SELECT init();
SELECT add_time('1 day'), add_time('2 days'), add_time('3 days');
SELECT done();
add_time | add_time | add_time
------------+------------+------------
2005-12-10 | 2005-12-11 | 2005-12-12
Заметьте, что параметры для spi_prepare обозначаются как $1, $2, $3 и т. д., так что по воз-
можности не записывайте строки запросов в двойных кавычках, чтобы не спровоцировать труд-
ноуловимые ошибки.
Ещё
один
пример,
spi_exec_prepared:
иллюстрирующий
использование
необязательного
CREATE TABLE hosts AS SELECT id, ('192.168.1.'||id)::inet AS address
FROM generate_series(1,3) AS id;
CREATE OR REPLACE FUNCTION init_hosts_query() RETURNS VOID AS $$
$_SHARED{plan} = spi_prepare('SELECT * FROM hosts
1201
параметраPL/Perl — процедурный язык Perl
WHERE address << $1', 'inet');
$$ LANGUAGE plperl;
CREATE OR REPLACE FUNCTION query_hosts(inet) RETURNS SETOF hosts AS $$
return spi_exec_prepared(
$_SHARED{plan},
{limit => 2},
$_[0]
)->{rows};
$$ LANGUAGE plperl;
CREATE OR REPLACE FUNCTION release_hosts_query() RETURNS VOID AS $$
spi_freeplan($_SHARED{plan});
undef $_SHARED{plan};
$$ LANGUAGE plperl;
SELECT init_hosts_query();
SELECT query_hosts('192.168.1.0/30');
SELECT release_hosts_query();
query_hosts
-----------------
(1,192.168.1.1)
(2,192.168.1.2)
(2 rows)
spi_commit()
spi_rollback()
Эти функции фиксируют или откатывают текущую транзакцию. Они могут вызываться только в
процедурах или в анонимных блоках кода (в команде DO), вызываемых из кода верхнего уровня.
(Заметьте, что выполнить SQL-команды COMMIT или ROLLBACK через spi_exec_query или подоб-
ную функцию нельзя. Соответствующие операции могут выполняться только данными функци-
ями.) После завершения одной транзакции следующая начинается автоматически, отдельной
функции для этого нет.
Пример:
CREATE PROCEDURE transaction_test1()
LANGUAGE plperl
AS $$
foreach my $i (0..9) {
spi_exec_query("INSERT INTO test1 (a) VALUES ($i)");
if ($i % 2 == 0) {
spi_commit();
} else {
spi_rollback();
}
}
$$;
CALL transaction_test1();
45.3.2. Вспомогательные функции в PL/Perl
elog(уровень, сообщение)
Выдаёт служебное сообщение или сообщение об ошибке. Возможные уровни сообщений: DEBUG
(ОТЛАДКА), LOG (СООБЩЕНИЕ), INFO (ИНФОРМАЦИЯ), NOTICE (ЗАМЕЧАНИЕ), WARNING (ПРЕ-
ДУПРЕЖДЕНИЕ) и ERROR (ОШИБКА). С уровнем ERROR выдаётся ошибка; если она не перехва-
1202PL/Perl — процедурный язык Perl
тывается окружающим кодом Perl, она распространяется в вызывающий запрос, что приводит к
прерыванию текущей транзакции или подтранзакции. По сути то же самое делает команда die
языка Perl. При использовании других уровней происходит просто вывод сообщения с задан-
ным уровнем важности. Будут ли сообщения определённого уровня передаваться клиенту и/или
записываться в журнал, определяется конфигурационными параметрами log_min_messages и
client_min_messages. За дополнительными сведениями обратитесь к Главе 19.
quote_literal(строка)
Оформляет переданную строку для использования в качестве текстовой строки в SQL-опера-
торе. Включённые в неё апострофы и обратная косая черта при этом дублируются. Заметьте,
что quote_literal возвращает undef, когда получает аргумент undef; если такие аргументы
возможны, часто лучше использовать quote_nullable.
quote_nullable(строка)
Оформляет переданную строку для использования в качестве текстовой строки в SQL-операто-
ре; либо, если поступает аргумент undef, возвращает строку "NULL" (без кавычек). Символы
апостроф и обратная косая черта дублируются должным образом.
quote_ident(строка)
Оформляет переданную строку для использования в качестве идентификатора в SQL-операто-
ре. При необходимости идентификатор заключается в кавычки (например, если он содержит
символы, недопустимые в открытом виде, или буквы в разном регистре). Если переданная стро-
ка содержит кавычки, они дублируются.
decode_bytea(строка)
Возвращает неформатированные двоичные данные, представленные содержимым заданной
строки, которая должна быть закодирована как bytea.
encode_bytea(строка)
Возвращает закодированные в виде bytea двоичные данные, содержащиеся в переданной стро-
ке.
encode_array_literal(массив)
encode_array_literal(массив, разделитель)
Возвращает содержимое указанного массива в виде строки в формате массива (см. Подраз-
дел 8.15.2). Возвращает значение аргумента неизменённым, если это не ссылка не массив. Раз-
делитель элементов в строке массива по умолчанию — ", " (если разделитель не определён
или undef).
encode_typed_literal(значение, имя_типа)
Преобразует переменную Perl в значение типа данных, указанного во втором аргументе, и воз-
вращает строковое представление этого значения. Корректно обрабатывает вложенные масси-
вы и значения составных типов.
encode_array_constructor(массив)
Возвращает содержимое переданного массива в виде строки в формате конструктора массива
(см. Подраздел 4.2.12). Отдельные значения заключаются в кавычки функцией quote_nullable.
Возвращает значение аргумента, заключённое в кавычки функцией quote_nullable, если ар-
гумент — не ссылка на массив.
looks_like_number(строка)
Возвращает значение true, если содержимое переданной строки похоже на число, по правилам
Perl, и false в обратном случае. Возвращает undef для аргумента undef. Ведущие и замыкающие
1203PL/Perl — процедурный язык Perl
пробелы игнорируются. Строки Inf и Infinity считаются представляющими число (бесконеч-
ность).
is_array_ref(аргумент)
Возвращает значение true, если переданный аргумент можно воспринять как ссылку на массив,
то есть это ссылка на ARRAY или PostgreSQL::InServer::ARRAY. В противном случае возвращает
false.
45.4. Глобальные значения в PL/Perl
Вы можете использовать для хранения данных, включая ссылки на код, глобальный хеш %_SHARED.
Эти данные будут сохраняться между вызовами функции на протяжении всего текущего сеанса.
Простой пример работы с разделяемыми данными:
CREATE OR REPLACE FUNCTION set_var(name text, val text) RETURNS text AS $$
if ($_SHARED{$_[0]} = $_[1]) {
return 'ok';
} else {
return "cannot set shared variable $_[0] to $_[1]";
}
$$ LANGUAGE plperl;
CREATE OR REPLACE FUNCTION get_var(name text) RETURNS text AS $$
return $_SHARED{$_[0]};
$$ LANGUAGE plperl;
SELECT set_var('sample', 'Hello, PL/Perl!
SELECT get_var('sample');
How''s tricks?');
Это чуть более сложный пример, в котором используется ссылка на код:
CREATE OR REPLACE FUNCTION myfuncs() RETURNS void AS $$
$_SHARED{myquote} = sub {
my $arg = shift;
$arg =~ s/(['\\])/\\$1/g;
return "'$arg'";
};
$$ LANGUAGE plperl;
SELECT myfuncs(); /* инициализация функции */
/* Определение функции, использующей функцию заключения в кавычки */
CREATE OR REPLACE FUNCTION use_quote(TEXT) RETURNS text AS $$
my $text_to_quote = shift;
my $qfunc = $_SHARED{myquote};
return &$qfunc($text_to_quote);
$$ LANGUAGE plperl;
(Код выше можно было бы упростить до однострочной команды return
>($_[0]); в ущерб читаемости.)
$_SHARED{myquote}-
По соображениям безопасности, PL/Perl выполняет функции, вызываемые некоторой ролью SQL,
в отдельном интерпретаторе Perl, выделенном для этой роли. Это предотвращает случайное или
злонамеренное влияние одного пользователя на поведение функций PL/Perl другого пользователя.
В каждом интерпретаторе будет своё значение переменной %_SHARED и собственное глобальное
состояние. Таким образом, две функции PL/Perl будут разделять одно значение %_SHARED, только
если они выполняются одной ролью SQL. В приложении, выполняющем код в одном сеансе с раз-
1204PL/Perl — процедурный язык Perl
ными ролями SQL (вызывающем функции SECURITY DEFINER, использующем команду SET ROLE и т.
д.) может понадобиться явно предпринять дополнительные меры, чтобы функции на PL/Perl могли
разделять данные через %_SHARED. Для этого сначала установите для функций, которые должны
взаимодействовать, одного владельца, а затем задайте для них свойство SECURITY DEFINER. Разу-
меется, при этом нужно позаботиться о том, чтобы эти функции не могли сделать ничего непреду-
смотренного.
45.5. Доверенный и недоверенный PL/Perl
Обычно PL/Perl устанавливается в базу данных как «доверенный» язык программирования с име-
нем plperl. При этом в целях безопасности определённые операции в Perl запрещаются. Вообще
говоря, запрещаются все операции, взаимодействующие с окружением. В том числе, это опера-
ции с файлами, require и use (для внешних модулей). Поэтому функции на PL/Perl, в отличие от
функций на C, никаким образом не могут взаимодействовать с внутренними механизмами сервера
баз данных или обращаться к операционной системе с правами серверного процесса. Вследствие
этого, использовать этот язык можно разрешить любому непривилегированному пользователю баз
данных.
В следующем примере показана функция, которая не будет работать, потому что операции с фай-
ловой системы запрещены по соображениям безопасности:
CREATE FUNCTION badfunc() RETURNS integer AS $$
my $tmpfile = "/tmp/badfile";
open my $fh, '>', $tmpfile
or elog(ERROR, qq{could not open the file "$tmpfile": $!});
print $fh "Testing writing to a file\n";
close $fh or elog(ERROR, qq{could not close the file "$tmpfile": $!});
return 1;
$$ LANGUAGE plperl;
Создать эту функцию не удастся, так как при проверке её правильности будет обнаружено исполь-
зование запрещённого оператора.
Иногда возникает желание написать на Perl код, функциональность которого не будет ограничи-
ваться. Например, может потребоваться функция на Perl, которая будет посылать почту. Для таких
потребностей PL/Perl также можно установить как «недоверенный» язык (обычно его называют
PL/PerlU). В этом случае будут доступны все возможности языка Perl. Устанавливая язык, укажите
имя plperlu, чтобы выбрать недоверенную вариацию PL/Perl.
Автор функции на PL/PerlU должен позаботиться о том, чтобы эту функцию нельзя было использо-
вать не по назначению, так как она может делать всё, что может пользователь с правами админи-
стратора баз данных. Заметьте, что СУБД позволяет создавать функции на недоверенных языках
только суперпользователям базы данных.
Если показанная выше функция будет создана суперпользователем, и при этом будет выбран язык
plperlu, она выполнится успешно.
Таким же образом, в анонимном блоке кода на Perl разрешены абсолютно любые операции, если
в качестве языка вместо plperl выбирается plperlu, но выполнять этот код должен суперпользо-
ватель.
Примечание
Тогда как функции на PL/Perl исполняются отдельными интерпретаторами Perl для
каждой роли SQL, все функции на PL/PerlU, вызываемые в рамках сеанса, исполняются
в одном интерпретаторе Perl (отличном от тех, что исполняют функции PL/Perl). Бла-
годаря этому, функции PL/PerlU могут свободно разделять общие данные, но между
функциями PL/Perl и PL/PerlU взаимодействие невозможно.
1205PL/Perl — процедурный язык Perl
Примечание
Perl поддерживает работу нескольких интерпретаторов в одном процессе, только если
он был собран с нужными флагами, а именно, с флагом usemultiplicity или с флагом
useithreads. (В отсутствие веских причин использовать потоки предпочтительным яв-
ляется вариант usemultiplicity. Дополнительную информацию вы можете получить
на странице man perlembed.) При использовании PL/Perl с версией Perl, собранной без
этих флагов, в рамках сеанса можно будет запустить только один интерпретатор Perl,
так что в сеансе будет возможно выполнять либо функции PL/PerlU, либо функции PL/
Perl (и вызывать их должна одна роль SQL).
45.6. Триггеры на PL/Perl
PL/Perl можно использовать для написания триггерных функций. В триггерной функции хеш-мас-
сив $_TD содержит информацию о произошедшем событии триггера. $_TD — глобальная перемен-
ная, которая получает нужное локальное значение при каждом вызове триггера. Хеш-массив $_TD
содержит следующие поля:
$_TD->{new}{foo}
Новое значение столбца foo
$_TD->{old}{foo}
Старое значение столбца foo
$_TD->{name}
Имя вызываемого триггера
$_TD->{event}
Событие триггера: INSERT, UPDATE, DELETE, TRUNCATE или UNKNOWN
$_TD->{when}
Когда вызывается триггер: BEFORE (ДО), AFTER (ПОСЛЕ), INSTEAD OF (ВМЕСТО) или UNKNOWN
(НЕИЗВЕСТНО)
$_TD->{level}
Уровень триггера: ROW (СТРОКА), STATEMENT (ОПЕРАТОР) или UNKNOWN (НЕИЗВЕСТНЫЙ)
$_TD->{relid}
OID таблицы, для которой сработал триггер
$_TD->{table_name}
Имя таблицы, для которой сработал триггер
$_TD->{relname}
Имя таблицы, для которой сработал триггер. Это обращение устарело и может быть ликвиди-
ровано в будущем выпуске. Используйте вместо него $_TD->{table_name}.
$_TD->{table_schema}
Имя схемы, содержащей таблицу, для которой сработал триггер
$_TD->{argc}
Число аргументов в триггерной функции
1206PL/Perl — процедурный язык Perl
@{$_TD->{args}}
Аргументы триггерной функции. Не определено, если $_TD->{argc} равно 0.
В триггерах уровня строки возможны следующие варианты возврата:
return;
Выполнить операцию
"SKIP"
Не выполнять операцию
"MODIFY"
Указывает, что строка NEW была изменена триггерной функцией
Следующий пример триггерной функции иллюстрирует описанные выше варианты:
CREATE TABLE test (
i int,
v varchar
);
CREATE OR REPLACE FUNCTION valid_id() RETURNS trigger AS $$
if (($_TD->{new}{i} >= 100) || ($_TD->{new}{i} <= 0)) {
return "SKIP";
# пропустить команду INSERT/UPDATE
} elsif ($_TD->{new}{v} ne "immortal") {
$_TD->{new}{v} .= "(modified by trigger)";
return "MODIFY"; # изменить строку и выполнить команду INSERT/UPDATE
} else {
return;
# выполнить команду INSERT/UPDATE
}
$$ LANGUAGE plperl;
CREATE TRIGGER test_valid_id_trig
BEFORE INSERT OR UPDATE ON test
FOR EACH ROW EXECUTE FUNCTION valid_id();
45.7. Событийные триггеры на PL/Perl
PL/Perl можно использовать для написания функций событийных триггеров. В функции событий-
ного триггера хеш-массив $_TD содержит информацию о произошедшем событии триггера. $_TD —
глобальная переменная, которая получает нужное локальное значение при каждом вызове триг-
гера. Хеш-массив $_TD содержит следующие поля:
$_TD->{event}
Имя события, при котором срабатывает этот триггер.
$_TD->{tag}
Тег команды, для которой срабатывает этот триггер.
Возвращаемое значение триггерной функции игнорируется.
Следующий пример функции событийного триггера иллюстрирует описанное выше:
CREATE OR REPLACE FUNCTION perlsnitch() RETURNS event_trigger AS $$
elog(NOTICE, "perlsnitch: " . $_TD->{event} . " " . $_TD->{tag} . " ");
$$ LANGUAGE plperl;
CREATE EVENT TRIGGER perl_a_snitch
1207PL/Perl — процедурный язык Perl
ON ddl_command_start
EXECUTE FUNCTION perlsnitch();
45.8. Внутренние особенности PL/Perl
45.8.1. Конфигурирование
В этом разделе описываются параметры конфигурации, влияющие на работу PL/Perl.
plperl.on_init (string)
Задаёт код Perl, который будет выполняться при первой инициализации интерпретатора Perl, до
того, как он получает специализацию plperl или plperlu. Когда этот код выполняется, функ-
ции SPI ещё не доступны. Если выполнение кода завершается ошибкой, инициализация интер-
претатора прерывается и ошибка распространяется в вызывающий запрос, в результате чего
текущая транзакция или подтранзакция прерывается.
Размер этого кода ограничивается одной строкой. Более объёмный код можно поместить в мо-
дуль и загрузить этот модуль в строке on_init. Например:
plperl.on_init = 'require "plperlinit.pl"'
plperl.on_init = 'use lib "/my/app"; use MyApp::PgInit;'
Любые модули, загруженные в plperl.on_init, явно или неявно, будут доступны для исполь-
зования в коде на языке plperl. Это может создать угрозу безопасности. Чтобы определить,
какие модули были загружены, можно выполнить:
DO 'elog(WARNING, join ", ", sort keys %INC)' LANGUAGE plperl;
Если библиотека plperl включена в shared_preload_libraries, инициализация произойдёт в глав-
ном процессе (postmaster) и в этом случае необходимо очень серьёзно оценить риск наруше-
ния работоспособности этого процесса. Основной смысл использовать эту возможность в том,
чтобы модули Perl, подключаемые в plperl.on_init, загружались только при запуске главно-
го процесса, и это исключало бы издержки загрузки для отдельных сеансов. Однако, имейте
в виду, что эти издержки исключаются только при загрузке в сеансе первого интерпретатора
Perl — будь то PL/PerlU или PL/Perl для первой SQL-роли, вызывающей функцию на PL/Perl. Лю-
бые дополнительные интерпретаторы Perl, создаваемые в сеансе базы данных, должны будут
выполнять plperl.on_init заново. Также учтите, что в Windows предварительная загрузка не
даёт никакого выигрыша, так как интерпретатор Perl, созданный в главном процессе, не пере-
даётся дочерним процессам.
Задать этот параметр можно только в postgresql.conf или в командной строке при запуске
сервера.
plperl.on_plperl_init (string)
plperl.on_plperlu_init (string)
В этих параметрах задаётся код Perl, который будет выполняться в момент, когда интерпрета-
тор Perl получает специализацию plperl или plperlu, соответственно. Это произойдёт, когда в
рамках сеанса будет первый раз вызвана функция на PL/Perl или PL/PerlU, либо когда потребу-
ется дополнительный интерпретатор при использовании другого языка или при вызове функ-
ции PL/Perl новой SQL-ролью. Этот код выполняется после инициализации, произведённой в
plperl.on_init. Однако функции SPI в момент исполнения этого кода ещё не доступны. Код в
plperl.on_plperl_init запускается после того, как интерпретатор «помещается под замок»,
так что в нём разрешаются только доверенные операции.
Если этот код завершается ошибкой, инициализация прерывается и ошибка распространяется
в вызывающий запрос, что приводит к прерыванию текущей транзакции или подтранзакции.
При этом любые действия, уже произведённые в Perl, не будут отменены; однако использовать-
ся этот интерпретатор больше не будет. При следующей попытке использовать этот язык си-
стема попытается заново инициализировать свежий интерпретатор Perl.
1208PL/Perl — процедурный язык Perl
Изменять эти параметры разрешено только суперпользователям. Хотя изменить их можно в
рамках сеанса, такие изменения не повлияют на работу интерпретаторов Perl, задействованных
для выполнения функций ранее.
plperl.use_strict (boolean)
При значении, равном true, последующая компиляция функций PL/Perl будет выполняться с
включённым указанием strict. Этот параметр не влияет на функции, уже скомпилированные
в текущем сеансе.
45.8.2. Ограничения и недостающие возможности
Следующие возможности в настоящее время в PL/Perl отсутствуют, но их реализация будет желан-
ной доработкой.
• Функции на PL/Perl не могут напрямую вызывать друг друга.
• SPI ещё не полностью реализован.
• Если вы выбираете очень большие наборы данных, используя spi_exec_query, вы должны по-
нимать, что все эти данные загружаются в память. Вы можете избежать этого, используя пару
функций spi_query/spi_fetchrow, как показано ранее.
Похожая проблема возникает, если функция, возвращающая множество, передаёт в
PostgreSQL большое число строк, выполняя return. Этой проблемы так же можно избежать,
выполняя для каждой возвращаемой строки return_next, как показано ранее.
• Когда сеанс завершается штатно, не по причине критической ошибки, в Perl выполняются все
блоки END, которые были определены. Никакие другие действия в настоящее время не выпол-
няются. В частности, буферы файлов автоматически не сбрасываются и объекты автоматиче-
ски не уничтожаются.
1209