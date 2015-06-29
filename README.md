# PostGis-sample
В цій статті наведений приклад роботи з PostGis

## Робота з PostgreSQL:

Працювати з базами даних можна за допомогою графічної утиліти `pgAdmin` або консольної утиліти `psql`.

Нижче показана робота у консольній утиліті `psql` в Ubuntu.

Запускаємо утиліту `psql` з правами користувача `postgres`:

    psql -U postgres
    
## Створення бази данних
Створюємо БД:

    postgres=# CREATE DATABASE sample;

Перевіряємо, чи створилась БД:

    postgres=# \l
```    
                                      Список баз данных
    Имя    | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   |     Права доступа     
-----------+----------+-----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 | 
 sample    | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 | 
 template0 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 | =c/postgres          +
           |          |           |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 | =c/postgres          +
           |          |           |             |             | postgres=CTc/postgres
(4 строки)
```

Підключаємось до бази:
    
    \c sample;

Створюємо таблицю:
```
create table points(
    id serial primary key not null
);
SELECT AddGeometryColumn ('points','point',0,'POINT',2);
ALTER TABLE points ADD COLUMN description text;
```

Перевіряємо, чи правильно створилась таблиця:

    sample=# \d points
```
                               Таблица "public.points"
   Колонка   |       Тип       |                    Модификаторы                     
-------------+-----------------+-----------------------------------------------------
 id          | integer         | NOT NULL DEFAULT nextval('points_id_seq'::regclass)
 point       | geometry(Point) | 
 description | text            | 
Индексы:
    "points_pkey" PRIMARY KEY, btree (id)
```

Вставляємо декілька значень:
```
insert into points (point, description) values
     ('POINT(30.420171 50.458585)', 'Beresteyska'),
     ('POINT(30.445495 50.454605)', 'Shulyavska')
;
```
## Вибірка даних

Дивимось, що міститься в таблиці:

    select * from points;
    
```
 id |                   point                    | description 
----+--------------------------------------------+-------------
  1 | 010100000045BA9F53906B3E40D4B7CCE9B23A4940 | Beresteyska
  2 | 01010000001288D7F50B723E408ECC237F303A4940 | Shulyavska
(2 строки)
```
Гео-дані PostGIS зберігаються в базі даних у 16-річному форматі. Для перетворення їх у текст застосуємо функцію `ST_AsText`:

    select id, ST_AsText(point), description from points;
    
```
 id |         st_astext          | description 
----+----------------------------+-------------
  1 | POINT(30.420171 50.458585) | Beresteyska
  2 | POINT(30.445495 50.454605) | Shulyavska
(2 строки)
```

Знайдемо відстань від наших точок до точки (30.437051,50.455477) - проспект Перемоги, 49/2:
```
select description as from_point, ST_Distance_Sphere(point, ST_MakePoint(30.437051,50.455477)) as distance
from points;
```
```
 from_point  |    distance    
-------------+----------------
 Beresteyska | 1243.957557487
 Shulyavska  |  605.614506566
(2 строки)
```
Вставимо ще декілька рядків:
```
insert into points (point, description) values
     ('POINT(30.465968 50.450814)', 'Politechnic Institute'),
     ('POINT(30.481944 50.4625)', 'Lukianivska')
;
```
Знайдемо станції метро, що знаходяться в радіусі 1500 метрів від будинку проспект Перемоги, 49/2:
```
select id, description, ST_AsText(point) as coordinates, ST_Distance_Sphere(point, ST_MakePoint(30.437051,50.455477)) as distance
from points
where ST_Distance_Sphere(point, ST_MakePoint(30.437051,50.455477)) < 1500;
```
```
 id | description |        coordinates         |    distance    
----+-------------+----------------------------+----------------
  1 | Beresteyska | POINT(30.420171 50.458585) | 1243.957557487
  2 | Shulyavska  | POINT(30.445495 50.454605) |  605.614506566
(2 строки)
```
І в радіусі 4000 метрів:
```
select id, description, ST_AsText(point) as coordinates, ST_Distance_Sphere(point, ST_MakePoint(30.437051,50.455477)) as distance
from points
where ST_Distance_Sphere(point, ST_MakePoint(30.437051,50.455477)) < 4000;
```
```
 id |      description      |        coordinates         |    distance    
----+-----------------------+----------------------------+----------------
  1 | Beresteyska           | POINT(30.420171 50.458585) | 1243.957557487
  2 | Shulyavska            | POINT(30.445495 50.454605) |  605.614506566
  3 | Politechnic Institute | POINT(30.465968 50.450814) | 2111.930342163
  4 | Lukianivska           | POINT(30.481944 50.4625)   | 3272.524360151
(4 строки)
```
Пошук точок в полігоні:
```
select id, description, ST_AsText(point) as coordinates
from points
where ST_Contains(ST_MakePolygon(ST_GeomFromText('LINESTRING(30.482056 50.466716, 30.455421 50.447533, 30.497939 50.437833, 30.482056 50.466716)')), point) = TRUE;
```
```
 id |      description      |        coordinates         
----+-----------------------+----------------------------
  3 | Politechnic Institute | POINT(30.465968 50.450814)
  4 | Lukianivska           | POINT(30.481944 50.4625)
(2 строки)
```

## Geography

```
CREATE TABLE geo_points ( 
    id SERIAL PRIMARY KEY,
    point GEOGRAPHY(POINT,4326),
    description TEXT   
  );

INSERT INTO geo_points (point, description) VALUES (ST_GeographyFromText('SRID=4326;POINT(30.420171 50.458585)'), 'Beresteyska' ),
						   (ST_GeographyFromText('SRID=4326;POINT(30.445495 50.454605)'), 'Shulyavska' ),
						   (ST_GeographyFromText('SRID=4326;POINT(30.465968 50.450814)'), 'Politechnic Institute' ),
						   (ST_GeographyFromText('SRID=4326;POINT(30.481944 50.4625)'), 'Lukianivska' );

select id, ST_AsText(point), description from geo_points;

select id, description, ST_AsText(point) as coordinates, ST_Distance(point, ST_GeogFromText('SRID=4326;POINT(30.437051 50.455477)')) as distance
from geo_points
where ST_Distance(point, ST_GeogFromText('SRID=4326;POINT(30.437051 50.455477)')) < 1500;
```

## Корисні посилання
* [Introduction to PostGIS][1]
* Книга **PostGIS in Action**

[1]: http://workshops.boundlessgeo.com/postgis-intro/index.html
