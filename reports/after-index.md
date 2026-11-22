# Замер after-index

## Условия опыта

- дата: 2026-11-22 12:08:02Z
- коммит: 6f0b725
- ядер у docker: 16
- память docker: 8318644224 байт
- лимиты app: 500000000 nanocpus, 536870912 байт
- сценарий: load/mixed.ts
- профиль: load/profile.json
- объем данных: 2 000 302 поездки, 10 000 пассажиров, таблица trips 423 MB

## Итоги прогона

Числа в reports/after-index.json.

## Вывод

Ни в какой пределе не уперлись, ошибки отсутствуют.

```
█ THRESHOLDS

    dropped_iterations
    ✓ 'count==0' count=0

    http_req_duration{scenario:history}
    ✓ 'p(95)<30000' p(95)=3.81ms

    http_req_failed
    ✓ 'rate<=0.1' rate=0.00%


  █ TOTAL RESULTS

    checks_total.......: 953     23.81971/s
    checks_succeeded...: 100.00% 953 out of 953
    checks_failed......: 0.00%   0 out of 953

    ✓ trip is read
    ✓ history is read
    ✓ trip is created

    HTTP
    http_req_duration..............: avg=3.13ms min=481.7µs  med=2.74ms max=16.02ms p(90)=5.94ms p(95)=6.55ms
      { expected_response:true }...: avg=3.13ms min=481.7µs  med=2.74ms max=16.02ms p(90)=5.94ms p(95)=6.55ms
      { scenario:history }.........: avg=2.58ms min=657.75µs med=2.55ms max=5.52ms  p(90)=3.47ms p(95)=3.81ms
    http_req_failed................: 0.00%  0 out of 953
    http_reqs......................: 953    23.81971/s

    EXECUTION
    dropped_iterations.............: 0      0/s
    iteration_duration.............: avg=3.43ms min=567.12µs med=3.06ms max=16.67ms p(90)=6.23ms p(95)=6.87ms
    iterations.....................: 953    23.81971/s
    vus............................: 0      min=0        max=0
    vus_max........................: 65     min=65       max=65

    NETWORK
    data_received..................: 1.1 MB 26 kB/s
    data_sent......................: 103 kB 2.6 kB/s
```
