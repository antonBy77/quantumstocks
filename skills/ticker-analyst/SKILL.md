---
name: ticker-analyst
description: Глубокий разбор одного тикера через QuantumStock MCP (QSmcp) — фундаментал и скоринг, теханализ, Smart Money Concepts, HMM-режим, ML-прогноз LSTM-ансамблем, опционы/GEX, риск-метрики, confluence-вердикт, новости компании. Использовать когда просят «проанализируй тикер», «что по NVDA», «стоит ли заходить в X», «дай вердикт/прогноз по акции», нужен confluence BUY/SELL/HOLD с обоснованием.
---

# Глубокий разбор тикера через QuantumStock MCP

Сервер: `QSmcp`. Время UTC. Дата в `predict_future_price_lstm` — формат `YYYY-MM-DD-HH-MM`.

## Правила платформы (кратко, полное — в скилле market-analyst)

- Сначала дешёвое (`fast_*`, синхронная техника), тяжёлое — потом и по одному.
- 202-ответ с `job_id` — расчёт в фоне: `get_job_status(job_id)` через 30–60с, либо повторить вызов через минуту.
- Таймаут ~50с — деградация облака: пропустить/повторить, не паниковать, параллелить тяжёлое не надо.
- 401 — истёк или не задан токен `QS_ACCESS_TOKEN` (см. market-analyst и README плагина).

## Воркфлоу разбора (все пункты — один тикер T)

Слои идут от быстрого к тяжёлому. Для быстрого вопроса можно останавливаться после п.3–5; полный разбор — все слои.

1. **Finviz-метрики (секунды, опционально)**: `fast_finviz_ticker(ticker=T)` — базовая панель. ⚠️ Finviz-кэш на сервере часто не прогрет: 404 «Нет finviz данных» — штатный ответ, шаг пропускается без ущерба для разбора. Произвольный dtype из Redis: `fast_ticker_dtype(ticker=T, dtype=D)`; наличие в кэше — `warmup_status`.
2. **Фундаментал**: `get_company_fundamentals(ticker_symbol=T)` — профиль, выручка/маржа, P/E, EPS, cap, 52w, таргеты аналитиков, FRED-контекст. Глубже: `get_advanced_company_fundamentals(ticker_symbol=T, detail_level="standard"|"deep")` (3 уровня + финрейтинг) и `get_company_financial_scores(ticker_symbols=T)` — можно несколько тикеров через запятую для сравнения.
3. **Техника**: `get_technical_analysis(ticker=T)` — RSI, MACD/signal/hist, Bollinger (upper/middle/lower), EMA 20/50/100, ATP. Читать: RSI>70/<30, цена vs BB и EMA-стек, MACD-гистограмма.
4. **Smart Money (SMC)**: `get_smart_money_analysis(ticker=T, period="3mo", interval="1h", swing_length=20)` — структура рынка, Order Blocks, FVG, FTM-сигналы, зоны ликвидности.
5. **Режим HMM**: `get_hmm_regime(ticker=T, period="1y", interval="1d", n_states=5)` — какой режим сейчас и вероятности переходов; прогноз ML взвешивается по нему при `use_hmm=True`.
6. **Прогноз ML**: `get_ml_forecast(ticker=T, window=160, use_hmm=True)` — Sharpe-weighted ансамбль 5 LSTM-архитектур + HMM consensus; направление (UP/DOWN) и оценка качества. Классическая регрессия цен: `predict_future_price_lstm(ticker, start_date, end_date, interval ∈ 1m/5m/15m/1h/1d, future_steps 10–100, time_step 100–512)` — возвращает MAE/MSE/RMSE/MAPE, VaR 95%, ATR, волатильность и рекомендованные стопы (`stop_loss_var/atr/volatility`). **ML-прогноз — не гарантия**: всегда показывать метрики качества рядом с прогнозом. Кэш ML/LSTM обычно холодный (`warmup_status`: ml/lstm = 0) — вызов уходит в COMPUTE: жди 202+job_id → `get_job_status` через 30–60с, либо повтори через минуту.
7. **Опционы/GEX**: `get_options_expirations(ticker=T)` → `get_options_data(ticker=T, expiration_date=ближайшая)` — греки, OI, Max Pain; глубже `analytics_gex(ticker=T)` — net GEX, режим гаммы, gamma flip, call/put walls, PCR, IV-улыбка. Стены и флип — уровни, где маркетмейкеры создают притяжение/отталкивание.
8. **Риск**: `analytics_enriched(ticker=T)` — GARCH(1,1)-вол, VaR 95/99, CVaR, ATR, Sharpe, drawdown, skew/kurt (может быть 202+job). `analytics_vwap(ticker=T)` — VWAP, z-score, зона (overbought/oversold), σ-полосы.
9. **Confluence-вердикт**: `analytics_signals(ticker=T)` — BUY/SELL/HOLD + confluence_score 0–1 + сколько источников согласно. Это ядро вывода.
10. **Новости**: `get_company_news(ticker=T)` — 5 последних (Yahoo RSS).

Локальная панель опционов (история страйков, ΔDEX, конструктор стратегий) — отдельный локальный скилл `qs-options-gex` (порт :3085), не через MCP.

## Формат вердикта

Заголовок: тикер, цена-контекст, confluence-вердикт и score. Затем по блокам: фундаментал (оценка vs рост, таргеты) → техника (тренд, перекупленность) → SMC (ключевые зоны OB/FVG) → HMM-режим → ML-прогноз (с метриками качества!) → опционные уровни (стены, max pain, flip) → риск (VaR, GARCH-вол, просадка) → новости, которые двигают. Финал: bullish/bearish/нейтрально-касс с 2–3 главными аргументами за и против, ключевые уровни (стены/флип/BB/стопы из VaR). Ничего не выдумывать: недоступный блок помечать как недоступный.
