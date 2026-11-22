---
status: accepted
date: 2026-11-22
---

# Составной индекс (passenger_id, created_at DESC) для истории поездок

Составной индекс (passenger_id, created_at DESC) для истории поездок

## Контекст и проблема

По базовым замерам мы почти достигли требования p(95)<50ms. Необходимо исследовать запрос на выборку истории.

Запрос на котором проводим тестирование:
```
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF, COSTS OFF)
SELECT id, price, created_at
FROM trips
WHERE passenger_id = 1234
ORDER BY created_at DESC
LIMIT 20;
```


Результаты EXPLAIN запроса:
```
                                QUERY PLAN
 Limit (actual rows=20.00 loops=1)
   Buffers: shared hit=15934 read=38294 dirtied=1
   ->  Gather Merge (actual rows=20.00 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=15934 read=38294 dirtied=1
         ->  Sort (actual rows=16.00 loops=3)
               Sort Key: created_at DESC
               Sort Method: top-N heapsort  Memory: 27kB
               Buffers: shared hit=15934 read=38294 dirtied=1
               Worker 0:  Sort Method: top-N heapsort  Memory: 27kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 27kB
               ->  Parallel Seq Scan on trips (actual rows=66.67 loops=3)
                     Filter: (passenger_id = 1234)
                     Rows Removed by Filter: 666650
                     Buffers: shared hit=15858 read=38294 dirtied=1
 Planning:
   Buffers: shared hit=86 read=5
 Planning Time: 1.179 ms
 Execution Time: 49.368 ms
(20 rows)
```

Показатель "Rows Removed by Filter: 666650" говорит нам о том, что слишком много отброшенных строк в каждом проходе плана узла.

## Какие варианты сравнивали

Вариант 1 - составной индекс (passenger_id, created_at).  
Позволяем сразу выбрать необходимые нам отсортированные записи по конкретному пассажиру.

```
CREATE INDEX trips_passenger_created ON trips (passenger_id, created_at DESC);
```

План запроса после применения индекса:
```
Limit (actual rows=20.00 loops=1)
  Buffers: shared hit=23 read=3
  ->  Index Scan using trips_passenger_created on trips (actual rows=20.00 loops=1)
        Index Cond: (passenger_id = 1234)
        Index Searches: 1
        Buffers: shared hit=23 read=3
Planning:
  Buffers: shared hit=22 read=1
Planning Time: 0.736 ms
Execution Time: 0.393 ms
(10 rows)
```

По Index Cond видно что индекс отработал, исчезла сортировка. Число обращений к страницам (Buffers) сильно уменьшилось по сравнению с базовым замером. План запроса использует индекс и не читает таблицу целиком, чего мы и добивались.

Отчет на нагрузочное тестирование после добавления индекса - reports/after-index.md

Вариант 2 - индекс на (passenger_id).

```
CREATE INDEX trips_passenger ON trips (passenger_id);
```

```
Limit (actual rows=20.00 loops=1)
  Buffers: shared hit=59 read=144
  ->  Sort (actual rows=20.00 loops=1)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 27kB
        Buffers: shared hit=59 read=144
        ->  Bitmap Heap Scan on trips (actual rows=200.00 loops=1)
              Recheck Cond: (passenger_id = 1234)
              Heap Blocks: exact=200
              Buffers: shared hit=59 read=144
              ->  Bitmap Index Scan on trips_passenger (actual rows=200.00 loops=1)
                    Index Cond: (passenger_id = 1234)
                    Index Searches: 1
                    Buffers: shared read=3
Planning:
  Buffers: shared hit=17 read=1
Planning Time: 0.206 ms
Execution Time: 0.628 ms
(18 rows)
```

Хоть Index Cond и присутствует, появляется узел Sort. Также значительно выросли Buffers. 

Такой вариант не принимаем: индекс находит строки пассажира, но PostgreSQL читает всю его историю (200 строк, 200 страниц таблицы) и сортирует ее ради 20 записей. Чем больше поездок у пассажира, тем дольше запрос.

Вариант 3 - оставить как есть.
Такой вариант не принимаем: p95 = 49.83 мс формально укладывается в цель, но запрос читает всю таблицу (Parallel Seq Scan), и время растет вместе с ней. Сейчас в trips ~2 млн строк, за год будет ~73 млн, поэтому цель 50 мс нарушится уже через несколько дней роста данных.

Отчет на нагрузочное тестирование без индекса - reports/baseline.md.

## Решение

Выбран вариант 1 с составным индексом как дающий самый хороший план выполнения запроса. Создана миграция в db/migrations/007-passenger-history-index.sql.

Условие отката: удаляем индекс (`DROP INDEX IF EXISTS trips_passenger_created;`), если план запроса истории перестанет его использовать или запись поездок заметно замедлится из-за обновления индекса.

## Последствия

Хорошее.
Запрос выполняется быстрее, меньше чтения страниц. 

Плохое.
Индекс занимает место. Необходимо это принимать во внимание. 

## Подтверждение

### Результаты нагрузочного тестирования

До добавления индекса:
```
dropped_iterations
   ✓ 'count==0' count=0

   http_req_duration{scenario:history}
   ✓ 'p(95)<30000' p(95)=49.83ms

   http_req_failed
   ✓ 'rate<=0.1' rate=0.00%
```

После добавления индекса:
```
dropped_iterations
   ✓ 'count==0' count=0

   http_req_duration{scenario:history}
   ✓ 'p(95)<30000' p(95)=3.81ms

   http_req_failed
   ✓ 'rate<=0.1' rate=0.00%
```

http_req_duration{scenario:history} улучшился в 10+ раз. С учетом того что размер индекса будет расти, необходимо слежение за его размером.

```
SELECT
    pg_size_pretty(pg_relation_size('trips')) AS table_size,
    pg_size_pretty(pg_indexes_size('trips')) AS indexes_size;
```

Профиль нагрузки показывает, что внесенные изменения положительно повлияли на среднее время ответа на запрос истории поездок пользователя.
