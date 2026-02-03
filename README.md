-- CTE для вычисления основных статистик по событиям
WITH event_stats AS (
  SELECT platform,
         referrer,
         COUNT(*) AS total_events, -- Общее количество событий
         COUNT(DISTINCT user_id) AS unique_users, -- Количество уникальных пользователей
         COUNT(*)::float / COUNT(DISTINCT user_id) AS avg_events_per_user -- Среднее количество событий на пользователя
  FROM tools_shop.events
  GROUP BY platform, referrer
), 
-- CTE для подсчёта событий с пустой страницей перехода
empty_referrer_stats AS (
  SELECT platform,
         COUNT(*) AS total_empty_events -- Общее количество событий без информации о странице, с которой был сделан переход
  FROM tools_shop.events
  WHERE referrer IS NULL
  GROUP BY platform
), 
-- CTE для подсчёта общего количества событий по платформам
platform_activity AS (
  SELECT platform,
         SUM(total_events) AS total_platform_events -- Общее количество событий по платформам
  FROM event_stats
  GROUP BY platform
), 
-- CTE для определения платформы с наибольшей активностью
most_active_platform AS (
  SELECT platform
  FROM platform_activity
  ORDER BY total_platform_events DESC
  LIMIT 1
), 
-- CTE для определения платформы с наименьшей активностью
least_active_platform AS (
  SELECT platform
  FROM platform_activity
  ORDER BY total_platform_events ASC
  LIMIT 1
) 
-- Основной запрос для получения всех данных и вычисления
-- доли событий с пустой информацией о странице, с которой был сделан переход
SELECT es.platform,
       es.referrer,
       es.total_events,
       es.unique_users,
       es.avg_events_per_user,
       COALESCE(ers.total_empty_events, 0)::float / es.total_events AS empty_referrer_share,
       (SELECT platform FROM most_active_platform) AS most_active_platform,
       (SELECT platform FROM least_active_platform) AS least_active_platform
FROM event_stats AS es
LEFT JOIN empty_referrer_stats AS ers ON es.platform = ers.platform
ORDER BY es.platform, es.referrer;
