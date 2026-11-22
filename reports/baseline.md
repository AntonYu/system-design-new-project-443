# Замер baseline

## Условия опыта

- дата: 2026-11-22 11:35:20Z
- коммит: 6f0b725
- ядер у docker: 16
- память docker: 8318644224 байт
- лимиты app: 500000000 nanocpus, 536870912 байт
- сценарий: load/mixed.ts
- профиль: load/profile.json
- объем данных: 2 000 302 поездки, 10 000 пассажиров, таблица trips 423 MB

## Итоги прогона

Числа в reports/baseline.json.

## Вывод

HTTP
    http_req_duration..............: avg=8.67ms  min=430.45µs med=888.2µs max=53.08ms p(90)=45.85ms p(95)=47.78ms
      { expected_response:true }...: avg=8.67ms  min=430.45µs med=888.2µs max=53.08ms p(90)=45.85ms p(95)=47.78ms
      { scenario:history }.........: avg=45.97ms min=32.52ms  med=46.58ms max=53.08ms p(90)=48.98ms p(95)=49.83ms
    http_req_failed................: 0.00%  0 out of 953
    http_reqs......................: 953    23.795173/s
    
p(95) загрузки истории почти подошел к нашему лимиту в системе (50ms).

EXECUTION
    dropped_iterations.............: 0      0/s
    iteration_duration.............: avg=8.8ms   min=472µs    med=982.5µs max=53.34ms p(90)=46ms    p(95)=47.87ms
    iterations.....................: 953    23.795173/s
    vus............................: 0      min=0        max=0
    vus_max........................: 65     min=65       max=65


THRESHOLDS

  dropped_iterations
  ✓ 'count==0' count=0

  http_req_duration{scenario:history}
  ✓ 'p(95)<30000' p(95)=49.83ms

  http_req_failed
  ✓ 'rate<=0.1' rate=0.00%


Ошибок нет вообще.
