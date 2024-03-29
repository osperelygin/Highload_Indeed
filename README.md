# Highload Indeed

## Содержание
* ### [1. Тема, целевая аудитория](#1)
* ### [2. Расчет нагрузки](#2)
* ### [3. Глобальная балансировка нагрузки](#3)
* ### [4. Локальная балансировка нагрузки](#4)
* ### [5. Логическая схема БД](#5)
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

### Размер хранилища

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

### Расчет RPS:

Для расчета RPS необходимо определить количество запросов по поиску вакансий и поиску соискателей, а также запросов по откликам на вакансию. Поскольку indeed не предоставляет подобные данные в публичный доступ, воспользуемся значениями метрик аналогичного сервиса популярного в СНГ - HeadHunter, MAU которого равен 27.6M [^8]. Учитывая отношение MAU этих двух сервисов, можем оценить значения требуемых метрик сервиса Indeed. 

Согласно данным HeadHunter за второе полугодие 2022 года было совершенно около 4.651 млрд запросов по поиску работы [^7]

Оценка количества запросов по поиску вакансий в день:

> 350 / 27.6 * 4 651 / 6 / 30 = 328М

Запросы по поиску вакансий:

> 328 000 000 / 86400 = 3796 RPS

Согласно данным HeadHunter за март 2022 года соискатели отправили 41 328 829 откликов [^9]

Оценка количества откликов в день:

> 350 / 27.6 * 41 328 829 / 30 = 17 469 916

Отклики соискателей по вакансиям:

> 17 469 916 / 86400 = 202 RPS

Оценку количества запросов по поиску резюме в день получим исходя из предположения, что количество запросов по поиску резюме меньше количества запросов по поиску вакансий в M раз, M - соотношение количества активных резюме к количеству доступных вакансий на рынке. Согласно данным HeadHunter данное соотоноешние за февраль 2024 года составляет 3.4. [^10]

Оценка количества запросов по поиску резюме в день:

> 328М / 3.4 = 96.5M

Запросы по поиску резюме:

> 96 500 000 / 86400 = 1167 RPS

За месяц обновляется/создается 8M резюме:

> 8 000 000 / 30 / 86400 = 3 RPS

За месяц добавляется 9.2M вакансий:

> 9 200 000 / 30 / 86400 = 4 RPS

| Тип запроса | RPS |
| --- | --- |
| Поиск вакансий | `3796` |
| Поиск резюме | `1167` |
| Отклики | `202` |
| Добавление вакансий | `4` |
| Создание/обновление резюме | `3` |
| Суммарное значение | `5172` |

### Расчет сетевого трафика:

С помощью инструментов разработчикам было определено среднее значение запрашиваемой страницы с листингом вакансий - 168 kB и листингом резюме - 172 kB

Средний трафик в день на запросы поиска вакансий:

> 3796 RPS * 168 kB = 326 562 kBps = 4.87 Gbps

Средний трафик в день на запросы поиска резюме:

> 1167 * 172 kB = 326 562 kBps = 1.53 Gbps

Трафик на создание/обновление резюме, добавление вакансий и откликов на вакансиии является принебрежимо малым, поэтому его в расчет не включаем.

| Тип запроса | Средний трафик в день, Gbps |
| --- | --- |
| Поиск вакансий | `4.87` |
| Поиск резюме | `1.53` |
| Суммарное значение | `6.4` |

## 3. Глобальная балансировка нагузки <a name="3"></a>

### Обоснование расположения ДЦ

| Страна            | Процент пользователей |   RPS    |
|:------------------|:----------------------|:---------|
| США               |        `44.53`        |   2303   |
| Канада            |           `9`         |   465    |
| Великобритания    |         `8.21`        |   424    |
| Франция           |         `3.82`        |   198    |
| Индия             |         `3.51`        |   182    |

Больше половины аудитории сервиса находится в Северной Америки, в связи с этим данный регион подвергается наибольшей нагрузке. С учетом плотности населения США выберем: Нью-Йорк, Лос-Анджелес, Маями и Сиэтл. В Канаде целесообразно расположить ДЦ в городе с наибольшим населением - Торонто. Аудитория сервиса, проживающая в западной Канаде будет использовать ДЦ в Сиэтле из-за близкого географического положения. Для наглядности приведена карта с плотностью населения Северной Америки 

![image](https://github.com/osperelygin/Highload_Indeed/assets/115468412/a82325d7-b6eb-47a4-afde-6ca73c2542f0)

Для европейской части аудитории разумно будет расположить один ДЦ в немецком Франкфурте, а для южной азиатской - в индийском Мумбаи.

### Схема DNS балансировки

Geo-based DNS разумно использовать для распределенной географической балансировки на уровне стран. 

### Схема Anycast балансировки

Поскольку общий пул резолвер может использоваться на достаточно большую географическую зону, кроме Geo-based DNS для выбора наиболее географически близкого к пользователю сервера будем также применять технологию BGP anycast, позволяющую после резолва DNS в IP-адреса влиять на выбор ДЦ, на который будет сделан запрос. 

## 4. Локальная балансировка нагузки <a name="4"></a>

Для балансировки нагрузки внутри ДЦ и обеспечения автоматического горизонтального масштабирования будем использовать систему оркестрации kubernetes.

### Схема балансировки для входящих запросов

Для баланисировки на сетевом уровне в каждом ДЦ будет расположен Linux Virtual Server(LVS), реализованный с использованием технологии туннелирования IP. При таком подходе не будет происходить перегрузки балансировщика по сетевой карте и CPU, тк исходящий из ДЦ трафик не проходит через балансировщик. L4-балансировщик будет напралять трафик на ингресс котроллеры, на которых будет происходить терминация ssl трафика, для уменьшия оверхеда на установления соединения в доверенной сети k8s. Ингресс контроллеры в соотвествии с указанными правилами распределяют нагрузки на сервисы k8s. Существует множество реализаций ингресс контроллера, выберем nginx ingress controller, тк он регулярно поддерживается и имеет большое сообщество. 

### Схема балансировки для межсервисных запросов

Балансировка нагрузки межсервисных запросов является нативной в k8s, за это отвечает компонет kube-proxy. На данный момент существует две наиболее популярные модели kube-proxy - iptables и IPVS. Профит от использования второй модели можно получить при наличии масштабах, значительно превышающих 1000 сервисов [^11], что не предусматривает MVP проекта. Для взаимодействия микросервисов разумно использовать headless service, который позволяет совершать запросы по IP-адреса подов напрямую, без оверхеда на определение сluster IP сервиса и последующее балансировки запроса.   

### Схема отказоустойчивости и масштабирования

Для обеспечения отказоустойчивости и автоматического горизонтального масштабирования будем использовать Horizontal Pod Autoscaling - абстракцию, позволяющую изменить количество подов в реплика сете в зависимости от значений настроенных метрик.  

## 5. Логическая схема БД <a name="5"></a>

## Список использованных источников: <a name="sources"></a>

[^1]: [SimilarWeb indeed.com](https://www.similarweb.com/website/indeed.com/)
[^2]: [HypeStat indeed.com](https://hypestat.com/info/indeed.com)
[^3]: [About Indeed](https://www.indeed.com/about/)
[^4]: [Indeed Statistics and Revenue 2023](https://homejobshub.com/indeed-revenue-and-usage-statistics/#Indeed-Company-Information)
[^5]: [Free vs Sponsored Jobs on Indeed](https://www.indeed.com/hire/resources/howtohub/free-vs-sponsored-jobs-on-indeed#:~:text=Since%209.2%20million%20jobs%20are,postings%20can%20lose%20visibility%20quickly.)
[^6]: [How to Search for Candidates on Indeed](https://www.indeed.com/hire/resources/howtohub/how-to-consistently-attract-and-filter-quality-applicants#:~:text=New%20talent%20added%20every%20day,alerts%20for%20new%20resume%20matches.)
[^7]: [Думать, как соискатель: как ищут работу пользователи hh.ru](https://hh.ru/article/31506)
[^8]: [About us. Main Page](https://investor.hh.ru/)
[^9]: [Время — деньги: как разбор откликов позволяет экономить время и ресурсы рекрутера](https://hh.ru/article/29100)
[^10]: [hh Статистика](https://stats.hh.ru/)
[^11]: [Comparing kube-proxy modes: iptables or IPVS?](https://www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs/)


<!-- * https://kids.britannica.com/students/assembly/view/166536
* https://www.indeed.com/lead/timing-matters-in-the-job-search
* https://www.indeed.com/lead/indeed-delivers-65-percent-online-hires
* https://github.com/hiring-lab/job_postings_tracker?tab=readme-ov-file
* https://github.com/hiring-lab/indeed-wage-tracker
* https://hh.ru/article/31506
* https://www.iapply.ai/blogs/what-does-many-applicants-mean-on-indeed.html#:~:text=How%20Many%20Applications%20Does%20a,likely%20have%20an%20interview%20scheduled.-->
