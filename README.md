# data_fusion_insight
Номинация 2, Insight

## Как добился самого высокого MRR скора на лидерборде в задаче Puzzle
На последний день соревнования у моего решения самый высокий MRR скор, при не самом высоком OCR. 

Это решение не смог имплементировать в main задаче из-за ограничения по времени в 60мин. Можно было частично имплементировать, но этим не занимался

### Гипотеза
Есть набор MCC и CAT, которые часто встречаются рядом (по времени) в пределах 2х часов. По этим сочетаниям можно угадать пары банк-ртк

### Что делал?
#### часть 1
Взял 15т. пар из трейн. Смержил transactions и clickstream по дате и user_id. Получилось примерно такое:

user_id  | date | mcc_code | cat_id | transaction_time | clickstream_time | delta_time_abs_sec
-------- | -----| -------- |------  |------            |------            |------ 
1051  | 10.07.2021 | 7832  | 552    |  08:05:23        |     09:25:43     | 4820 sec
1051  | 10.07.2021  | 5200 | 170    |  16:05:40        |     23:50:40     | 27900 sec 


Отбираем записи с delta_time_abs_sec < 2 часа и группируем по mcc и cat:

mcc_code|	cat_id|	delta_time_abs_sec_mean|	count|	clients
-------- | -----| -------- |------  |------
2941 |	7832|	552	|6504|	35|	33
3132|	5200|	170|	6801|	80|	27

Сортируем по delta_time_abs_sec_mean asc и оставляем строки с clients > 20 (редкие сочетания игнорируем). Получ сочетания mcc и cat, которые буду использовать как фичи в модель.

#### часть 2
Берем top 400 сочетаний (выше). Берем знакомую нам модель а-ля baseline с эмбедами:

```
# sec1 - кол-во секунд с начала суток 
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

После сабмита навернулись слезы, когда скор был меньше baseline:)

![](https://github.com/timredz/data_fusion_insight/blob/main/img/cross.png)

> Follow your heart.

+ Item A
+ Item B
    + Item B 1
    + Item B 2
    + Item B 3
+ Item C
    * Item C 1
    * Item C 2
    * Item C 3


First Header  | Second Header
------------- | -------------
Content Cell  | Content Cell
Content Cell  | Content Cell 
