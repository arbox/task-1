# Лабораторная работа 01: Оптимизация потребления памяти программ на Руби

## Актуальная проблема

Существует скрипт, написанный на Руби, для агрегации данных пользовательской статистики.

Входные данные вида:

``` csv
user,0,Hazel,Margarete,19
session,0,0,Internet Explorer 50,81,2018-02-01
session,0,1,Safari 27,88,2017-10-28
session,0,2,Firefox 13,92,2016-11-02
session,0,3,Internet Explorer 40,37,2017-05-31
session,0,4,Internet Explorer 16,37,2017-11-21
session,0,5,Safari 19,22,2017-11-30
session,0,6,Chrome 31,74,2016-08-22
session,0,7,Firefox 46,76,2019-02-04
user,1,Wilfredo,Louetta,40
session,1,0,Firefox 38,68,2018-04-24
session,1,1,Internet Explorer 41,27,2016-09-06
session,1,2,Internet Explorer 1,74,2017-01-05
session,1,3,Firefox 47,28,2017-06-09
session,1,4,Internet Explorer 22,98,2016-11-03
```

Параметры входных данных:

``` shellsession
$ ls -hl data_large.txt
-rw-r--r-- 1 arbox arbox 129M Mar  8 21:10 data_large.txt

$ wc -l data_large.txt
3250940 data_large.txt

$ grep -c user data_large.txt
500000

$ grep -c session data_large.txt
2750940

$ file data_large.txt
data_large.txt: ASCII text

```
Корректность работы скрипта подтверждается автоматизированным тестом и ручной проверкой на небольшой выборке из входных данных.

Проблема: скрипт не отрабатывает за выделенное время (допустим, за время обеденного перерыва удаленного филиала),
полностью загружает одно из ядер процессора, потребление памяти продолжает расти.


## Испытательный стенд

Все данные собирались на машине со следующими параметрами:

``` shellsession
$ sudo lshw -short
H/W path               Device           Class          Description
==================================================================
                                        system         20L7CTO1WW (LENOVO_MT_20L7_BU_Think_FM_ThinkPad T480s)
/0                                      bus            20L7CTO1WW
/0/3                                    memory         16GiB System Memory
/0/3/0                                  memory         8GiB SODIMM DDR4 Synchronous Unbuffered (Unregistered) 2400 MHz (0.4 ns)
/0/3/1                                  memory         8GiB SODIMM DDR4 Synchronous Unbuffered (Unregistered) 2400 MHz (0.4 ns)
/0/7                                    memory         256KiB L1 cache
/0/8                                    memory         1MiB L2 cache
/0/9                                    memory         8MiB L3 cache
/0/a                                    processor      Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz
/0/b                                    memory         128KiB BIOS
/0/100/1d/0                             storage        NVMe SSD Controller SM981/PM981
/0/100/1f                               bridge         Intel(R) 100 Series Chipset Family LPC Controller/eSPI Controller - 9D4E
/3                     wwp0s20f0u6      network        Ethernet interface

$ uname -a
Linux gamma 4.18.0-15-generic #16-Ubuntu SMP Thu Feb 7 10:56:39 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

$ ruby -v
ruby 2.5.3p105 (2018-10-18 revision 65156) [x86_64-linux]
```
Ruby 2.6.1 еще не поддерживается `rbspy`.

## Инструменты

Для эксплоративного анализа использовались утилиты `time`, `free`, `ps` и `htop`.
Для детального профилирования &mdash; стандартная библиотека `benchmark`, `memory_profiler` и [`rbspy`](https://rbspy.github.io/).

## Формирование метрики

Первичная оценка работы скрипта показала загрузку одного из ядер в 100% и постоянно растущий объем потребляемой памяти.
Такое поведение типично для программ, создающих большое количество промежуточных данных.

Постараемся снизить накладные расходы полезной работы, будем оценивать общее время исполнения и объем потребляемой памяти.

## Гарантия корректности работы оптимизированной программы

Программа поставлялась с тестом. Выполнение этого теста позволяет не допустить
изменения логики программы при оптимизации.

TODO: Подтвердить вид функции роста подтребления памяти через сравнение на входных данных разного размера.

## Feedback-Loop

На полном наборе входящих данных оценить производительность нет возможности.
Пользователь является основной единицей, по которой группируются данные.
Для построения эффективной обратной связи нам нужно подготовить (скажем) три
тестовых набора данных небольшого размера: 100, 1000, 10000 пользователей.

``` shell
for limit in {100,1000,10000}; do
  ruby -pe 'BEGIN { $limit = '${limit}'; $count = 0 }; $count += 1 if /^user/; break if $count > $limit' data_large.txt > data_${limit}.txt
done
```

Параметризируем функцию `work()`, чтобы можно было задать входные и выходные данные со стандартными значениями и с любыми пользовательскими.

Вынесем тест в отдельный файл.

Будем работать в следующе цикле (можно использовать любой make):

1. запустить тест;
2. запустить оценку производительности;
3. решить, достаточны ли результаты;
4. goto 1

## Вникаем в детали системы, чтобы найти 20% точек роста

Первичный сбор данных показал:

``` shellsession
$ ruby task-1.rb
                     user     system      total        real
100 users:     23 Mb
                     0.024253   0.000054   0.034553   (0.038230)
1.000 users:   62 Mb
                     0.998884   0.004177   1.014272   (1.014491)
10.000 users:  172 Mb
                     153.656013   0.192062 153.856626 (153.871385)

```
Показатели затраченной памяти кумулятивные и не повторяются в точности из-за работающего сборщика мусора.

Следующим шагом подключили `memory_profiler`.

### Оптимизация №1
Больше всего памяти алоцируется для классов `Array` и `String`.

Попробуем использовать существующие массивы, а не создавать новые (bang methods).
Попробуем минимизировать создание строк заменой на символы.

### Оптимизация №2
Самые часто создаваеммые строки &mdash; это `,` и пустая строка, вынесем их в константы.

### Оптимизация №3
Так как массивы по-прежнему потребляют больше всего памяти, постараемся не создавать большие массивы.
Не будет зачитывать исходные данные полностью в память. Это возможно, так как формат данных поддерживате потоковую обработку.

### Оптимизация №4
Есть ненужные преобразования, как то двойное преобразование дат и ненужное хранение имен и фамилий.
Эти преобразования можно сделать в один шаг.

### Оптимизация №5
Избавимся от многократных итераций по `user.sessions`.

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.

Результаты оптимизации по времени и объему используемой памяти приведены ниже:

``` shellsession
                     user     system      total        real
100 users:     18 Mb
                     0.014782   0.000000   0.023090 (  0.022927)
1.000 users:   27 Mb
                     0.108842   0.007690   0.124158 (  0.124160)
10.000 users:  87 Mb
                     1.150900   0.024334   1.184030 (  1.182995)

```

## Защита от регресса производительности

Для защиты метрик стоит реализовать автомматизированных тестовый стенд
(выделенная виртуальная машина с последовательным выполнение задач),
на котором будут проверяться три параметра:

- _корректность_ работы программы на тестовых данных;
- _сохранение функции роста_ потребления памяти / времени работы на нескольких
  заданных тестовых наборах возрастающего размера (пусть это будет экспонента, но мы будем об этом знать)
- завершение работы программы в заданных максимальных параметрах
  (обеденный перерыв удаленного филиала, машина с 16 Гб) на _случайно_
  сгенерированной выборке входных данных _максимального_ размера.