# Анализ данных сетевого трафика с использованием аналитической
in-memory СУБД DuckDB


## Цель

1.  Изучить возможности СУБД DuckDB для обработки и анализ больших
    данных

2.  Получить навыки применения DuckDB совместно с языком
    программирования R

3.  Получить навыки анализаметаинфомации о сетевом трафике

4.  Получить навыки применения облачных технологий хранения, подготовки
    и анализа данных: Yandex Object Storage, Rstudio Server.

## ️Исходные данные

1.  Ноутбук c Windows 11 и установленным Git, R и RStudio, ssh

2.  Доступ к RStudio Server

## ️Ход работы

1.  Подключиться по ssh к удаленному серверу и установить сетевой
    туннель

2.  Подключиться к RStudio Server

3.  Решение заданий:

    1.  Надите утечку данных из Вашей сети.

    2.  Надите утечку данных 2

    3.  Надите утечку данных 3

4.  Оформить отчет в соответствии с шаблоном

## Содержание ЛР

### Задание 1

Важнейшие документы с результатами нашей исследовательской деятельности
в области создания вакцин скачиваются в виде больших заархивированных
дампов. Один из хостов в нашей сети используется для пересылки этой
информации – он пересылает гораздо больше информации на внешние ресурсы
в Интернете, чем остальные компьютеры нашей сети. Определите его
IP-адрес.

``` r
library(duckdb)
```

    Loading required package: DBI

``` r
con <- dbConnect(duckdb::duckdb())


df <- dbGetQuery(con,
    "SELECT src FROM (SELECT src, SUM(bytes) AS bytesSum
    FROM read_parquet('tm_data.pqt')
    WHERE (src LIKE '12.%' OR src LIKE '13.%' OR src LIKE '14.%') 
    AND (dst NOT LIKE '12.%' AND dst NOT LIKE '13.%' AND dst NOT LIKE '14.%')
    GROUP BY src ORDER BY bytesSum DESC LIMIT 1);")

print(df)
```

               src
    1 13.37.84.125

### Задание 2

Другой атакующий установил автоматическую задачу в системном
планировщике cron для экспорта содержимого внутренней wiki системы. Эта
система генерирует большое количество трафика в нерабочие часы, больше
чем остальные хосты.

Определите IP этой системы. Известно, что ее IP адрес отличается от
нарушителя из предыдущей задачи.

Установка ggplot2:

``` r
install.packages("ggplot2")
```

    Installing package into '/home/user24/R/x86_64-pc-linux-gnu-library/4.4'
    (as 'lib' is unspecified)

Подготовка данных:

``` r
library(duckdb)
library(dplyr)
```


    Attaching package: 'dplyr'

    The following objects are masked from 'package:stats':

        filter, lag

    The following objects are masked from 'package:base':

        intersect, setdiff, setequal, union

``` r
library(ggplot2)
library(lubridate)
```


    Attaching package: 'lubridate'

    The following objects are masked from 'package:base':

        date, intersect, setdiff, union

``` r
con <- dbConnect(duckdb::duckdb())

df <- dbGetQuery(con,
    "SELECT *
    FROM read_parquet('tm_data.pqt')
    WHERE (src LIKE '12.%' OR src LIKE '13.%' OR src LIKE '14.%') 
    AND (dst NOT LIKE '12.%' AND dst NOT LIKE '13.%' AND dst NOT LIKE '14.%' AND src NOT LIKE '13.37.84.125');")

dftime <- df %>%
      select(timestamp, src, dst, bytes) %>%
      mutate(time = hour(as_datetime(timestamp/1000)))

dfdatetime <- df %>%
      select(timestamp, src, dst, bytes) %>%
      mutate(time = as_datetime(timestamp/1000))
```

``` r
bytraffic <- dftime %>%
      group_by(time) %>%
      summarise(traffic = sum(bytes), count = n()) %>% 
      arrange(desc(traffic))

bycount <- bytraffic %>% arrange(desc(count))

avgPerHost <- bytraffic %>% mutate(avg = traffic/count) %>% arrange(desc(avg))

res <- dftime %>%
      filter(time < 16) %>% 
      group_by(src) %>%
      summarise(traffic = sum(bytes)) %>% #, num = n()) %>%
      #mutate(avg = traffic / num) %>%
      arrange(desc(traffic))
      #top_n(10)

res7 <- dftime %>%
      filter(time == 7) %>% 
      group_by(src) %>%
      summarise(traffic = sum(bytes)) %>% #, num = n()) %>%
      #mutate(avg = traffic / num) %>%
      arrange(desc(traffic))
      #top_n(10)

res1255 <- dfdatetime %>%
      filter(src == '12.55.77.96')
```

Анализ данных:

``` r
ggplot(data = bytraffic, aes(x = time, y = traffic)) + 
  geom_line() +
  geom_point()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-5-1.png)

``` r
ggplot(data = bytraffic, aes(x = time, y = count)) + 
  geom_line() +
  geom_point()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-6-1.png)

Так как с 16 до 24 объем трафика и количество запросов стабильно высоки
с сравнении с другими часами, этот интервал является рабочими часами.

``` r
ggplot(head(res, 10), aes(traffic, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-7-1.png)

Заметно, что в нерабочие часы трафик наибольший у 12.55.77.96

``` r
ggplot(data = avgPerHost, aes(x = time, y = avg)) +
  geom_line() + 
  geom_point()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-8-1.png)

Так как в 7 часов был значительный всплеск среднего трафика на хост, что
является аномалией, скорее всего именно в этот период происходила утечка
данных.

``` r
ggplot(head(res7, 10), aes(traffic, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-9-1.png)

Заметно, что в 7 часов трафик наибольший у 12.55.77.96.

``` r
ggplot(data = res1255, aes(x = time, y = bytes)) + 
  geom_line() +
  geom_point()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-10-1.png)

Каждый день у 12.55.77.96 был всплеск трафика в 7 утра. Таким образом,
именно у этой системы был настроен cron для экспорта содержимого
внутренний wiki системы.

### Задание 3

Еще один нарушитель собирает содержимое электронной почты и отправляет в
Интернет используя порт, который обычно используется для другого типа
трафика. Атакующий пересылает большое количество информации используя
этот порт, которое нехарактерно для других хостов, использующих этот
номер порта. Определите IP этой системы. Известно, что ее IP адрес
отличается от нарушителей из предыдущих задач.

Подготовка данных:

``` r
library(duckdb)
library(dplyr)
library(ggplot2)
library(lubridate)

con <- dbConnect(duckdb::duckdb())

df <- dbGetQuery(con,
  "SELECT *
  FROM read_parquet('tm_data.pqt')
  WHERE (src LIKE '12.%' OR src LIKE '13.%' OR src LIKE '14.%') 
  AND (dst NOT LIKE '12.%' AND dst NOT LIKE '13.%' AND dst NOT LIKE '14.%') 
  AND (src NOT LIKE '13.37.84.125' AND src NOT LIKE '12.55.77.96');")
```

``` r
dfport <- df %>%
  select(src, port, bytes)
```

``` r
dfportstat <- dfport %>%
  group_by(port) %>%
  summarise(med = median(bytes), max = max(bytes), razn = max - med) %>%
  arrange(desc(razn))

print(dfportstat)
```

    # A tibble: 46 × 4
        port    med    max    razn
       <int>  <dbl>  <int>   <dbl>
     1    37 30669  209402 178733 
     2    39 30713  198527 167814 
     3   105 30598. 197766 167168.
     4    40 30579  195144 164565 
     5    75 30685  194650 163965 
     6    89 30628  194106 163478 
     7   102 30645  193588 162943 
     8    81 30721  192430 161709 
     9   119 30623  190151 159528 
    10    74 30679  189818 159139 
    # ℹ 36 more rows

``` r
ggplot(data = dfportstat, aes(x = port, y = razn)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-14-1.png)

Скорее всего подозрительным портом является один из первой десятки тех,
у которых нибольшая разница между медианой и максимальным значением
переданного трафика за раз.

#### Порт 37

``` r
p = 37
dfportN <- dfport %>%
  filter(port == p) %>%
  group_by(src) %>%
  summarise(traffic = sum(bytes), count = n(), avg = traffic/count, med = median(bytes)) %>%
  arrange(desc(avg))

ggplot(head(dfportN, 10), aes(avg, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-15-1.png)

``` r
ggplot(head(dfportN, 10), aes(med, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-16-1.png)

Для прота 37 (топ-1) замечена аномальная активность на src =
14.31.207.42, но разница со вторым местом src = 14.42.60.94 минимальна.

#### Порт 39

``` r
p = 39
dfportN <- dfport %>%
  filter(port == p) %>%
  group_by(src) %>%
  summarise(traffic = sum(bytes), count = n(), avg = traffic/count, med = median(bytes)) %>%
  arrange(desc(avg))

ggplot(head(dfportN, 10), aes(avg, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-17-1.png)

``` r
ggplot(head(dfportN, 10), aes(med, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-18-1.png)

Для порта 39 (топ-2) замечена аномальная активность: src = 13.36.102.77.

#### Порт 105

``` r
p = 105
dfportN <- dfport %>%
  filter(port == p) %>%
  group_by(src) %>%
  summarise(traffic = sum(bytes), count = n(), avg = traffic/count, med = median(bytes)) %>%
  arrange(desc(avg))

ggplot(head(dfportN, 10), aes(avg, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-19-1.png)

``` r
ggplot(head(dfportN, 10), aes(med, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-20-1.png)

Для порта 105 (топ-3) замечена аномальная активность: src =
14.31.107.42.

#### Порт 40

``` r
p = 40
dfportN <- dfport %>%
  filter(port == p) %>%
  group_by(src) %>%
  summarise(traffic = sum(bytes), count = n(), avg = traffic/count, med = median(bytes)) %>%
  arrange(desc(avg))

ggplot(head(dfportN, 10), aes(avg, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-21-1.png)

``` r
ggplot(head(dfportN, 10), aes(med, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-22-1.png)

Для порта 40 (топ-4) замечена аномальная активность: src = 13.41.85.73.

#### Порт 75

``` r
p = 75
dfportN <- dfport %>%
  filter(port == p) %>%
  group_by(src) %>%
  summarise(traffic = sum(bytes), count = n(), avg = traffic/count, med = median(bytes)) %>%
  arrange(desc(avg))

ggplot(head(dfportN, 10), aes(avg, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-23-1.png)

``` r
ggplot(head(dfportN, 10), aes(med, src)) + geom_col()
```

![](README.markdown_strict_files/figure-markdown_strict/unnamed-chunk-24-1.png)

Для порта 75 (топ-5) не замечена аномальная активность.

#### Результат

Наибольшая разница в трафике, который был направлен через один порт,
получена на порту 105. Ответ на задание: 14.31.107.42

## Оценка результатов

Задача выполнена при помощи DuckDB, удалось познакомится и изучить возможности СУБД DuckDB для анализа больших данных.

## Вывод

В данной работе успешно удалось изучить возможности DuckDB для анализа данных сетевой активности в сегментированной корпоративной сети.
