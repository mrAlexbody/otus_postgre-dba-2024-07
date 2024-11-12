# Домашнее задание 11
## Работа с индексами

### Цель:
* знать и уметь применять основные виды индексов PostgreSQL
* строить и анализировать план выполнения запроса
* уметь оптимизировать запросы для с использованием индексов

### Описание/Пошаговая инструкция выполнения домашнего задания:
Создать индексы на БД, которые ускорят доступ к данным.

**В данном задании тренируются навыки:**
* определения узких мест
* написания запросов для создания индекса
* оптимизации

**Необходимо:**
1. Создать индекс к какой-либо из таблиц вашей БД
2. Прислать текстом результат команды explain,
3. в которой используется данный индекс
4. Реализовать индекс для полнотекстового поиска
5. Реализовать индекс на часть таблицы или индекс
6. на поле с функцией
7. Создать индекс на несколько полей
8. Написать комментарии к каждому из индексов
9. Описать что и как делали и с какими проблемами
10. столкнулись
---

#### Создадим тестовую БД: 
```postgresql
postgres=# \c otus
otus=# CREATE TABLE "user" (
        id bigint,
        login varchar(255),
        surname varchar(255),
        name varchar(255),
        lastname varchar(255),
        CONSTRAINT user_pk PRIMARY KEY (id)
);
CREATE TABLE
otus=# \dt
            Список отношений
 Схема  |   Имя    |   Тип   | Владелец
--------+----------+---------+----------
 public | user     | таблица | postgres
(1 строки)

otus=# INSERT INTO "user" SELECT id, MD5(random()::TEXT)::TEXT, MD5(random()::TEXT)::TEXT, MD5(random()::TEXT)::TEXT, MD5(random()::TEXT)::TEXT FROM generate_series(100, 200) AS id;
INSERT 0 101

otus=# SELECT * FROM public.user LIMIT 20;
 id  |              login               |             surname              |               name               |             lastname
-----+----------------------------------+----------------------------------+----------------------------------+----------------------------------
 100 | adf991ed6106ff182647b8a9dd33620d | da74b734e0ccb0c7affb05ab185e5d88 | ab883df8d58f3c3324c2172d687a378f | dcd5d52541ced5fccf092431231f26fc
 101 | 89091b64d454e07c974dbcdad17fcefb | 37c0282a355271815431a053061582db | 2074a7bf744ebec28291b13f0124163c | 384b102ec426f48b7f5c323593d74444
 102 | 99242e1c2e4bdf72c9c574ff9e58f0e6 | 7bf7851836b084db34a18e7c15a9e903 | 7ef4fd35b1ec0e05c12bd14b975f9549 | d51059a0fd13415bf32462bee95da6fb
 103 | 6aad079c8fc66acfa67501e77f0aae37 | 373a09eddb796a594d55ea5cd3e711bb | 6d4ce4eacef2cd50a7d6ff0bdb3e24d3 | a2fedc4945604a809e6fc62723d19179
 104 | aeb0db5412e47ac90a2bfe7c2359a8b8 | 7692a605302604cb98b1ca8c8cafd2ef | 8c1e9a27d5d39fe01a14575180b99a53 | a2de88a66094aebeda1f1bafe3c2f31b
 105 | 1c3192747a19103ee2c08a0293f1b3e6 | e81d402107c98325760b537306c2d2dd | e6831f222ed661135367fe4488b04b88 | 9ad0e82847c1c88ca86d19b43d234bae
 106 | 64909d07f221f86cb354cbb527673241 | 0ea4484e03cb705138ebea4aba869921 | c22b4991a719cce4ae14a3cec0af0976 | 1ac6c063471b264993a47af776a8e761
 107 | a567436a5a7ed430ac400aafa3f4599c | 8521a624030681d29c7d4fbd33926429 | f65d2c666e91d3b0b321f842ef34be77 | 0712d1bffae27bf39f1302797d5e6dd2
 108 | ccbbf4c4a4af4840cbd1471250774ac0 | 7261ba09f353ab9afef3bbbe8a184323 | 0a490430deeb45f3279b57cf6f8877f0 | b1e24bc6139fdf7d6ca709938bd2afc2
 109 | d1ee53f11c5214fc3d2204b8c7713197 | 65069c6e4798ce4a7a5db18145ff2f6d | 2b1c87d58f043e56a132e5bb4432de7e | 53bdc67731093fabce6e7a61d757859d
 110 | 917588e6acc1b3853d37100b59901358 | 70d4efab8eefbe41a922fff1d7ca26ab | 22b8c9b15169ce58bfc9f8756ea08055 | cef352db9dfc59af5c66504c0f59eb86
 111 | d22682b6968c02bef83c7812c7f4e348 | f31f922f1d890a4596f47193fcbc8c27 | f16747fc1a57b47cb9f6a32d36d93469 | ffb563a9f45bf7d350bcf5a2bf1864a1
 112 | 9cd29f04dd8d252e5fde1423b2b16a7e | e6fa5aaee8d6f58c86117fb0a2f1e2fd | 4b0152e814044f17d1ace2c90484134d | 0a972db5d110e1f39a74e8cc16fa3d5d
 113 | acb0a0256696d6a976b7148945d0714c | f154d8f876d5de52a01f2c91cad47e5c | a368d0dc870011b26fbef2248f62b162 | 62be4c8b193c597df7ed64a9cf777262
 114 | ec22844d38571685c6aa26fcb14b16a5 | f4fe94ea2e38d6cce2df45ccdfc0e90e | b9eac8bb8dcadb50800bb02d5d1c8955 | ad0607fef8fa015c8aa09b31b610501c
 115 | c5896414a19ea1d1f5f98aa70dff4ea5 | 8a2e3a3c53e5d50fb04c68fcfad7169e | 6caeed8f69ea984c68e153494d5dcfbc | 687f0ec42bec7308ebd484bef50562b9
 116 | f688d3774eec1dc9060aa9a4fc67ae62 | f318222497a4914f253bca870acde0f0 | db76573082833b88fe99bf4abee54337 | 6365b914779a5124c8101a7b63c49ba1
 117 | f9587bb9e526f9ba0d6920d293984036 | fc5eaf6d6f1503206f4211a14f9124b0 | 12801e3853919b8993e99c8e8e01f65d | 1e86cfea4fc3737f8b132b186a5385b4
 118 | 3a23cbfcffbb3a2fef5570d39266f3a9 | 71447d53fb4a3a659210d6648e7b9447 | be93fa392222fa576ea7dbad34a689b1 | a46bcb747afd6745d8cd202786447463
 119 | d4bbd775dd25464b0578dc7b94237b83 | 3c603416136a07044c5dc33b4a498748 | 4b87717cbbec1733330cbf74c93a9821 | 880221c1a310b9e4db9843b921ce249d
(20 строк)

otus=# analyze public.user;
ANALYZE

otus=# EXPLAIN SELECT * FROM "user" WHERE id=200;
                               QUERY PLAN
------------------------------------------------------------------------
Index Scan using user_pk on "user"  (cost=0.14..2.36 rows=1 width=140)
    Index Cond: (id = 200)
(2 строки)
                               
otus=# EXPLAIN ANALYZE SELECT * FROM "user" WHERE id=200;
                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------
Index Scan using user_pk on "user"  (cost=0.14..2.36 rows=1 width=140) (actual time=0.006..0.007 rows=1 loops=1)
    Index Cond: (id = 200)
Planning Time: 0.244 ms
Execution Time: 0.019 ms
(4 строки)                               
```
#### Создадим индекс и сделаем EXPLAIN:
```postgresql
otus=# CREATE INDEX ON public.user(id);
CREATE INDEX

otus=# ANALYZE public.user;
ANALYZE

otus=# EXPLAIN (costs off) SELECT * FROM public.user WHERE id=200;
               QUERY PLAN
----------------------------------------
 Index Scan using user_id_idx on "user"
   Index Cond: (id = 200)
(2 строки)

otus=# EXPLAIN SELECT * FROM public.user;
               QUERY PLAN
----------------------------------------------------------
 Seq Scan on "user"  (cost=0.00..4.01 rows=101 width=140)
(1 строка)

otus=# EXPLAIN (costs off) SELECT * FROM public.user WHERE id=100;
               QUERY PLAN
----------------------------------------
 Index Scan using user_id_idx on "user"
   Index Cond: (id = 100)
(2 строки)

otus=# EXPLAIN (costs off) SELECT * FROM public.user WHERE id<=200;
               QUERY PLAN
-----------------------
 Seq Scan on "user"
   Filter: (id <= 200)
(2 строки)
```
Добавим ещё значений: 
```postgresql
otus=# INSERT INTO public.user SELECT id, MD5(random()::TEXT)::TEXT, MD5(random()::TEXT)::TEXT, MD5(random()::TEXT)::TEXT, MD5(random()::TEXT)::TEXT FROM generate_series(201, 1000) AS id;
INSERT 0 800

otus=# EXPLAIN ANALYZE SELECT * FROM "user" WHERE id = 500;
                                                      QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Index Scan using user_id_idx on "user"  (cost=0.43..2.65 rows=1 width=140) (actual time=0.065..0.068 rows=1 loops=1)
   Index Cond: (id = 500)
 Planning Time: 0.320 ms
 Execution Time: 0.086 ms
(4 строки)

otus=# EXPLAIN ANALYZE SELECT * FROM "user" WHERE id <= 150;
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Index Scan using user_pk on "user"  (cost=0.43..53267.37 rows=1036008 width=140) (actual time=0.016..0.024 rows=51 loops=1)
   Index Cond: (id <= 150)
 Planning Time: 0.057 ms
 Execution Time: 0.037 ms
(4 строки)
```
>> Тут PostgreSQL не выгодно использовать Seq Scan
```postgresql
otus=# EXPLAIN (costs off) SELECT * FROM "user" WHERE id <= 1000;
       QUERY PLAN
------------------------
 Seq Scan on "user"
   Filter: (id <= 1000)
(2 строки)

otus=# EXPLAIN ANALYZE SELECT * FROM "user" WHERE id <= 1000;
                                                  QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 Seq Scan on "user"  (cost=0.00..87445.19 rows=2071808 width=140) (actual time=0.009..76.449 rows=901 loops=1)
   Filter: (id <= 1000)
 Planning Time: 0.758 ms
 Execution Time: 76.480 ms
(4 строки)
```
>> А тут стало выгодно использовать Seq Scan.

#### Реализовать индекс для полнотекстового поиска
```postgresql
otus=# CREATE INDEX ON public.user (surname);
CREATE INDEX
otus=# EXPLAIN SELECT * FROM public.user WHERE id < 500 AND surname = 'd';
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Index Scan using user_surname_idx on "user"  (cost=0.28..2.50 rows=1 width=140)
   Index Cond: ((surname)::text = 'd'::text)
   Filter: (id < 500)
(3 строки)

otus=# EXPLAIN SELECT * FROM public.user WHERE surname = 'f318222497a4914f253bca870acde0f0';
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Index Scan using user_surname_idx on "user"  (cost=0.28..2.50 rows=1 width=140)
   Index Cond: ((surname)::text = 'f318222497a4914f253bca870acde0f0'::text)
(2 строки)

otus=# EXPLAIN SELECT * FROM public.user WHERE id <= 1000 AND surname = 'dc';
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Index Scan using user_surname_idx on "user"  (cost=0.28..2.50 rows=1 width=140)
   Index Cond: ((surname)::text = 'dc'::text)
   Filter: (id <= 1000)
(3 строки)

otus=# EXPLAIN ANALYZE SELECT * FROM public.user WHERE id <= 1000 AND surname = 'dc';
                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Index Scan using user_surname_idx on "user"  (cost=0.28..2.50 rows=1 width=140) (actual time=0.011..0.011 rows=0 loops=1)
   Index Cond: ((surname)::text = 'dc'::text)
   Filter: (id <= 1000)
 Planning Time: 0.871 ms
 Execution Time: 0.023 ms
(5 строк)
                                                        
otus=# EXPLAIN ANALYZE SELECT * FROM public.user WHERE id > 200 LIMIT 5;
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=10.31..11.42 rows=1 width=140) (actual time=273.966..274.013 rows=5 loops=1)
   ->  Bitmap Heap Scan on "user"  (cost=10.31..11.42 rows=1 width=140) (actual time=273.965..273.967 rows=5 loops=1)
         Recheck Cond: (id > 200)
         Heap Blocks: exact=59246
         ->  Bitmap Index Scan on user_id_idx  (cost=0.00..10.31 rows=1 width=0) (actual time=138.227..138.228 rows=2785308 loops=1)
               Index Cond: (id > 200)
 Planning Time: 1.038 ms
 Execution Time: 274.290 ms
(8 строк)
```
#### Реализовать индекс на часть таблицы или индекс на поле с функцией
```postgresql

```