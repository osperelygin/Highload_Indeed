# Highload Indeed

## Содержание
* ### [1. Тема, целевая аудитория](#1)
* ### [2. Расчет нагрузки](#2)
* ### [3. Глобальная балансировка нагрузки](#3)
* ### [ Список источников ](#sources)

## 1. Тема и целевая аудитория <a name="1"></a>

Indeed — всемирный сервис по поиску работы.

### Целевая аудитория

Трудоспособная часть населения, HR-специалисты

Распределение пользователей по странам:

![Распределение пользователей по странам](images/counry_consumer_distribution.png)

Демография аудитории:

![Демография аудитории](images/demographic_distribution.png)

### Продуктовый функционал
* Поиск вакансий
* Поиск соискателей
* Отклик на вакансию
* Создание вакансий
* Создание/обновление резюме

Основной технической сложностью сервиса является поиск по большому количеству резюме соискателей (245M) и вакансий компаний (533K) в БД. При поиске сотруднкиов, как правило, HR-специалисты используют следующие фильтры: город, режим работы, тип занятости, профессия, должность, опыт работы, желаемый уровень дохода. Соотвественно необходимо обеспечить эффективный поиск в БД резюме соискателей с учетом применения различных фильтров.

### Аналоги
* [myworkdayjobs.com](https://workday.wd5.myworkdayjobs.com/Workday)
* [www.linkedin.com](https://www.linkedin.com)
* [hh.ru](https://hh.ru/)

## 2. Расчет нагрузки <a name="2"></a>

### Продуктовые метрики
* DAU: 21M [^1], [^2]  
* MAU: 350M [^1], [^2]
* Среднее время на сайте: 00:06:32 [^1], [^2] 
* Количество резюме: 245M [^3]
* Количество доступных вакансий: 533K [^3]
* Количество компаний: 14.6K [^4]
* Добавление новых вакансий за месяц: 9.2М [^5]
* Создание/обновление резюме за месяц: 8М [^6] 

### Технические метрики

Для расчета технических метрик необходимо определить количество запросов по поиску вакансий и соискателей. Поскольку indeed не предоставляет подобные данные в публичный доступ, воспользуемся значениями метрик аналогичного сервиса популярного в СНГ - HeadHunter. Согласно данным HeadHunter за второе полугодие 2022 года было совершенно около 4.651 млрд запросов по поиску работы [^8], при этом MAU HH = 27.6M [^9]. Учитывая отношение MAU этих двух сервисов, можем оценить количество запросов по поиску вакансий в сервисе Indeed.

Оценка количества запросов вакансий в день:

> 350 / 27.6 * 4 651 / 6 / 30 / = 328М

Аналогичным образом получим оценку количества откликов - по данным HeadHunter за март 2022 года соискатели отправили 41 328 829 откликов

Оценка количества откликов в день:

> 350 / 27.6 * 41 328 829 / 30 = 17 469 916

#### Размер хранилища

Средний размер харанилища соискателя:

| Хранимые данные | Размер |
| --- | --- |
| Персональные данные | `1 КБ` |
| Резюме | `32 KБ` |

Данные пользователей:

> 245 000 000 * (32 KB + 1 KB) = 7710 GB

Средний размер харанилища вакансии:

| Хранимые данные | Размер |
| --- | --- |
| Описание | `32 КБ` |

Данные вакансий:

> 533 000 * 32 KB = 16 GB

Суммарный объем хранилища

> 7710 GB + 16 GB = 7726 GB = 7.55 TB

#### Расчет RPS:

Запросы соискателей по поиску вакансий:

> 328 000 000 / 86400 = 3796 RPS

Отклики соискателей по вакансиям:

> 17 469 916 / 86400 = 202 RPS

За месяц обновляется/создается 8M резюме:

> 8 000 000 / 30 / 86400 = 3 RPS

За месяц добавляется 9.2M резюме:

> 9 200 000 / 30 / 86400 = 4 RPS

Суммарный RPS:

> 3796 + 202 + 3 + 4 = 4005 RPS

#### Расчет сетевого трафика:

С помощью инструментов разработчикам было определено среднее значение запрашиваемой страницы с листингом вакансий - 168 kB

Средний трафик в день на запросы вакансий:

> 328 000 000 * 168 kB / 86400 = 326 562 kBps = 4.87 Gbps

## 3. Глобальная балансировка нагузки <a name="3"></a>

### Обоснование расположения ДЦ

| Страна            | Процент пользователей |   RPS    |
|:------------------|:----------------------|:---------|
| США               |          44.53        |   899    |
| Канада            |           9           |   182    |
| Великобритания    |         8.21          |   166    |
| Франция           |         3.82          |   77     |
| Индия             |         3.51          |   71     |

Больше половины аудитории сервиса находится в Северной Америки, всвязи с этим данный регион подвергается наибольшей нагрузке. С учетом плотности населения США выберем: Нью-Йорк, Лос-Анджелес, Маями и Сиэтл. В Канаде целесообразно расположить ДЦ в городе с наибольшим населением - Торонто. Аудитория сервиса, проживающая в западной Канаде будет использовать ДЦ в Сиэтле из-за близкого географического положения. Для наглядности приведена карта с плотностью населения Северной Америки 
![Heatmap North America](images/heatmap_north_america.png)

Для европейской части аудитории разумно будет расположить один ДЦ в немецком Франкфурте, а для южной азиатской расположим ДЦ в индийском Мумбаи.

### Схема DNS балансировки

Geo-based DNS разумно использовать для распределенной географической балансировки на уровне стран. 

### Схема Anycast балансировки

Поскольку общий пул резолвер может использоваться на достаточно большую географическую зону, кроме Geo-based DNS для выбора наиболее географически близкого к пользователю сервера будем также применять технологию BGP anycast, позволяющую после резолва DNS в IP-адреса влиять на выбор ДЦ, на который будет сделан запрос. 

## Список использованных источников: <a name="sources"></a>

[^1]: [SimilarWeb indeed.com](https://www.similarweb.com/website/indeed.com/)
[^2]: [HypeStat indeed.com](https://hypestat.com/info/indeed.com)
[^3]: [About Indeed](https://www.indeed.com/about/)
[^4]: [Indeed Statistics and Revenue 2023](https://homejobshub.com/indeed-revenue-and-usage-statistics/#Indeed-Company-Information)
[^5]: [Free vs Sponsored Jobs on Indeed](https://www.indeed.com/hire/resources/howtohub/free-vs-sponsored-jobs-on-indeed#:~:text=Since%209.2%20million%20jobs%20are,postings%20can%20lose%20visibility%20quickly.)
[^6]: [How to Search for Candidates on Indeed](https://www.indeed.com/hire/resources/howtohub/how-to-consistently-attract-and-filter-quality-applicants#:~:text=New%20talent%20added%20every%20day,alerts%20for%20new%20resume%20matches.)
[^7]: [empty](#)
[^8]: [Думать, как соискатель: как ищут работу пользователи hh.ru](https://hh.ru/article/31506)
[^9]: [About us. Main Page]((https://investor.hh.ru/))
[^10]: [Время — деньги: как разбор откликов позволяет экономить время и ресурсы рекрутера](https://hh.ru/article/29100)


<!-- * https://kids.britannica.com/students/assembly/view/166536
* https://www.indeed.com/lead/timing-matters-in-the-job-search
* https://www.indeed.com/lead/indeed-delivers-65-percent-online-hires
* https://github.com/hiring-lab/job_postings_tracker?tab=readme-ov-file
* https://github.com/hiring-lab/indeed-wage-tracker
* https://hh.ru/article/31506
* https://www.iapply.ai/blogs/what-does-many-applicants-mean-on-indeed.html#:~:text=How%20Many%20Applications%20Does%20a,likely%20have%20an%20interview%20scheduled.-->
