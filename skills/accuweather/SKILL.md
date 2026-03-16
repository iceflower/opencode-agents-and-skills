---
name: accuweather
description: >-
  AccuWeather-based weather information lookup rules.
  Use when users ask for weather forecasts, hourly forecasts, weather conditions,
  sunrise/sunset times, UV index, or air quality (미세먼지/초미세먼지).
---

# AccuWeather Weather Information Rules

## 1. Overview

AccuWeather provides detailed weather forecasts including hourly forecasts, daily forecasts, air quality, and RealFeel temperature. Use the AccuWeather website to fetch weather information.

## 2. URL Structure

### Base URL

```text
https://www.accuweather.com/ko/kr/{city}/{location-key}/{forecast-type}/{location-key}
```

### URL Parameters

| Parameter         | Description                     | Example         |
| ----------------- | ------------------------------- | --------------- |
| `{city}`          | City name in Korean URL format  | `seoul`         |
| `{location-key}`  | AccuWeather location identifier | `226081`        |
| `{forecast-type}` | Type of forecast                | See table below |

### Forecast Types

| Type                       | Description                 |
| -------------------------- | --------------------------- |
| `weather-forecast`         | Today's weather             |
| `hourly-weather-forecast`  | Hourly forecast             |
| `daily-weather-forecast`   | 30-day daily forecast       |
| `minute-weather-forecast`  | MinuteCast (precipitation)  |
| `air-quality-index`        | Air quality                 |

### Example URLs

```text
# Today's weather
https://www.accuweather.com/ko/kr/seoul/226081/weather-forecast/226081

# Hourly forecast (tomorrow)
https://www.accuweather.com/ko/kr/seoul/226081/hourly-weather-forecast/226081?day=2

# Daily forecast
https://www.accuweather.com/ko/kr/seoul/226081/daily-weather-forecast/226081
```

## 3. Location Keys

### Major Korean Cities

| City          | Location Key |
| ------------- | ------------ |
| 서울특별시    | `226081`     |
| 부산광역시    | `226082`     |
| 대구광역시    | `226083`     |
| 인천광역시    | `226084`     |
| 광주광역시    | `226085`     |
| 대전광역시    | `226086`     |
| 울산광역시    | `226087`     |
| 세종특별자치시| `2330435`    |
| 강릉시        | `226088`     |
| 춘천시        | `226089`     |
| 전주시        | `226090`     |
| 청주시        | `226091`     |
| 제주시        | `226092`     |
| 서귀포시      | `226093`     |

### Finding Location Keys

If a location key is unknown:

1. Search on AccuWeather website: `https://www.accuweather.com/ko/kr/{search-term}`
2. Use web search: "AccuWeather {location name} location key"
3. Check the URL after searching for the city on AccuWeather

## 4. Hourly Forecast Query

### URL with Day Parameter

```text
# Today
https://www.accuweather.com/ko/kr/{city}/{key}/hourly-weather-forecast/{key}?day=1

# Tomorrow
https://www.accuweather.com/ko/kr/{city}/{key}/hourly-weather-forecast/{key}?day=2

# Day after tomorrow
https://www.accuweather.com/ko/kr/{city}/{key}/hourly-weather-forecast/{key}?day=3
```

### Hourly Forecast Availability

**IMPORTANT**: AccuWeather hourly forecasts are **only available for up to 3 days** (today + 2 days). For dates beyond 3 days, hourly data cannot be retrieved. In such cases, inform the user and provide daily forecast instead.

| Day Parameter | Availability |
| ------------- | ------------ |
| `day=1`       | Today        |
| `day=2`       | Tomorrow     |
| `day=3`       | Day after tomorrow |
| `day=4+`      | **Not available** - use daily forecast |

### Hourly Data Points

- Temperature (기온)
- RealFeel temperature (체감온도)
- Weather condition (날씨 상태)
- Precipitation probability (강수 확률)
- Humidity (습도)
- Wind direction and speed (풍향/풍속)
- Air quality (대기질)
- Cloud cover (구름량)

## 5. Daily Forecast Query

### Daily Forecast URL

```text
https://www.accuweather.com/ko/kr/{city}/{key}/daily-weather-forecast/{key}
```

### Daily Data Points

- High/Low temperature (최고/최저 기온)
- RealFeel temperature range
- Weather condition
- Precipitation probability
- UV index (자외선 지수)
- Wind speed

### Day Parameter for Specific Date

```text
# Day 3 (2 days from today)
https://www.accuweather.com/ko/kr/seoul/226081/daily-weather-forecast/226081?day=3
```

## 6. Sunrise/Sunset Query

### Sun/Moon URL

Sunrise and sunset times are available on the `weather-today` page:

```text
https://www.accuweather.com/ko/kr/{city}/{key}/weather-today/{key}
```

### Available Data

AccuWeather provides the following sun/moon information:

| Data             | Korean Term |
| ---------------- | ----------- |
| Sunrise          | 일출        |
| Sunset           | 일몰        |
| Moonrise         | 월출        |
| Moonset          | 월몰        |
| Day length       | 가조 시간   |
| Moon phase       | 달 위상     |

### Example Output

```markdown
**해와 달**

- 일출: 오전 6:47
- 일몰: 오후 6:37
- 가조 시간: 11시간 50분
- 월출: 오전 3:20
- 월몰: 오후 12:27
```

### Alternative Source: KASI (한국천문연구원)

For more precise astronomical data, use the Korea Astronomy and Space Science Institute (KASI) website:

| Service                          | URL                                       |
| -------------------------------- | ----------------------------------------- |
| 일출일몰시각계산                 | `https://astro.kasi.re.kr/life/pageView/9`|
| 월별 해/달 출몰시각              | `https://astro.kasi.re.kr/life/pageView/6`|

KASI provides:

- Sunrise/sunset times (일출/일몰)
- Moonrise/moonset times (월출/월몰)
- Civil twilight (시민박명)
- Nautical twilight (항해박명)
- Astronomical twilight (천문박명)

## 7. UV Index Query

### UV Index in Daily Forecast

UV index is available in daily forecast and weather-today pages:

```text
# Daily forecast with UV index
https://www.accuweather.com/ko/kr/{city}/{key}/daily-weather-forecast/{key}
```

### UV Index Scale

| Index Range | Korean Level        | English Level       |
| ----------- | ------------------- | ------------------- |
| 0-2         | 좋음                | Good                |
| 3-5         | 보통                | Moderate            |
| 6-7         | 해로움 (민감 그룹)  | High                |
| 8-10        | 건강에 해로움       | Very High           |
| 11+         | 매우 해로움         | Extreme             |

### UV Index Output Example

```markdown
- 자외선지수: 6 (해로움 - 민감 그룹)
```

## 8. Air Quality Query

### Air Quality URL

```text
https://www.accuweather.com/ko/kr/{city}/{key}/air-quality-index/{key}
```

### AQI (Air Quality Index)

AccuWeather provides comprehensive air quality data including:

| Pollutant | Korean Name     | Description                          |
| --------- | --------------- | ------------------------------------ |
| PM2.5     | 초미세먼지      | Particles ≤ 2.5 micrometers          |
| PM10      | 미세먼지        | Particles ≤ 10 micrometers           |
| O3        | 오존            | Ground-level ozone                   |
| NO2       | 이산화질소      | Nitrogen dioxide                     |
| SO2       | 이산화황        | Sulfur dioxide                       |
| CO        | 일산화탄소      | Carbon monoxide                      |

### AQI Scale

| AQI Range | Korean Level      | English Level    | Health Impact                          |
| --------- | ----------------- | ---------------- | -------------------------------------- |
| 0-19      | 완벽함            | Perfect          | Suitable for all outdoor activities    |
| 20-49     | 보통              | Fair             | Acceptable for most people             |
| 50-99     | 나쁨              | Poor             | Sensitive groups may be affected       |
| 100-149   | 건강에 해로움     | Unhealthy        | Limit outdoor activities               |
| 150-249   | 건강에 매우 해로움| Very Unhealthy   | Avoid outdoor activities               |
| 250+      | 위험              | Hazardous        | Everyone should avoid outdoor exposure |

### Air Quality Output Example

```markdown
**대기질**

- AQI: 55 (나쁨)
- PM2.5 (초미세먼지): 30 µg/m³ (보통)
- PM10 (미세먼지): 17 µg/m³ (완벽함)
- 오존 (O3): 50 µg/m³ (완벽함)
```

## 9. Output Format

### Hourly Forecast Table

Present hourly data in a table format:

```markdown
| 시간 | 기온 | 체감온도 | 날씨 | 강수확률 | 습도 | 풍향/풍속 |
|------|------|----------|------|----------|------|-----------|
| 0시  | 3℃  | 4℃      | 흐림 | 0%       | 56% | 북서 6km/h |
```

### Daily Forecast Summary

```markdown
**날짜** (요일)

- 날씨: [상태]
- 기온: 최저 X℃ / 최고 Y℃
- 체감온도: X℃ ~ Y℃
- 강수확률: X%
- 자외선지수: X (등급)
- 풍속: X km/h
```

### Air Quality Summary

```markdown
**대기질**

- AQI: X (등급)
- 초미세먼지(PM2.5): X µg/m³ (등급)
- 미세먼지(PM10): X µg/m³ (등급)
```

## 10. Weather Terms (Korean)

| Korean Term     | English Term           |
| --------------- | ---------------------- |
| 맑음            | Clear/Sunny            |
| 대체로 맑음     | Mostly clear           |
| 일부 화창       | Partly sunny           |
| 간헐적으로 흐림 | Intermittent clouds    |
| 약간 흐림       | Mostly cloudy          |
| 대체로 흐림     | Mostly cloudy          |
| 흐림            | Cloudy                 |
| 구름이 줄어듦   | Decreasing clouds      |
| 점차 흐려짐     | Increasing clouds      |
| 비              | Rain                   |
| 소나기          | Shower                 |
| 가벼운 비       | Light rain             |
| 눈              | Snow                   |
| 비 또는 눈      | Rain or snow           |
| 천둥            | Thunderstorm           |

## 11. Best Practices

### Query Strategy

1. For current weather: Use `weather-forecast` page
2. For hourly forecast: Use `hourly-weather-forecast` with `day` parameter
3. For multi-day forecast: Use `daily-weather-forecast`
4. For sunrise/sunset: Use `weather-today` page
5. For air quality: Use `air-quality-index` page

### Information Prioritization

When user asks for weather without specifying details:

1. Temperature (high/low or current)
2. Weather condition
3. Precipitation probability
4. Wind information
5. UV index and Air quality (if relevant)
6. Sunrise/sunset (if relevant)

### Time Reference

- Always clarify the date being referenced (today, tomorrow, specific date)
- Use 24-hour format for time in Korean context
- Note the forecast issuance time if available

### Attribution

Always cite AccuWeather as the source:

```markdown
출처: AccuWeather
```

## 12. Limitations

- AccuWeather website may change structure; adapt as needed
- Some detailed data may require JavaScript rendering
- Location keys may change; verify periodically
- Forecast accuracy decreases with longer time ranges
- **Hourly forecasts are only available for up to 3 days** (day=1 to day=3). For dates beyond 3 days, only daily forecasts can be provided.
