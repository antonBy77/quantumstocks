---
name: macro-fred
description: Макро-контекст через QuantumStock MCP (QSmcp) — данные FRED (ставки, инфляция, ВВП, рынок труда), поиск экономических серий, экономический календарь событий, структурированные индикаторы из аналитических блогов (CDS, VIX, GDPNow, FedOdds), RSS-потоки. Использовать когда спрашивают про макро («где ставка», «что с CPI», «когда ФРС»), нужен фон для рынка или контекст перед разбором тикера.
---

# Макро-контекст через QuantumStock MCP

Сервер: `QSmcp`. Время UTC. Правила платформы (таймауты облака, 202+job_id, 401=токен) — как в скилле market-analyst.

## FRED (Federal Reserve Economic Data)

1. **Найти серию**: `get_fred_series_search(search_text=...)` — полнотекстовый поиск, `limit` по умолчанию 100. Примеры запросов: «treasury yield 10 year», «inflation expectations».
2. **Взять данные**: `get_fred_series_data(series_id=...)` — частые ID: `DGS10`/`DGS2` (10Y/2Y доходности), `T10Y2Y` (кривая), `CPIAUCSL` (CPI), `PCEPI` (PCE), `GDP`/`GDPC1`, `UNRATE` (безработица), `PAYEMS` (payrolls), `DFF` (fed funds), `VIXCLS` (VIX), `DTWEXBGS` (доллар-индекс).
3. Для макро-фона конкретной компании FRED-контекст уже вшит в `get_company_fundamentals` — отдельно дергать не надо.

## Экономический календарь

`fast_economic_calendar` — из Redis (Forex Factory). Фильтры: `high_only=true` (только high-impact), `country="USD"` — **фильтр принимает код валюты, а не страны** (в данных `USD`/`JPY`/`EUR`/`All`; значение `"US"` молча вернёт пустой список), `refresh=true` принудительно обновить, `limit`. Всегда показывать: что вышло сегодня (факт vs прогноз) и что ждёт на ближайшие дни.

## Индикаторы из блогов (CDS / VIX / GDPNow / FedOdds)

- Список источников: `blog_sources_list`.
- Структурированные индикаторы: `blog_indicators_read(blog=<имя>)` — распарсенные CDS-спреды, VIX, GDPNow, вероятности ФРС; `refresh=true` для принудительного обновления. Быстрый вариант из Redis: `fast_indicators(blog=<имя>)`.

## RSS-потоки

`get_rss_sources` — доступные источники → `get_rss_feed(source=<имя>)` полный парсинг или `fast_news(source=<имя>)` из Redis. Глобальный брифинг одним вызовом: `get_global_market_news` (Markdown; медленный, при таймауте фолбэк на fast_news).

## Типовые связки

- «Что с ставками и кривой»: `get_fred_series_data(T10Y2Y)` + `get_fred_series_data(DGS10)` + `fast_economic_calendar(high_only=true)` + FedOdds из `blog_indicators_read`.
- «Инфляция vs рынок»: `CPIAUCSL` + `PCEPI` + календарь.
- «Макро-фон перед разбором тикера»: вызвать перед ticker-analyst; серии по ситуации, календарь на неделю.
