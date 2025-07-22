# SQL-
## Итоговый учебный проект 

Во время обучения был выполнен итоговый проект по теме «SQL и получение данных».

- Цель проекта: выполнение запросов, согласно заданиям (9 задач)

- Оценка проекта: для зачета необходимо набрать минимум 130 баллов максимум 200 баллов

- Достижение цели: 195 баллов.

<img width="370" height="107" alt="image" src="https://github.com/user-attachments/assets/9c4968a4-d355-4e77-8c06-d6545d499830" />


## Выполненный проект:  

[Описание базы данных](https://edu.postgrespro.ru/bookings.pdf)


## №1-- Выведите названия самолетов, которые имеют менее 50 посадочных мест

<img width="446" height="89" alt="image" src="https://github.com/user-attachments/assets/534aae60-2a9e-4d3d-ac21-025c9089f11f" />


## №2 --Выведите процентное изменение ежемесячной суммы бронирования билетов, окгругленной до сотых

<img width="649" height="88" alt="image" src="https://github.com/user-attachments/assets/b8ed2fb4-53f4-4725-a4f7-984e7be46af3" />


## №3 -- Выведите названия самолетов без бизнес-класса. Используйте в решении функцию  array-agg


SELECT  a.model, array_position(array_agg (fare_conditions), 'Business')
from aircrafts a
join seats s on s.aircraft_code = a.aircraft_code
group by a.model
having array_position(array_agg (fare_conditions), 'Business')is null


№4 Выведите накопительный итог количества мест в самолётах по каждому аэропорту на каждый день. 
Учтите только те самолеты, которые летали пустыми и только те дни, когда из одного аэропорта 
вылетело более одного такого самолёта.
Выведите в результат код аэропорта, дату вылета, количество пустых мест и накопительный итог.
 
explain analyze

    with cte as(
    select count(seat_no), aircraft_code
	from seats
	group by aircraft_code),
        cte1 as(
		select fv.departure_airport, actual_departure, aircraft_code,
		count(*) over (partition by departure_airport, actual_departure::date)
		from flights_v fv 
		left join boarding_passes bp  on bp.flight_id = fv.flight_id  
		where  boarding_no is null and actual_departure is not null)
    select departure_airport as "Код аэропорта", actual_departure as "Фактическое время вылета",cte.count as "Количество мест",
	sum(cte.count) over (partition by departure_airport, actual_departure::date order by actual_departure)as "Накопительный итог"
	from cte
    join cte1 on cte.aircraft_code = cte1.aircraft_code
    where cte1.count >'1'

№5 Найдите процентное соотношение перелётов по маршрутам от общего количества перелётов. 
Выведите в результат названия аэропортов и процентное отношение.
Используйте в решении оконную функцию.

explain analyze

select distinct departure_airport_name , arrival_airport_name,
round( 100.*count (flight_no)over ( partition by flight_no)/count(flight_id) over (),2)as "% соотношение "
from flights_v fv 



№6 Выведите количество пассажиров по каждому коду сотового оператора. 
Код оператора – это три символа после +7


explain analyze

select count(passenger_name)as "Количество пассажиров" , 
substring(contact_data ->>'phone',3, 3) as "код оператора"
from tickets 
group by substring(contact_data ->>'phone',3,3)


№7 Классифицируйте финансовые обороты (сумму стоимости перелетов) по маршрутам:
●	до 50 млн – low
●	от 50 млн включительно до 150 млн – middle
●	от 150 млн включительно – high
Выведите в результат количество маршрутов в каждом полученном классе.

select fin_class,count(flight_no)"Количество маршрутов"
from (
select flight_no,sum(amount),
    case 
	     when sum(amount) < 50000000 then 'low'
    	 when sum(amount) >= 50000000 and sum(amount) < 150000000 then 'middle'
    	 when sum(amount) >= 150000000 then 'High'
    end fin_class
from flights_v fv
 join ticket_flights tf on fv.flight_id = tf.flight_id
group by flight_no)
group by fin_class


№8 Вычислите медиану стоимости перелетов, медиану стоимости бронирования 
и отношение медианы бронирования к медиане стоимости перелетов, результат округлите до сотых.

select *,
  round(((select percentile_cont(0.5)within group (order by b.total_amount)as "Медиана бронирования" 
  from bookings b)/
   (select percentile_cont(0.5)within group (order by tf.amount)as "Медиана перелетов" 
   from ticket_flights tf ))::numeric,2)
from (select percentile_cont(0.5)within group (order by tf.amount)as "Медиана перелетов"from ticket_flights tf ) t,
	 (select percentile_cont(0.5)within group (order by b.total_amount)as "Медиана бронирования" from bookings b)

№9 Найдите значение минимальной стоимости одного километра полёта для пассажира.
Для этого определите расстояние между аэропортами и учтите стоимость перелета.

Для поиска расстояния между двумя точками на поверхности Земли используйте дополнительный модуль earthdistance. 
Для работы данного модуля нужно установить ещё один модуль – cube.
Важно: 
●	Установка дополнительных модулей происходит через оператор CREATE EXTENSION название_модуля.
●	В облачной базе данных модули уже установлены.
●	Функция earth_distance возвращает результат в метрах.

CREATE extension earthdistance

CREATE extension cube



select min ("min_cost_km")as "Минимальная стоимость 1 км"
	from (select sum(amount)/
	(earth_distance(ll_to_earth(a.latitude,a.longitude), ll_to_earth(a2.latitude,a2.longitude))/1000) /
	count(tf.ticket_no) as "min_cost_km"
from flights f
join airports a on a.airport_code = f.departure_airport 
join airports a2 on a2.airport_code = f.arrival_airport
join ticket_flights tf on f.flight_id = tf.flight_id
group by f.flight_no,a.latitude,a.longitude,a2.latitude,a2.longitude)

  
