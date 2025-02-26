# Задание 1
```sql
with p1 as 
(
select distinct to_char(u.date_joined,'YYYY-MM') as yw, 
ue.user_id, 
date_part('day',ue.entry_at-u.date_joined) as days_entry 
from users u 
inner join userentry ue
on ue.user_id = u.id 
where to_char(u.date_joined,'YYYY') = '2022'
),
p2 as
(
select p1.yw, p1.user_id,
max(case when p1.days_entry >= 0 then 1 else 0 end) as day0,
max(case when p1.days_entry >= 1 then 1 else 0 end) as day1,
max(case when p1.days_entry >= 3 then 1 else 0 end) as day3,
max(case when p1.days_entry >= 7 then 1 else 0 end) as day7,
max(case when p1.days_entry >= 14 then 1 else 0 end) as day14,
max(case when p1.days_entry >= 30 then 1 else 0 end) as day30,
max(case when p1.days_entry >= 60 then 1 else 0 end) as day60,
max(case when p1.days_entry >= 90 then 1 else 0 end) as day90 
from p1 
group by p1.yw, p1.user_id 
),
p3 as
(
select p1.yw, 
count(case when p1.days_entry >= 0 then user_id end) as day0,
count(case when p1.days_entry >= 1 then user_id end) as day1,
count(case when p1.days_entry >= 3 then user_id end) as day3,
count(case when p1.days_entry >= 7 then user_id end) as day7,
count(case when p1.days_entry >= 14 then user_id end) as day14,
count(case when p1.days_entry >= 30 then user_id end) as day30,
count(case when p1.days_entry >= 60 then user_id end) as day60,
count(case when p1.days_entry >= 90 then user_id end) as day90 
from p1 
group by p1.yw, p1.user_id 
)
select p2.yw, 
round(sum(p2.day0)*100.00/count(p2.user_id),2) as day0,
round(sum(p2.day1)*100.00/count(p2.user_id),2) as day1,
round(sum(p2.day3)*100.00/count(p2.user_id),2) as day3,
round(sum(p2.day7)*100.00/count(p2.user_id),2) as day7,
round(sum(p2.day14)*100.00/count(p2.user_id),2) as day14, 
round(sum(p2.day30)*100.00/count(p2.user_id),2) as day30,
round(sum(p2.day60)*100.00/count(p2.user_id),2) as day60,
round(sum(p2.day90)*100.00/count(p2.user_id),2) as day90 
from p2 
group by p2.yw
order by p2.yw
```
Выводы: Основная масса людей перестала пользоваться сайтом на следующий день после регистрации, второе падение посещаемости после 30 дней. Потенциальные причины:
1. Возникли проблемы с решением задач или пониманием работы сайта
2. Регистрация была необходима для прохождения курсов компаний, с которыми сотрудничает платформа, и само решение задач было отложено или не выполнено как необязательное.

# Задание 2

```sql
with p1 as( 
select t.user_id,
sum(case when t.type_id in (1, 23, 24, 25, 26, 27, 28, 30) then t.value else 0 end) as balance_out,
sum(case when t.type_id not in (1, 23, 24, 25, 26, 27, 28, 30) then t.value else 0 end) as balance_in
from transaction t 
join transactiontype t1
on t1.type = t.type_id 
group by t.user_id
)
select 
round(avg(balance_out), 2) as balance_out_avg, 
round(avg(balance_in), 2) as balance_in_avg, 
round(avg(balance_in - balance_out), 2) as balance_avg, 
percentile_cont(0.5) within group(order by (balance_in - balance_out)) as balance_median 
from p1
```
Выводы: Основная масса пользователей мало начисляет и мало списывает (медиана намного ниже среднего). Но активное меньшинство много начисляет (возможно, во многом за счет бонусов).

# Задание 3
```sql
-- Сколько в среднем пользователь решает задач 
with p1 as 
(
select problem_id, user_id from coderun 
union all 
select problem_id, user_id from codesubmit 
), 
p2 as
(
select user_id, count(distinct problem_id) as problems 
from p1
group by user_id 
)
select round(avg(problems),2) 
from p2

-- Сколько в среднем пользователь проходит тестов 
with p1 as 
(
select user_id, count(distinct test_id) as tests 
from teststart
group by user_id 
)
select round(avg(tests),2) 
from p1

--Сколько в среднем пользователь делает попыток для решения 1 задачи
with p1 as 
(
select problem_id, user_id from coderun 
union all 
select problem_id, user_id from codesubmit 
), 
p2 as
(
select user_id, problem_id, count(problem_id) as tryouts 
from p1
group by user_id, problem_id 
)
select round(avg(tryouts),2) 
from p2

-- Сколько в среднем пользователь делает попыток для прохождения 1 теста
with p1 as 
(
select user_id, test_id, count(test_id) as tests 
from teststart
group by user_id, test_id 
)
select round(avg(tests),2) 
from p1

-- Какая доля от общего числа пользователей решала хотя бы одну задачу или начинала проходить хотя бы один тест
with p1 as 
(
select user_id from coderun
union all
select user_id from codesubmit 
union all
select user_id from teststart 
), 
p2 as
(
select distinct u.id as user_id, p1.user_id as user_try_id
from users u 
left join p1 
on p1.user_id = u.id 
)
select 
round(sum(case when p2.user_try_id is not null then 1 else 0 end)*100.00/count(u.id),2) as part
from users u
left join p2 on p2.user_id = u.id

/*
1.Сколько человек открывало задачи за кодкоины
2.Сколько человек открывало тесты за кодкоины
3.Сколько человек открывало подсказки за кодкоины
4.Сколько человек открывало решения за кодкоины
5.Сколько подсказок/тестов/задач/решений было открыто за кодкоины (если задача/... открыта разными людьми, то это считаем разными фактами открытия)
6.Сколько человек покупало хотя бы что-то из вышеперечисленного
7.Сколько человек всего имеют хотя бы 1 транзакцию, пусть даже только начисление
*/

with p1 as 
(
select distinct t.user_id, t.type_id 
from transaction t 
join transactiontype t1
on t.type_id = t1.type
)
select 
count(distinct case when type_id = 23 then user_id end) as cnt_problems, -- 1 
count(distinct case when type_id in (26, 27) then user_id end) as cnt_tests, -- 2 
count(distinct case when type_id = 24 then user_id end) as cnt_clues, -- 3 
count(distinct case when type_id = 25 then user_id end) as cnt_decisions, -- 4 
count(distinct case when type_id in (23, 24, 25, 26, 27) then user_id end) as cnt_all, -- 5 
count(distinct user_id) as cnt_user_any_transact, -- 6 
count(case when type_id in (23, 24, 25, 26, 27) then 'OK' end) as cnt_anytrans --7 
from p1 
```

Выводы: Пользователи делают довольно большое число попыток (5,75) для решения задач, что в свою очередь влияет на высокое число покупок за коины. 
Популярность задач выше тестов (в системе больше задач, чем тестов), но пользователей, купивших тестов больше, значит и важность их выше. 
Нужно увеличивать количество тестов (значит и пользователей, которые хотят их купить)

# Дополнительное задание

Мне кажется в первые 3 дня резко падает посещаемость платформы (примерно в 2 раза) и нужно оценить,  в какой именно день идет падение. Для этого я предлагаю посчитать в разрезе дней 0-2:
- количество задач, решенных в эти дни (попытки)
- количество задач, решенных в эти дни (решения)
- количество тестов, пройденных в эти дни (раздельно по дням)
  Потому что попытки и решения задач, а также прохождение тестов могут делать с разными целями. Например, тесты нужны для прохождения собеседований, а ршенные задачи нужны для прохождения курсов.
```sql
-- количество задач, решенных в эти дни (попытки)
with p1 as
(
select c.user_id, u.date_joined, c.created_at, 
(date(c.created_at)-date(u.date_joined)) as days_diff 
from coderun c 
join users u 
on u.id = c.user_id 
where (date(c.created_at)-date(u.date_joined)) in (0, 1, 2)
)
select days_diff, count(user_id) as coderuns 
from p1 
group by days_diff
order by days_diff

-- количество задач, решенных в эти дни (решения)
with p1 as
(
select c.user_id, u.date_joined, c.created_at, 
(date(c.created_at)-date(u.date_joined)) as days_diff 
from codesubmit c 
join users u 
on u.id = c.user_id 
where (date(c.created_at)-date(u.date_joined)) in (0, 1, 2)
)
select days_diff, count(user_id) as coderuns 
from p1 
group by days_diff
order by days_diff

-- количество тестов, пройденных в эти дни (раздельно по дням)
with p1 as
(
select c.user_id, u.date_joined, c.created_at, 
(date(c.created_at)-date(u.date_joined)) as days_diff 
from teststart c 
join users u 
on u.id = c.user_id 
where (date(c.created_at)-date(u.date_joined)) in (0, 1, 2)
)
select days_diff, count(user_id) as coderuns 
from p1 
group by days_diff
order by days_diff 
```
Выводы: Существенное снижение активности уже на следующий день после регистрации по всем направлениям.

## Итоговые выводы по смене модели монетизации
Думаю, можно в первые дни увеличить начисления монет за решения, задач, прохождения тестов (для того, чтобы пользователь адаптировался к платформе) и уменьшать постепенно в течение, например, одного месяца. Но предлагать скидку на длительную подписку для клиентов, которые активны более 30 дней. 
Также желательно создавать опрос о целях регистрации на платформе. Например, при решении задач для разового/пробного семинара нет необходимости использовать платформу более 1-2 дней.
   
