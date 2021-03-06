---
layout: post
title: Глава 60. Генетический оптимизатор запросов
description: ""
tags: [PostgreSQL, PostgreSQL_Book_11]
image:
  feature: abstract-11.jpg
  #credit: dargadgetz
  #creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
share: true
modified: 2018-12-03 T15:14:43-04:00
---

Глава 60. Генетический оптимизатор запросов


Автор
Разработал Мартин Утеш (<utesch@aut.tu-freiberg.de>) для Института автоматиче-
ского управления в Техническом университете Фрайбергская горная академия, Герма-
ния.
60.1. Обработка запроса как сложная задача оптими-
зации
Среди всех реляционных операторов самым сложным для обработки и оптимизации является со-
единение. В первую очередь потому, что по мере увеличения числа соединений в запросе число
возможных планов запроса увеличивается экспоненциально. Дополнительная сложность оптими-
зации связана с наличием различных методов соединения (например, в PostgreSQL это вложен-
ный цикл, соединение по хешу и соединение слиянием) для каждого отдельного соединения и раз-
нообразием индексов (например, в PostgreSQL это B-дерево, хеш, GiST и GIN), определяющих путь
доступа к отношениям.
Традиционный оптимизатор запросов PostgreSQL выполняет почти исчерпывающий поиск во всём
множестве возможных стратегий. Этот алгоритм, появившийся в СУБД IBM System R, находит
порядок соединений, близкий к оптимальному, но может требовать огромного количества време-
ни и памяти, когда число соединений оказывается большим. В результате обычный оптимизатор
PostgreSQL оказывается неподходящим для запросов, в которых соединяется большое количество
таблиц.
Институт автоматического управления в Техническом университете Фрайбергская горная акаде-
мия, Германия, столкнулся с этими проблемами, разрабатывая систему принятия решений на ос-
нове базы знаний для обслуживания электростанций, в которой в качестве СУБД планировалось
применять PostgreSQL. Для машины, делающей выводы на основе базы знаний, СУБД должна была
выполнять запросы с таким количеством соединений, что использование обычного оптимизатора
запросов оказалось неприемлемым.
Далее мы опишем реализацию генетического алгоритма, который решает проблему выбора по-
рядка соединений эффективным способом для запросов с большим числом соединений.
60.2. Генетические алгоритмы
Генетический алгоритм (ГА) реализует метод эвристической оптимизации, построенный на слу-
чайном поиске. В данном контексте множество возможных решений проблемы оптимизации на-
зывается популяцией особей. Степень адаптации особи к среде определяет функция приспособ-
ленности.
Координаты особи в пространстве поиска представляются хромосомами, которые по сути являют-
ся символьными строками. Фрагмент хромосомы, кодирующий значение одного оптимизируемого
параметра, называется геном. Обычно ген кодируется в виде двоичного или целочисленного зна-
чения.
В результате симуляции эволюционных операций (скрещивания, мутации и селекции) данный ал-
горитм формирует новые поколения особей, у которых приспособленность в среднем будет выше,
чем у их предшественников.
Как сказано в ответах на вопросы в группе comp.ai.genetic, нельзя не отметить, что ГА реализует
не чисто случайный поиск решения проблемы. В ГА происходят вероятностные процессы, но ре-
зультат явно оказывается не случайным (лучше случайного).
2115Генетический опти-
мизатор запросов
Рисунок 60.1. Диаграмма структуры генетического алгоритма
P(t) поколение предков на момент t
P''(t) поколение потомков на момент t
+=========================================+
|>>>>>>>>>>> Алгоритм ГА <<<<<<<<<<<<<<|
+=========================================+
| ИНИЦИАЛИЗАЦИЯ t := 0
|
+=========================================+
| ИНИЦИАЛИЗАЦИЯ P(t)
|
+=========================================+
| вычислить ПРИСПОСОБЛЕННОСТЬ P(t)
|
+=========================================+
| пока не выполняется УСЛОВИЕ ОСТАНОВКИ
|
|
+-------------------------------------+
|
| P'(t) := СКРЕЩИВАНИЕ{P(t)}
|
|
+-------------------------------------+
|
| P''(t) := МУТАЦИЯ{P'(t)}
|
|
+-------------------------------------+
|
| P(t+1) := СЕЛЕКЦИЯ{P''(t) + P(t)}
|
|
+-------------------------------------+
|
| вычислить ПРИСПОСОБЛЕННОСТЬ P''(t) |
|
+-------------------------------------+
|
| t := t + 1
|
+===+=====================================+
60.3. Генетическая оптимизация запросов (GEQO) в
PostgreSQL
Модуль GEQO (Genetic Query Optimization, Генетическая оптимизация запросов) подходит к
проблеме оптимизации запроса как к хорошо известной задаче коммивояжёра (TSP, Traveling
Salesman Problem). Возможные планы запроса кодируются числами в строковом виде. Каждая
строка представляет порядок соединения одного отношения из запроса со следующим. Например,
дерево соединения
/\
/\ 2
/\ 3
4 1
кодируется строкой целых чисел '4-1-3-2', которая означает: сначала соединить отношения '4' и
'1', потом добавить '3', а затем '2', где 1, 2, 3, 4 — идентификаторы отношений внутри оптимизатора
PostgreSQL.
Реализация GEQO в PostgreSQL имеет следующие особые характеристики:
• Использование ГА с зафиксированным состоянием (когда заменяются наименее приспособ-
ленные особи популяции, а не всё поколение) способствует быстрой сходимости к улучшен-
ным планам запроса. Это важно для обработки запроса за приемлемое время;
• Использование скрещивания с обменом рёбер, которое очень удачно минимизирует число по-
терянных рёбер при решении задачи коммивояжёра с применением ГА;
• Мутация как генетический оператор считается устаревшей, так что для получения допусти-
мых путей TSP не требуются механизмы исправления.
Части модуля GEQO взяты из алгоритма Genitor, разработанного Д. Уитли.
В результате, модуль GEQO позволяет оптимизатору запросов PostgreSQL эффективно выполнять
запросы со множеством соединений, обходясь без полного перебора вариантов.
2116Генетический опти-
мизатор запросов
60.3.1. Построение возможных планов с GEQO
В процедуре планирования в GEQO используется код стандартного планировщика, который строит
планы сканирования отдельных отношений. Затем вырабатываются планы соединений с примене-
нием генетического подхода. Как было сказано выше, каждый план соединения представляется по-
следовательностью чисел, определяющей порядок соединений базовых отношений. На начальной
стадии код GEQO просто случайным образом генерирует несколько возможных последовательно-
стей. Затем для каждой рассматриваемой последовательности вызывается функция стандартного
планировщика, оценивающая стоимость запроса в случае выбора этого порядка соединений. (Для
каждого шага последовательности рассматриваются все три возможные стратегии соединения и
все изначально выбранные планы сканирования отношений. Результирующей оценкой стоимости
будет минимальная из всех возможных.) Последовательности соединений с наименьшей оценкой
стоимости считаются «более приспособленными», чем последовательности с большей оценкой.
Проанализировав возможные последовательности, генетический алгоритм отбрасывает наименее
приспособленные из них. Затем генерируются новые кандидаты путём объединения генов более
приспособленных последовательностей — для этого выбираются случайные фрагменты известных
последовательностей с низкой стоимостью, из которых складываются новые последовательности
для рассмотрения. Этот процесс повторяется, пока не будет рассмотрено некоторое предопреде-
лённое количество последовательностей соединений; после этого для построения окончательного
плана выбирается лучшая последовательность, найденная за всё время поиска.
Этот процесс по природе своей недетерминирован, вследствие случайного выбора при форми-
ровании начальной популяции и последующей «мутации» лучших кандидатов. Но во избежание
неожиданных изменений выбранного плана, на каждом проходе алгоритм GEQO перезапускает
свой генератор случайных чисел с текущим значением параметра geqo_seed. Поэтому пока зна-
чение geqo_seed и другие параметры GEQO остаются неизменными, для определённого запроса
(и других входных данных планировщика, в частности, статистики) будет строиться один и тот же
план. Если вы хотите поэкспериментировать с разными путями соединений, попробуйте изменить
geqo_seed.
60.3.2. Будущее развитие модуля PostgreSQL GEQO
Требуется провести дополнительную работу для выбора оптимальных параметров генетического
алгоритма. В файле src/backend/optimizer/geqo/geqo_main.c, подпрограммах gimme_pool_size
и gimme_number_generations, мы должны найти компромиссные значения параметров, удовлетво-
ряющие двум несовместимым требованиям:
• Оптимальность плана запроса
• Время вычисления
В текущей реализации приспособленность каждой рассматриваемой последовательности соеди-
нений рассчитывается стандартным планировщиком, который каждый раз вычисляет избиратель-
ность соединения и стоимость заново. С учётом того, что различные кандидаты могут содержать
общие подпоследовательности соединений, при этом будет повторяться большой объём работы.
Таким образом, расчёт можно значительно ускорить, сохраняя оценки стоимости для внутренних
соединений, но сложность состоит в том, чтобы уместить это состояние в разумные объёмы памяти.
На более общем уровне не вполне понятно, насколько уместно для оптимизации запросов исполь-
зовать ГА, предназначенный для решения задачи коммивояжёра. В этой задаче стоимость, свя-
занная с любой подстрокой (частью тура) не зависит от остального маршрута, но это определённо
не так для оптимизации запросов. Таким образом, возникает вопрос, насколько эффективно скре-
щивание путём обмена рёбрами.
60.4. Дополнительные источники информации
Дополнительную информацию о генетических алгоритмах можно получить в следующих источни-
ках:
• The Hitch-Hiker's Guide to Evolutionary Computation, (Руководство для путешествующих авто-
стопом по эволюционным вычислениям, Ответы на часто задаваемые вопросы в группе news://
comp.ai.genetic)
2117Генетический опти-
мизатор запросов
• Evolutionary Computation and its application to art and design (Эволюционные вычисления и их
применение в искусстве и дизайне), Крейг Рейнольдс
• elma04
• fong
2118
