---
title: "Почему PostgreSQL ошибается на correlated predicates и уходит в Nested Loop"
date: 2026-04-22T00:00:00+02:00
draft: true
description: "How Column Independence Assumption underestimates row counts on correlated business data and pushes PostgreSQL from Hash Join into CPU-expensive Nested Loop."
tags:
  - postgresql
  - query-planner
  - cardinality-estimation
  - sql-optimization
  - performance
  - database-internals
categories:
  - Databases
  - Backend Engineering
icon: "/images/icons/squares-accent.svg"
showToc: true
canonicalURL: "" 

---

TL;DR: Статья разбирает сценарий, в котором бизнес-корреляция колонок нарушает математику планировщика PostgreSQL. Из-за ошибки в оценке кардинальности база направляет миллионы строк в Nested Loop, а наличие индекса лишь создает иллюзию эффективности, еще сильнее убеждая оптимизатор выбрать неверный план.
Код для эмуляции и локального воспроизведения проблемы доступен в [GitHub-репозитории](https://github.com/nikita-kibitkin/postgres-mcv-demo).


В математической модели PostgreSQL колонки по умолчанию считаются независимыми. На практике бизнес-данные часто содержат кросс-колоночную корреляцию. Здесь статистика расходится с реальностью: cardinality estimate занижается на уровне базовой таблицы, и эта недооценка распространяется вверх по дереву (error propagation).

В результате база может выбрать Nested Loop там, где фактический объем данных уже требует Hash Join. При этом наличие подходящего индекса может даже усугубить ситуацию, приводя к избыточному случайному чтению и эффекту CPU-amplification.

Ниже разберем один конкретный сценарий. Он специально упрощен для лёгкости понимания и несколько гиперболизирован для воспроизведимости на малых масштабах синтетического эксперимента, но сам механизм типичен для production.

## Problem

Есть две таблицы по 10 млн строк.

```sql
CREATE TABLE transactions (
    id bigserial primary key,
    transfer_type text not null,
    route_type text not null
);

CREATE INDEX idx_transactions_type_route
    ON transactions (transfer_type, route_type);

CREATE TABLE compliance_docs (
    transaction_id bigint not null,
    notes text not null
);

CREATE UNIQUE INDEX idx_compliance_docs_txid
    ON compliance_docs (transaction_id);
```
Таблица transactions здесь является base relation с двумя статически зависимыми predicates. Мы делаем 
JOIN transactions с compliance_docs.
Для простоты предполагается идеальный `1:1` между `transactions` и `compliance_docs`: одна строка в `compliance_docs` на один `transaction_id`. Я намеренно избегаю fan-out join для чистоты эксперимента.

Бизнес-правило такое:

- 10% всех переводов имеют `transfer_type = 'SWIFT'`;
- в целом по таблице 10% строк имеют `route_type = 'CROSS_BORDER'`;
- все переводы `SWIFT` идут по `route_type = 'CROSS_BORDER'`.

В реальной платежной системе не любой `CROSS_BORDER` перевод означает `SWIFT`. Есть `SEPA`, локальные gateway-маршруты и другие каналы. Но в этом synthetic slice доли специально сделаны равными, чтобы математика ошибки оставалась прозрачной: planner видит `10% × 10% = 1%`, тогда как реальное пересечение остается `10%`.

То есть в данных есть сильная корреляция:

```text
P(route_type = 'CROSS_BORDER' | transfer_type = 'SWIFT') = 1
```

Запрос:

```sql
SELECT t.id, d.notes
FROM transactions t
JOIN compliance_docs d
  ON t.id = d.transaction_id
WHERE t.transfer_type = 'SWIFT'
  AND t.route_type = 'CROSS_BORDER';
```

## Act 1. Problem Physics: planner math

Planner PostgreSQL по умолчанию исходит из **Column Independence Assumption**. Если в `WHERE` есть два predicates по двум разным колонкам одной таблицы, он оценивает их совместную селективность как произведение отдельных вероятностей:

```text
P(A ∩ B) = P(A) × P(B)
```

Для planner'а в нашем примере:

```text
A := transfer_type = 'SWIFT'
B := route_type = 'CROSS_BORDER'
```

Если по обычной per-column statistics видно, что:

```text
P(A) = 0.1
P(B) = 0.1
```

то estimate будет таким:

```text
P(A ∩ B) = 0.1 × 0.1 = 0.01
```

На таблице в 10 млн строк это дает:

```text
E-Rows = 10,000,000 × 0.01 = 100,000
```

Но реальная бизнес-семантика другая. Для `SWIFT` условие по типу маршрута не просто "часто совпадает". Оно функционально следует из первого условия:

```text
P(B | A) = 1
P(A ∩ B) = P(A) × P(B | A) = 0.1 × 1 = 0.1
```

Значит фактическая кардинальность такая:

```text
A-Rows = 10,000,000 × 0.1 = 1,000,000
```

Ошибка planner'а в этом toy example составляет `10x`. В production она легко вырастает до `100x+`, если одно условие редкое, второе тоже редкое, а вместе они почти детерминированы.

Это неприятно уже на двух predicates. Но с ростом числа зависимых predicates ошибка растет экспоненциально. Каждый новый `AND`, который planner считает независимым от предыдущих условий, еще раз увеличивает расхождение estimate. Если две скоррелированные колонки дают ошибку порядка `10x`, то три колонки при той же логике уже легко дают `100x`, а четыре — `1000x`. Поэтому самые опасные запросы в таких схемах — не короткие OLTP-фильтры, а длинные compliance или analytics predicates, где planner с каждым дополнительным условием увеличивает расхождение предполагаемого размера множества base relation и реальности.

Псевдо-`EXPLAIN` на этом шаге выглядит так:

```text
Bitmap Heap Scan on transactions t
  Recheck Cond: ((transfer_type = 'SWIFT') AND (route_type = 'CROSS_BORDER'))
  ->  Bitmap Index Scan on idx_transactions_type_route
        Index Cond: ((transfer_type = 'SWIFT') AND (route_type = 'CROSS_BORDER'))
  rows=100000  -- E-Rows
```

А в реальности `EXPLAIN ANALYZE` покажет что-то концептуально такое:

```text
Bitmap Heap Scan on transactions t
  ...
  rows=100000 loops=1      -- estimate
  actual rows=1000000      -- reality
```

Этого уже достаточно, чтобы запустить error propagation и нарушить вычисление cost model.

## Act 2. Under-the-Hood Mechanics: why Nested Loop appears

Выбор между `Nested Loop` и `Hash Join` чувствителен к размеру outer side. В нашем случае outer side - это тот самый отфильтрованный base relation (transactions).

Если planner думает, что после фильтра останется примерно `100k` строк, картинка для него выглядит так:

1. Быстро достать outer rows из `transactions` по `(transfer_type, route_type)`.
2. Для каждой найденной строки сделать cheap index lookup в `compliance_docs(transaction_id)`.
3. Поскольку outer side "маленькая", цена `100k` B-Tree probes кажется приемлемой.

План на бумаге получается примерно такой:

```text
Nested Loop
  -> Bitmap Heap Scan on transactions t
       rows=100000
  -> Index Scan using idx_compliance_docs_txid on compliance_docs d
       Index Cond: (transaction_id = t.id)
```

Но outer side на самом деле не `100k`, а `1,000,000`. Ошибка в base relation estimate переносится дальше и превращает логичный `100k` nested lookup в `1M` отдельных index probes.

Важная оговорка: node на чтении `transactions` не обязан быть именно `Index Scan`. На 10% таблицы planner вполне может выбрать `Bitmap Index Scan + Bitmap Heap Scan`, а на очень широких строках или других `cost settings` даже `Seq Scan`. Проблема не в конкретном scan node. Проблема в том, что join stage получает на порядок больше outer rows, чем ожидалось.

### What "1 million index lookups" means physically

Это не одна большая линейная операция. Это миллион отдельных проходов по B-Tree:

1. descend от root к leaf;
2. поиск нужного `transaction_id`;
3. чтение соответствующего TID;
4. переход к heap tuple или index-only path, если повезло;
5. повторить это снова.

Если рабочее множество влезает в память, запрос часто становится **CPU-bound**: слишком много коротких, плохо предсказуемых операций, слишком много buffer pin/unpin, слишком много `LWLock` и latch activity вокруг горячих страниц и metadata в `shared_buffers`. Если рабочее множество не влезает в память, поверх этого добавляется random I/O.

Именно поэтому "индексный" план может оказаться хуже последовательного чтения на порядки. Планы с линейным профилем (`Seq Scan` + `Hash Join`) платят за большую линейную работу один раз. `Nested Loop` — немного, но миллион раз.

Псевдо-`EXPLAIN ANALYZE` для плохого случая:

```text
Nested Loop  (cost=...)
  rows=100000
  actual rows=1000000
  -> Bitmap Heap Scan on transactions t
       rows=100000
       actual rows=1000000
  -> Index Scan using idx_compliance_docs_txid on compliance_docs d
       rows=1
       loops=1000000
```

Обратите внимание на `loops=1000000`.

Если бы planner заранее видел реальный размер outer side, экономика плана была бы другой:

- один `Hash Join`;
- одно последовательное чтение большой стороны;
- одна фаза build/probe вместо миллиона B-Tree traversals.

Но неправильная оценка кардинальности привела к выбору неоптимального join algorithm.

## Act 3. False Engineering Intuitions

### Myth 1. "Just add an index"

В этом сценарии индексы уже есть:

- композитный индекс на `transactions(transfer_type, route_type)`;
- индекс на `compliance_docs(transaction_id)`.

Более того, именно наличие этих индексов математически оправдывает выбор `Nested Loop`:

- Оценка (`E-Rows = 100k`): planner рассчитывает, что стоимость `100,000` точечных обращений (`Index Probes`) ниже, чем накладные расходы на выделение памяти и построение хэш-таблицы. За счет низкой базовой стоимости обхода индекса `Nested Loop` получает меньшую итоговую оценку cost.

- Выполнение (`A-Rows = 1M`): ошибка оценки в `10x` приводит к кратному росту итераций цикла. Движок делает `1,000,000` обходов `B-Tree` вместо `100,000`, что уже дороже `Hash Join` по ресурсам.

Таким образом, индекс не создает проблему, но его низкая расчетная стоимость маскирует математическую ошибку planner'а и приводит к выбору неэффективного плана выполнения.

### Myth 2. "We load-tested this, it looked fine"

Обычный synthetic dataset часто генерируется через что-то вроде:

```sql
transfer_type = CASE WHEN random() < 0.1 THEN 'SWIFT' ELSE 'SEPA' END
route_type = CASE WHEN random() < 0.1 THEN 'CROSS_BORDER' ELSE 'DOMESTIC' END
```

Такая генерация делает колонки статистически независимыми. То есть в тестах искусственно выполняется предположение planner'а:

```text
P(A ∩ B) ≈ P(A) × P(B)
```

Тогда `E-Rows` действительно будут близки к `A-Rows`, planner выберет "правильный" план, а тесты скажут, что все стабильно.

После выката в production появляется реальное бизнес-правило:

```text
IF transfer_type = 'SWIFT'
THEN route_type = 'CROSS_BORDER'
```

И planner начинает систематически недооценивать outer side. То есть нагрузочный стенд валидировал не реальную workload, а искусственную случайную модель данных.

Так такие проблемы и проходят в production: the test volume is realistic, but the real-world data correlations are not.

## Act 4. Solution: extended statistics via MCV

Исправление выглядит так:

```sql
CREATE STATISTICS tx_correlation (mcv)
ON transfer_type, route_type
FROM transactions;

ANALYZE transactions;
```

Что делает `mcv`?

Обычная column statistics хранит популярные значения отдельно по каждой колонке. Extended statistics типа **MCV (Most Common Values)** хранят компактный список наиболее частых комбинаций значений и их частот. Это не полная двумерная матрица по всем возможным парам. Это именно metadata о тех value pairs, которые оказались достаточно частыми, чтобы попасть в multivariate statistics.

В терминах нашего примера planner перестает смотреть только на:

```text
P(transfer_type = 'SWIFT')
P(route_type = 'CROSS_BORDER')
```

по отдельности и начинает знать, что сама пара

```text
('SWIFT', 'CROSS_BORDER') -> 0.1
```

встречается заметно чаще, чем предсказывает произведение marginal probabilities. То есть **Independence Assumption** снимается не вообще для всей таблицы, а для тех combinations, которые реально попали в MCV list.

После `ANALYZE` estimate меняется концептуально так:

```text
Bitmap Heap Scan on transactions t
  ...
  rows=1000000
```

Здесь важна одна точность: `MCV` не чинит `join selectivity` напрямую. Он чинит estimate базового отношения `transactions`, отменяя ложное допущение о независимости колонок внутри одной таблицы.

Дальше planner просто пересчитывает join economics. С `1,000,000` outer rows `Nested Loop` перестает выглядеть выгодным. При такой оценке `Hash Join` или другой план с более линейным профилем часто становится дешевле.

Здесь есть еще один практический caveat. `MCV` помогает только если нужная комбинация вообще попала в statistics object. В нашем toy example это почти гарантировано: пара `('SWIFT', 'CROSS_BORDER')` занимает 10% таблицы и достаточно "горячая". Но в реальных схемах при более слабой корреляции или большом числе конкурирующих hot combinations может понадобиться повышать `STATISTICS` target, иначе `ANALYZE` просто не сохранит достаточно подробную MCV list.


Важно правильно оценить область эффекта. Extended statistics в PostgreSQL не используются напрямую для **join selectivity estimation**. Они используются для более точной оценки **base relation predicates**. Но в данном сценарии этого достаточно, потому что именно ошибка в размере отфильтрованной `transactions` и ломала выбор join strategy.

## Trade-off and decision rule

`MCV` не надо навешивать на каждую пару колонок "на всякий случай".

У extended statistics есть цена:

- `ANALYZE` становится тяжелее;
- planning получает чуть больше работы;
- системные каталоги `pg_statistic_ext` и `pg_statistic_ext_data` растут.

Эта цена обычно мала по сравнению с выигрышем от исправленного плана, но она не нулевая.

Практическое правило простое:

- колонки должны иметь сильную корреляцию или почти функциональную зависимость;
- эти колонки должны реально использоваться вместе в `WHERE`;
- misestimation должно плохо влиять на физический plan shape, обычно через выбор join algorithm;
- только тогда `CREATE STATISTICS ... (mcv)` окупается.

Типичные кандидаты:

- `status + type`;
- `country + city`;
- `payment_provider + payment_method`;

Если колонки редко встречаются вместе в predicates или ошибка кардинальности не влияет на physical plan, MCV будет просто дополнительным шумом в metadata.

## Historical context

`CREATE STATISTICS` появился раньше PostgreSQL 12, но именно **multi-column MCV statistics** были добавлены в PostgreSQL 12.

- до PostgreSQL 12 у инженеров не было встроенного MCV-механизма для частот конкретных multi-column combinations;
- в части кейсов использовали expression indexes, materialized helper columns или другие обходные пути;
- эти обходные пути имели write penalty, расходовали RAM и диск и добавляли operational complexity;
- metadata-level MCV такого write penalty не имеет, потому что не меняет путь записи строк, а только улучшает статистическую модель planner'а после `ANALYZE`.

## Measured behavior on real hardware

На Linux-машине, которая довольно близка к обычному серверному профилю, такой synthetic case проявляется именно так, как и ожидается от статьи. На ноутбуке с `16 GB RAM`, `AMD Ryzen 5 4500U`, `SSD`, `Debian 13`. PostgreSQL запущен через `docker-compose` с лимитом памяти `1 GB` — чтобы рабочее множество не помещалось целиком в кэш и I/O-профиль оставался реалистичным. На таблицах по `10 млн` строк baseline-сценарий (в среднем за 5 прогонов) показал:

```text
before MCV:
Nested Loop
rows=99734
actual rows=1000000
Execution Time: ~120000 ms

after MCV:
Hash Join
rows=985001
actual rows=1000000
Execution Time: ~40000 ms
```

Это и есть канонический outcome: `10x` underestimate на base relation, `Nested Loop -> Hash Join` после `MCV`, и уже явный wall-clock выигрыш у более линейного плана.

На очень быстром laptop-class hardware картина может быть менее чистой. На MacBook с Apple Silicon planner bug остается виден: `E-Rows` занижены, join strategy после `MCV` меняется, но улучшение от перехода на `Hash Join` может оказаться менее заметным или даже проиграть по скорости `Nested Loop`. Это не опровержение механизма. Это напоминание, что результат зависит не только от абстрактной математики planner cost model, но и от физики конкретного железа: CPU cache locality, состояния page cache и дисковой подсистемы.

Здесь важен и более практический caveat. Для latency benchmarking баз данных и брокеров macOS, включая Apple Silicon, не стоит считать эталонной средой. Docker Desktop работает внутри Linux VM, а результаты сильнее зависят от hypervisor, page cache и host storage behavior, чем на native Linux. Канонические measurements лучше снимать на Linux-железе, близком к production. Mac в этом контексте полезен для локальной диагностики, проверки plan shape и быстрой валидации механизма, но не как окончательный источник latency-выводов.

## Conclusion

В этом классе проблем решением является не индекс, а правильная статистическая модель данных.

Пока planner считает скоррелированные predicates независимыми, он занижает `Estimated Rows` на уровне base relation, из-за чего недооценивает размер outer side и выбирает `Nested Loop` там, где физика исполнения уже требует алгоритмов с линейной сложностью (например, Hash Join). Индексы лишь делают этот ошибочный выбор еще более убедительным в cost model.

Если бизнес-данные содержат устойчивые multi-column correlations, а ошибка в cardinality estimate меняет join strategy, то чинить надо не `JOIN` и не индекс, а саму модель селективности через `CREATE STATISTICS ... (mcv)`.

Полный код инициализации таблиц, генерации synthetic skewed data и SQL-запросы для воспроизведения доступны в [репозиторий на GitHub ](https://github.com/nikita-kibitkin/postgres-mcv-demo).
