---
layout: post
title: Глава 66. Индексы GIN
description: ""
tags: [PostgreSQL, PostgreSQL_Book_11]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 66. Индексы GIN

66.1. Введение

GIN расшифровывается как «Generalized Inverted Index» (Обобщённый инвертированный индекс).
GIN предназначается для случаев, когда индексируемые значения являются составными, а запро-
сы, на обработку которых рассчитан индекс, ищут значения элементов в этих составных объектах.
Например, такими объектами могут быть документы, а запросы могут выполнять поиск докумен-
тов, содержащих определённые слова.
Здесь мы используем термин объект, говоря о составном значении, которое индексируется, и тер-
мин ключ, говоря о включённом в него элементе. GIN всегда хранит и ищет ключи, а не объекты
как таковые.
Индекс GIN сохраняет набор пар (ключ, список идентификаторов), где список идентификаторов
содержит идентификаторы строк, в которых находится ключ. Один и тот же идентификатор строки
может фигурировать в нескольких списках, так как объект может содержать больше одного ключа.
Значение каждого ключа хранится только один раз, так что индекс GIN очень компактен в случаях,
когда один ключ встречается много раз.
GIN является обобщённым в том смысле, что код метода доступа GIN не должен знать о конкрет-
ных операциях, которые он ускоряет. Вместо этого задаются специальные стратегии для конкрет-
ных типов данных. Стратегия определяет, как извлекаются ключи из индексируемых объектов и
условий запросов, и как установить, действительно ли удовлетворяет запросу строка, содержащая
некоторые значения ключей.
Ключевым преимуществом GIN является то, что он позволяет разрабатывать дополнительные ти-
пы данных с соответствующими методами доступа экспертам в предметной области типа данных,
а не специалистам по СУБД. В этом аспекте он похож на GiST.
Сопровождением реализации GIN в PostgreSQL в основном занимаются Фёдор Сигаев и Олег Бар-
тунов. Дополнительные сведения о GIN можно получить на их сайте.
66.2. Встроенные классы операторов
В базовый дистрибутив PostgreSQL включены классы операторов GIN, перечисленные в Табли-
це 66.1. (Некоторые дополнительные модули, описанные в Приложении F, добавляют другие клас-
сы операторов GIN.)
Таблица 66.1. Встроенные классы операторов GIN
Имя Индексируемый тип данных Индексируемые операторы
array_ops anyarray
&& <@ = @>
jsonb_ops jsonb
? ?& ?| @>
jsonb_path_ops jsonb
@>
tsvector_ops tsvector
@@ @@@
Из двух классов операторов для типа jsonb классом по умолчанию является jsonb_ops. Класс
jsonb_path_ops поддерживает меньше операторов, но обеспечивает для них большую производи-
тельность. За подробностями обратитесь к Подразделу 8.14.4.
66.3. Расширяемость
Интерфейс GIN характеризуется высоким уровнем абстракции и таким образом требует от разра-
ботчика метода доступа реализовать только смысловое наполнение обрабатываемого типа данных.
2160Индексы GIN
Уровень GIN берёт на себя заботу о параллельном доступе, поддержке журнала и поиске в струк-
туре дерева.
Всё, что нужно, чтобы получить работающий метод доступа GIN — это реализовать несколько
пользовательских методов, определяющих поведение ключей в дереве и отношения между ключа-
ми, индексируемыми объектами и поддерживаемыми запросами. Словом, GIN сочетает расширя-
емость с универсальностью, повторным использованием кода и аккуратным интерфейсом.
Класс операторов должен предоставить GIN следующие два метода:
Datum *extractValue(Datum itemValue, int32 *nkeys, bool **nullFlags)
Возвращает массив ключей (выделенный через palloc) для индексируемого объекта. Число воз-
вращаемых ключей должно записываться в *nkeys. Если какой-либо из ключей может быть
NULL, нужно так же выделить через palloc массив из *nkeys полей bool, записать его адрес
в *nullFlags и установить эти флаги NULL как требуется. В *nullFlags можно оставить зна-
чение NULL (это начальное значение), если все ключи отличны от NULL. Эта функция может
возвратить NULL, если объект не содержит ключей.
Datum *extractQuery(Datum query, int32 *nkeys, StrategyNumber n, bool **pmatch, Pointer
**extra_data, bool **nullFlags, int32 *searchMode)
Возвращает массив ключей (выделенный через palloc) для запрашиваемого значения; то есть,
в query поступает значение, находящееся по правую сторону индексируемого оператора, по
левую сторону которого указан индексируемый столбец. Аргумент n задаёт номер стратегии
оператора в классе операторов (см. Подраздел 38.15.2). Часто функция extractQuery должна
проанализировать n, чтобы определить тип данных аргумента query и выбрать метод для из-
влечения значений ключей. Число возвращаемых ключей должно быть записано в *nkeys. Ес-
ли какие-либо ключи могут быть NULL, нужно так же выделить через palloc массив из *nkeys
полей bool, сохранить его адрес в *nullFlags, и установить эти флаги NULL как требуется. В
*nullFlags можно оставить значение NULL (это начальное значение), если все ключи отличны
от NULL. Эта функция может возвратить NULL, если query не содержит ключей.
Выходной аргумент searchMode позволяет функции extractQuery выбрать режим, в кото-
ром должен выполняться поиск. Если *searchMode имеет значение GIN_SEARCH_MODE_DEFAULT
(это значение устанавливается перед вызовом), подходящими кандидатами считаются толь-
ко те объекты, которые соответствуют минимум одному из возвращённых ключей. Если в
*searchMode установлено значение GIN_SEARCH_MODE_INCLUDE_EMPTY, то в дополнение к объ-
ектам с минимум одним совпадением ключа, подходящими кандидатами будут считаться
и объекты, вообще не содержащие ключей. (Этот режим полезен для реализации, напри-
мер, операторов A-является-подмножеством-B.) Если в *searchMode установлено значение
GIN_SEARCH_MODE_ALL, подходящими кандидатами считаются все отличные от NULL объекты в
индексе, независимо от того, встречаются ли в них возвращаемые ключи. (Этот режим намного
медленнее двух других, так как он по сути требует сканирования всего индекса, но он может
быть необходим для корректной обработки крайних случаев. Оператор, который выбирает этот
режим в большинстве ситуаций, скорее всего не подходит для реализации в классе операторов
GIN.) Символы для этих значений режима определены в access/gin.h.
Выходной аргумент pmatch используется, когда поддерживается частичное соответствие. Что-
бы использовать его, extractQuery должна выделить массив из *nkeys булевских элементов и
сохранить его адрес в *pmatch. Элемент этого массива должен содержать true, если соответ-
ствующий ключ требует частичного соответствия, и false в противном случае. Если переменная
*pmatch содержит NULL, GIN полагает, что частичное соответствие не требуется. В эту пере-
менную записывается NULL перед вызовом, так что этот аргумент можно просто игнорировать
в классах операторов, не поддерживающих частичное соответствие.
Выходной аргумент extra_data позволяет функции extractQuery передать дополнительные
данные методам consistent и comparePartial. Чтобы использовать его, extractQuery должна
выделить массив из *nkeys указателей и сохранить его адрес в *extra_data, а затем сохранить
всё, что ей требуется, в отдельных указателях. В эту переменную записывается NULL перед вы-
2161Индексы GIN
зовом, поэтому данный аргумент может просто игнорироваться классами операторов, которым
не нужны дополнительные данные. Если массив *extra_data задан, он целиком передаётся в
метод consistent, а в comparePartial передаётся соответствующий его элемент.
Класс операторов должен также предоставить функцию для проверки, соответствует ли индекси-
рованный объект запросу. Поддерживаются две её вариации: булевская consistent и троичная
triConsistent. Функция triConsistent покрывает функциональность обоих, так что достаточно
реализовать только её. Однако, если вычисление булевской вариации оказывается значительно
дешевле, может иметь смысл реализовать их обе. Если представлена только булевская вариация,
некоторые оптимизации, построенные на отбраковывании объектов до выборки всех ключей, от-
ключаются.
bool consistent(bool check[], StrategyNumber n, Datum query, int32
extra_data[], bool *recheck, Datum queryKeys[], bool nullFlags[])
nkeys,
Pointer
Возвращает true, если индексированный объект удовлетворяет оператору запроса с номером
стратегии n (или потенциально удовлетворяет, когда возвращается указание перепроверки).
Эта функция не имеет прямого доступа к значению индексированного объекта, так как GIN
не хранит сами объекты. Вместо этого, она знает о значениях ключей, извлечённых из запро-
са и встречающихся в данном индексированном объекте. Массив check имеет длину nkeys,
что равняется числу ключей, ранее возвращённых функцией extractQuery для данного зна-
чения query. Элемент массива check равняется true, если индексированный объект содержит
соответствующий ключ запроса; то есть, если (check[i] == true), то i-ый ключ в массиве ре-
зультата extractQuery присутствует в индексированном объекте. Исходное значение query
передаётся на случай, если оно потребуется методу consistent; с той же целью ему пере-
даются массивы queryKeys[] и nullFlags[], ранее возвращённые функцией extractQuery. В
аргументе extra_data передаётся массив дополнительных данных, возвращённый функцией
extractQuery, или NULL, если дополнительных данных нет.
Когда extractQuery возвращает ключ NULL в queryKeys[], соответствующий элемент check[]
содержит true, если индексированный объект содержит ключ NULL; то есть можно считать, что
элементы check[] отражают условие IS NOT DISTINCT FROM. Функция consistent может прове-
рить соответствующий элемент nullFlags[], если ей нужно различать соответствие с обычным
значением и соответствие с NULL.
В случае успеха в *recheck нужно записать true, если кортеж данных нужно перепроверить
с оператором запроса, либо false, если проверка по индексу была точной. То есть результат
false гарантирует, что кортеж данных не соответствует запросу; результат true со значением
*recheck, равным false, гарантирует, что кортеж данных соответствует запросу; а результат
true со значением *recheck, равным true, означает, что кортеж данных может соответствовать
запросу, поэтому его нужно выбрать и перепроверить, применив оператор запроса непосред-
ственно к исходному индексированному элементу.
GinTernaryValue triConsistent(GinTernaryValue check[], StrategyNumber n, Datum query,
int32 nkeys, Pointer extra_data[], Datum queryKeys[], bool nullFlags[])
Функция triConsistent подобна consistent, но вместо булевских значений в векторе check
ей передаются три варианта сравнений для каждого ключа: GIN_TRUE, GIN_FALSE и GIN_MAYBE.
GIN_FALSE и GIN_TRUE имеют обычное булевское значение, тогда как GIN_MAYBE означает, что
присутствие ключа неизвестно. Когда присутствуют значения GIN_MAYBE, функция должна воз-
вращать GIN_TRUE, только если объект удовлетворяет запросу независимо от того, содержит
ли индекс соответствующие ключи запроса. Подобным образом, функция должна возвращать
GIN_FALSE, только если объект не удовлетворяет запросу независимо от того, содержит ли он
ключи GIN_MAYBE. Если результат зависит от элементов GIN_MAYBE, то есть соответствие нельзя
утверждать или отрицать в зависимости от известных ключей запроса, функция должна вер-
нуть GIN_MAYBE.
Когда в векторе check нет элементов GIN_MAYBE, возвращаемое значение GIN_MAYBE равнознач-
но установленному флагу recheck в булевской функции consistent.
2162Индексы GIN
Кроме того, GIN нужно каким-то образом сортировать значения ключа, хранящиеся в индексе.
Класс операторов может определить порядок сортировки, указав метод сравнения:
int compare(Datum a, Datum b)
Сравнивает два ключа (не индексированные объекты!) и возвращает целое меньше нуля, ноль
или целое больше нуля, показывающее, что первый ключ меньше, равен или больше второго.
Ключи NULL никогда не передаются этой функции.
Если же класс операторов не определяет метод compare, GIN попытается найти класс операторов
B-дерева по умолчанию для типа данных ключа индекса и воспользоваться его функцией срав-
нения. Если класс операторов GIN предназначен только для одного типа данных, рекомендует-
ся задавать функцию сравнения в этом классе операторов, так как поиск класса операторов B-
дерева занимает несколько циклов. Однако для полиморфных классов операторов GIN (например,
array_ops) задать одну функцию сравнения обычно не представляется возможным.
Дополнительно класс операторов для GIN может предоставить следующий метод:
int comparePartial(Datum partial_key, Datum key, StrategyNumber n, Pointer extra_data)
Сравнивает ключ запроса с частичным соответствием с ключом индекса. Возвращает целое
число, знак которого отражает результат сравнения: отрицательное число означает, что ключ
индекса не соответствует запросу, но нужно продолжать сканирование индекса; ноль означа-
ет, что ключ индекса соответствует запросу; положительное число означает, что сканирова-
ние индекса нужно прекратить, так как других соответствий не будет. Функции передаётся
номер стратегии n оператора, сформировавшего запрос частичного соответствия, на случай,
если нужно знать его смысл, чтобы определить, когда прекращать сканирование. Кроме того,
ей передаётся extra_data — соответствующий элемент массива дополнительных данных, сфор-
мированного функцией extractQuery, либо NULL, если дополнительных данных нет. Значения
NULL этой функции никогда не передаются.
Для поддержки проверок на «частичное соответствие» класс операторов должен предоставить ме-
тод comparePartial, а метод extractQuery должен устанавливать параметр pmatch, когда встреча-
ется запрос на частичное соответствие. За подробностями обратитесь к Подразделу 66.4.2.
Фактические типы данных различных значений Datum, упоминаемых выше, зависят от класса опе-
раторов. Значения объектов, передаваемые в extractValue, всегда имеют входной тип класса опе-
раторов, а все значения ключей должны быть типа, заданного параметром STORAGE для класса.
Типом аргумента query, передаваемого функциям extractQuery, consistent и triConsistent, бу-
дет тот тип, что указан в качестве типа правого операнда оператора-члена класса, определяемого
по номеру стратегии. Это не обязательно должен быть индексируемый тип, достаточно лишь, что-
бы из него можно было извлечь значения ключей, имеющие нужный тип. Однако рекомендуется,
чтобы в SQL-объявлениях этих трёх опорных функций для аргумента query назначался индекси-
руемый тип класса операторов, даже несмотря на то, что фактический тип может быть другим, в
зависимости от оператора.
66.4. Реализация
Внутри индекс GIN содержит B-дерево, построенное по ключам, где каждый ключ является эле-
ментом одного или нескольких индексированных объектов (например, членом массива) и где каж-
дый кортеж на страницах листьев содержит либо указатель на B-дерево указателей на данные
(«дерево идентификаторов»), либо простой список указателей на данные («список идентификато-
ров»), когда этот список достаточно мал, чтобы уместиться в одном кортеже индекса вместе со
значением ключа.
Начиная с PostgreSQL версии 9.1, в индекс могут быть включены значения ключей, равные NULL.
Кроме того, в индекс вставляются NULL для индексируемых объектов, равных NULL или не содер-
жащих ключей, согласно функции extractValue. Это позволяет находить при поиске пустые объ-
екты, когда они должны быть найдены.
Составные индексы GIN реализуются в виде одного B-дерева по составным значениям (номер
столбца, значение ключа). Значения ключей для различных столбцов могут быть разных типов.
2163Индексы GIN
66.4.1. Методика быстрого обновления GIN
Природа инвертированного индекса такова, что обновление GIN обычно медленная операция: при
добавлении или изменении одной строки данных может потребоваться выполнить множество до-
бавлений записей в индекс (для каждого ключа, извлечённого из индексируемого объекта). На-
чиная с PostgreSQL 8.4, GIN может отложить большой объём этой работы, вставляя новые корте-
жи во временный, несортированный список записей, ожидающих индексации. Когда таблица очи-
щается, автоматически анализируется, вызывается функция gin_clean_pending_list или размер
этого списка временного списка становится больше чем gin_pending_list_limit, записи переносятся
в основную структуру данных GIN теми же методами массового добавления данных, что и при на-
чальном создании индекса. Это значительно увеличивает скорость обновления индекса GIN, даже
с учётом дополнительных издержек при очистке. К тому же эту дополнительную работу можно
выполнить в фоновом процессе, а не в процессе, непосредственно выполняющем запросы.
Основной недостаток такого подхода состоит в том, что при поиске необходимо не только прове-
рить обычный индекс, но и просканировать список ожидающих записей, так что если этот список
большой, поиск значительно замедляется. Ещё один недостаток состоит в том, что хотя в основ-
ном изменения производятся быстро, изменение, при котором этот список оказывается «слишком
большим», влечёт необходимость немедленной очистки и поэтому выполняется гораздо дольше
остальных изменений. Минимизировать эти недостатки можно, правильно организовав автоочист-
ку.
Если выдержанность времени операций важнее скорости обновления, применение списка ожида-
ющих записей можно отключить, выключив параметр хранения fastupdate для индекса GIN. За
подробностями обратитесь к CREATE INDEX.
66.4.2. Алгоритм частичного соответствия
GIN может поддерживать проверки «частичного соответствия», когда запрос выявляет не точное
соответствие одному или нескольким ключам, а возможные соответствия, попадающие в доста-
точно узкий диапазон значений ключей (при порядке сортировки ключей, определённым опор-
ным методом compare). В этом случае метод extractQuery возвращает не значение ключа, которое
должно соответствовать точно, а значение, определяющее нижнюю границу искомого диапазона,
и устанавливает флаг pmatch. Затем диапазон ключей сканируется методом comparePartial. Ме-
тод comparePartial должен вернуть ноль при соответствии ключа индекса, отрицательное значе-
ние, если соответствия нет, но нужно продолжать проверку диапазона, и положительное значе-
ние, если ключ индекса оказался за искомым диапазоном.
66.5. Приёмы и советы по применению GIN
Создание или добавление
Добавление объектов в индекс GIN может выполняться медленно, так как для каждого объек-
та скорее всего потребуется добавлять множество ключей. Поэтому при массовом добавлении
данных в таблицу рекомендуется удалить индекс GIN и пересоздать его по окончании добав-
ления.
Начиная с PostgreSQL 8.4, этот совет менее актуален, так как выполнение индексации может
быть отложенным (подробнее об этом в Подразделе 66.4.1). Но при очень большом объёме из-
менений может быть лучше всё-таки удалить и пересоздать индекс.
maintenance_work_mem
Время построения индекса GIN очень сильно зависит от параметра maintenance_work_mem; не
стоит экономить на рабочей памяти при создании индекса.
gin_pending_list_limit
В процессе последовательных добавлений в существующий индекс GIN с включённым режи-
мом fastupdate, система будет очищать список ожидающих индексации записей, когда его
размер будет превышать gin_pending_list_limit. Во избежание значительных колебаний ко-
2164Индексы GIN
нечного времени ответа имеет смысл проводить очистку этого списка в фоновом режиме (то
есть, применяя автоочистку). Избежать операций очистки на переднем плане можно, увели-
чив gin_pending_list_limit или проводя автоочистку более активно. Однако, если вследствие
увеличения порога операции очистки запустится очистка на переднем плане, она будет выпол-
няться ещё дольше.
Значение gin_pending_list_limit можно переопределить для отдельных индексов GIN, изме-
нив их параметры хранения, что позволяет задавать для каждого индекса GIN свой порог очист-
ки. Например, можно увеличить порог только для часто обновляемых индексов GIN и оставить
его низким для остальных.
gin_fuzzy_search_limit
Основной целью разработки индексов GIN было обеспечить поддержку хорошо расширяемого
полнотекстового поиска в PostgreSQL, а при полнотекстовом поиске нередко возникают ситуа-
ции, когда возвращается очень большой набор результатов. Однако чаще всего так происходит,
когда запрос содержит очень часто встречающиеся слова, так что полученный результат всё
равно оказывается бесполезным. Так как чтение множества записей с диска и последующая
сортировка их может занять много времени, это неприемлемо в производственных условиях.
(Заметьте, что поиск по индексу при этом выполняется очень быстро.)
Для управляемого выполнения таких запросов в GIN введено настраиваемое мяг-
кое ограничение сверху для числа возвращаемых строк: конфигурационный параметр
gin_fuzzy_search_limit. По умолчанию он равен 0 (то есть ограничение отсутствует). Если он
имеет ненулевое значение, возвращаемый набор строк будет случайно выбранным подмноже-
ством всего набора результатов.
«Мягким» оно называется потому, что фактическое число возвращаемых строк может несколь-
ко отличаться от заданного значения, в зависимости от запроса и качества системного гене-
ратора случайных чисел.
Из практики, со значениями в несколько тысяч (например, 5000 — 20000) получаются прием-
лемые результаты.
66.6. Ограничения
GIN полагает, что индексируемые операторы являются строгими. Это означает, что функция
extractValue вовсе не будет вызываться для значений NULL (вместо этого будет автоматически
создаваться пустая запись в индексе), так же как и extractQuery не будет вызываться с искомым
значением NULL (при этом сразу предполагается, что запрос не удовлетворяется). Заметьте, од-
нако, что при этом поддерживаются ключи NULL, содержащиеся в составных объектах или иско-
мых значениях.
66.7. Примеры
В базовый дистрибутив PostgreSQL включены классы операторов GIN, перечисленные ранее в Таб-
лице 66.1. Также классы операторов GIN содержатся в следующих модулях contrib:
btree_gin
Функциональность B-дерева для различных типов данных
hstore
Модуль для хранения пар (ключ, значение)
intarray
Расширенная поддержка int[]
pg_trgm
Схожесть текста на основе статистики триграмм
2165