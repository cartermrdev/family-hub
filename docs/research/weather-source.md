# Which weather data source should the hub use?

Researched 2026-10-01 for issue #7. All facts come from each provider's own docs, pricing or terms pages, linked inline and listed under Sources.

## Answer

Use **Open-Meteo**. It is free for non-commercial use, needs no API key or account, covers the whole world, and returns current conditions plus up to 16 days of hourly and daily forecast in one call. The only obligation is a CC BY 4.0 credit line ("Weather data by Open-Meteo.com", linked) on the Wall Display. NWS is a good no-cost alternative but only works in the US. The household's location is unknown, so it can't be the default. WeatherKit needs a paid $99/yr Apple Developer membership. OpenWeatherMap needs an account and key, and its free plan has no daily forecast.

## Expected load

One Wall Display refreshing every 15 minutes uses about 96 calls a day, or about 2,900 a month. Even at one call a minute (about 1,440 a day, 43,000 a month) that is far below every free limit below. Call limits are not the deciding factor. Cost, sign-up friction, coverage and attribution are.

## Comparison

| | Open-Meteo | Apple WeatherKit (REST) | OpenWeatherMap | NWS (api.weather.gov) |
|---|---|---|---|---|
| Cost for this use | Free (non-commercial) [1][2] | $99/yr Apple Developer Program membership [7], which includes 500k calls/month [5] | Free plan [9], or One Call 3.0 with 1,000 free calls/day, then $0.0015 per call [9][10] | Free, open data "free to use for any purpose" [12] |
| Account / API key | None. A key is only for commercial plans [3] | Paid developer account, plus key/JWT signing setup [5][6] | Account and API key required. Key activates up to 2 h after sign-up [9][11] | No key. A `User-Agent` header identifying the app is required [12] |
| Call limits (free) | 600/min, 5,000/h, 10,000/day, 300,000/month [1][2] | 500,000/month per membership, no rollover [5] | Free plan: 60/min, 1,000,000/month [9]. One Call 3.0: 1,000/day free, and a default daily cap can be adjusted in billing settings to avoid charges [10][11] | Not published. "Generous" for typical use. On a 429-style error, retry after about 5 s [12] |
| Current + short forecast | Yes: `current`, hourly and daily, 7 days by default and up to 16 [3] | Yes (current, hourly, daily, alerts) [6] | Free plan: current weather plus 5-day/3-hour forecast only. Hourly and daily need One Call 3.0 or a higher plan [9][10] | Yes: 12-hour forecast periods, hourly forecast, alerts. Needs a `/points` lookup first, which can be cached [12] |
| Coverage | Global (blends NOAA, ECMWF, JMA, BOM, etc.) [3] | Global [5] | Global [9] | **US only** [12] |
| Attribution | CC BY 4.0. Show `<a href="https://open-meteo.com/">Weather data by Open-Meteo.com</a>` next to the data [4] | Must show the Apple Weather trademark and a link to the data-source attribution page. Alerts have extra rules [5][8] | Required on Free through Professional: "Weather data provided by OpenWeather", a link to openweathermap.org, and the logo [11] | None required (public-domain government data) [12] |
| Uptime / SLA | No uptime guarantee on the free tier [2] | Not stated for the included tier | Not stated for the free plan | Not stated |
| Other terms | Free tier is non-commercial only. Open-Meteo may block misuse without notice [1] | Calls are pooled per membership, not per app [5] | One Call 3.0 is a separate "One Call by Call" subscription with billing details on file [10][11] | — |

## Recommendation

1. **Primary: Open-Meteo.** It is the only option with zero sign-up, zero cost, global coverage and both current and daily forecast data from one endpoint. That fits a single-household, non-commercial hobby project and makes no assumption about where the household is. Put the credit line in a corner of the weather widget. Cache the response server-side, for example by refreshing every 10 to 15 minutes, so the display never gets near the limits.
2. **Fallback / alternative: NWS**, *only if the household is in the US*. It is free, needs no key and has no attribution burden. However, it needs a two-step lookup (`/points` → `/gridpoints/.../forecast`), uses a less convenient period-based forecast shape, and has unpublished rate limits.
3. **Not recommended:**
   - **WeatherKit:** a $99/yr developer membership for one wall screen is unjustified unless the developer already pays for one.
   - **OpenWeatherMap:** its free plan has no daily forecast, and One Call 3.0 means keeping billing details on file and relying on a daily cap to stay at $0. Both add friction and risk for no benefit over Open-Meteo.

Caveat: keep the hub's weather client behind a small adapter, so the provider can be swapped later (for example to NWS for US alerts) if the household's location or needs change.

## Sources

1. Open-Meteo Terms: https://open-meteo.com/en/terms
2. Open-Meteo Pricing: https://open-meteo.com/en/pricing
3. Open-Meteo Forecast API docs: https://open-meteo.com/en/docs
4. Open-Meteo Licence (CC BY 4.0, attribution snippet): https://open-meteo.com/en/licence
5. Apple WeatherKit Get Started (included calls, pricing tiers, attribution): https://developer.apple.com/weatherkit/get-started/
6. WeatherKit REST API docs: https://developer.apple.com/documentation/weatherkitrestapi/
7. Apple Developer Program, what's included ($99/yr): https://developer.apple.com/programs/whats-included/
8. WeatherKit data-source attribution: https://developer.apple.com/weatherkit/data-source-attribution/
9. OpenWeatherMap Pricing: https://openweathermap.org/price
10. OpenWeatherMap One Call API 3.0: https://openweathermap.org/api/one-call-3
11. OpenWeatherMap FAQ (key activation, attribution, daily limits): https://openweathermap.org/faq
12. NWS API Web Service docs: https://www.weather.gov/documentation/services-web-api
