# data_fusion_insight
Номинация 2, Insight
* [сабмит читого решения](https://storage.yandexcloud.net/datasouls-ods/submissions/eaf6ecf1-7215-4e9f-80d0-4ee38f954259/b62c4b75/cross_v3.csv)
* [сабмит после смешивания](https://storage.yandexcloud.net/ds-ods/files/submissions/eaf6ecf1-7215-4e9f-80d0-4ee38f954259/456c5a53/bx2_d10x2_m_150.csv)

## Как добился самого высокого MRR скора на лидерборде в задаче Puzzle
Самый высокий MRR скор, при не самом высоком Precision (на последний день соревнования)

Это решение не смог имплементировать в main задаче из-за ограничения по времени в 60мин. Можно было частично имплементировать, но этим не занимался

### Гипотеза
Есть набор MCC и CAT, которые часто встречаются рядом (по времени) в пределах 2х часов. По этим сочетаниям можно угадать пары банк-ртк

### Что делал?
#### часть 1
Взял 15т. пар из трейн. Смержил transactions и clickstream по дате и user_id. Получилось примерно так:

user_id  | date | mcc_code | cat_id | transaction_time | clickstream_time | delta_time_abs_sec
-------- | -----| -------- |------  |------            |------            |------ 
1051  | 10.07.2021 | 7832  | 552    |  08:05:23        |     09:25:43     | 4820 sec
1051  | 10.07.2021  | 5200 | 170    |  16:05:40        |     23:50:40     | 27900 sec 


Отобрал записи с delta_time_abs_sec < 2 часа и группировал по mcc и cat:

mcc_code|	cat_id|	delta_time_abs_sec_mean|	count|	clients
-------- | -----| -------- |------  |------
2941 |	7832|	552	|6504|	35|	33
3132|	5200|	170|	6801|	80|	27

Сортировал по delta_time_abs_sec_mean asc и оставил строки с clients > 20 (редкие сочетания игнорировал). Получ сочетания mcc и cat, которые буду использовать как фичи в модель.

#### часть 2
Берем top 400 сочетаний (выше). Берем знакомую нам модель а-ля baseline с эмбедами:

```
# sec1 - кол-во секунд (минут) с начала суток 
bankclient_embed = transactions.pivot_table(index = ['user_id', 'dt'],
                            values=['sec1'],
                            columns=['mcc_code'],
                            aggfunc=['median']).fillna(0)
```

после джоина bankclient_embed и clickstream_embed в цикле создаем 400 фичей - разность по времени, по модулю

```
ps = bankclient_embed.merge(clickstream_embed, ...)
    
for i in range(len(top400_mcc_cat_pairs)):
    ps[mcc[i] + '_' + cat[i]] = np.sign(ps['v1_median-' + mcc[i]]) * 
                                np.sign(ps['v2_median-' + cat[i]]) * 
                                abs(ps['v1_median-' + mcc[i]] - ps['v2_median-' + cat[i]])
```
Удаляем все фичи, кроме этих 400 и в катбуст

Predict для задачи Puzzle (5k * 5k) считался около суток (после оптимизации в 2 раза быстрее)

После сабмита получил скор меньше baseline:) Но спустя месяц, заметил, что скор хоть и небольшой, но относительный MRR скор был в 2 раза выше чем на LB, все портило низкое OCR. 

![](https://github.com/timredz/data_fusion_insight/blob/main/img/cross.png)

А потом все заколосилось, догадался смешивать этот результат с другими сабмитами с высоким OCR. Брал свой обычный сабмит и просто пересортировывал порядок кандидатов.

В результате, итоговый скор подскакивал на 50%

Для сравнения:
![](https://github.com/timredz/data_fusion_insight/blob/main/img/resorted.png)

Ноутбук с решением и результатми прилагаю

P.S. Не получилось должного времени уделить, чтобы улучшить этот подход. По сути взял, что получилось первым 
  * Почему 400? -больше по памяти не проходило и долго считалось
  * Другие фичи, другие комбинации - не успел попробовать
  
#### Есть большая предиктивная сила во временной близости событий transactions и clickstream
