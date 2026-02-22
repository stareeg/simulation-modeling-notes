# SimPy — основы дискретно-событийного моделирования на Python

Привет! В этой лекции мы разберёмся с **SimPy** — Python-библиотекой для дискретно-событийного моделирования. Если до этого мы изучали теорию СМО и сетей массового обслуживания на бумаге, то теперь научимся строить имитационные модели **в коде**. Конспект основан на материалах ноутбуков `basic.ipynb` и `extra.ipynb`.

---

## Зачем это нужно

SimPy (SIMulation in PYthon) — это библиотека для **дискретно-событийного моделирования** (Discrete-Event Simulation, DES) на Python.

Почему именно SimPy?

- **Бесплатная и open-source** — никаких лицензий, всё на GitHub.
- **Чистый Python** — никаких внешних зависимостей, работает везде, где есть Python.
- **Полный программный контроль** — в отличие от GUI-инструментов (Arena, AnyLogic), ты управляешь каждой деталью модели из кода.
- **Легко интегрируется** — с NumPy, Pandas, Matplotlib и любыми другими Python-библиотеками для анализа и визуализации.

!!! warning "Нет графического интерфейса"
    SimPy — это **не** визуальная среда моделирования. Здесь нет drag-and-drop, нет 3D-анимации. Зато есть полная свобода: можно строить модели любой сложности, автоматизировать эксперименты, собирать статистику — всё, что позволяет Python.

---

## Установка

Установка максимально простая:

```bash
pip install simpy
```

И импорт:

```python
import simpy
```

Вот и всё — можно моделировать!

---

## Базовые концепции

SimPy строится на трёх ключевых понятиях:

| Понятие | Описание |
|---|---|
| **Environment** (среда) | Ядро симуляции. Управляет временем и расписанием событий. |
| **Process** (процесс) | Генераторная функция Python (`yield`), описывающая поведение сущности во времени. |
| **Timeout** (задержка) | Событие, которое наступает через заданное время. Используется для моделирования длительности действий. |

### Пример: парковка и вождение

Начнём с простейшего примера — машина, которая бесконечно чередует парковку (5 единиц времени) и вождение (2 единицы времени):

```python
def car(env):
    while True:
        print('Start parking at %d' % env.now)
        parking_duration = 5
        yield env.timeout(parking_duration)

        print('Start driving at %d' % env.now)
        trip_duration = 2
        yield env.timeout(trip_duration)
```

!!! note "Как это работает?"
    Функция `car` — это **генератор**. Каждый `yield env.timeout(...)` приостанавливает выполнение функции и говорит среде: «разбуди меня через столько-то единиц времени». Среда запоминает это и продвигает модельное время к ближайшему событию.

Запускаем:

```python
env = simpy.Environment()       # Создаём среду
env.process(car(env))           # Регистрируем процесс
env.run(until=15)               # Запускаем до t=15
```

Вывод:

```
Start parking at 0
Start driving at 5
Start parking at 7
Start driving at 12
Start parking at 14
```

Обрати внимание: `env.now` всегда показывает текущее модельное время. Время продвигается **дискретно** — от одного события к следующему, минуя «пустые» промежутки.

---

## Ожидание процесса

Процессы могут **ожидать завершения других процессов**. Это удобно для моделирования составных действий. Перепишем пример с машиной, добавив процесс зарядки:

```python
class Car(object):
    def __init__(self, env):
        self.env = env
        self.action = env.process(self.run())

    def run(self):
        while True:
            print('Start parking and charging at %d' % self.env.now)
            charge_duration = 5
            # Ждём завершения процесса зарядки
            yield self.env.process(self.charge(charge_duration))

            print('Start driving at %d' % self.env.now)
            trip_duration = 2
            yield self.env.timeout(trip_duration)

    def charge(self, duration):
        yield self.env.timeout(duration)
```

```python
env = simpy.Environment()
car = Car(env)
env.run(until=15)
```

Вывод:

```
Start parking and charging at 0
Start driving at 5
Start parking and charging at 7
Start driving at 12
Start parking and charging at 14
```

!!! tip "yield env.process(...)"
    Конструкция `yield self.env.process(self.charge(...))` запускает подпроцесс `charge` и **ждёт** его завершения. Только после этого выполнение `run` продолжается. Это аналог `await` в асинхронном программировании.

---

## Прерывание процессов

Иногда процесс нужно **прервать** извне. Например, водитель решает уехать, не дожидаясь полной зарядки. SimPy поддерживает это через механизм `Interrupt`.

Функция водителя, который прерывает зарядку через 3 единицы времени:

```python
def driver(env, car):
    yield env.timeout(3)
    car.action.interrupt()
```

Модифицированный класс Car с обработкой прерывания:

```python
class Car(object):
    def __init__(self, env):
        self.env = env
        self.action = env.process(self.run())

    def run(self):
        while True:
            print('Start parking and charging at %d' % self.env.now)
            charge_duration = 5
            try:
                yield self.env.process(self.charge(charge_duration))
            except simpy.Interrupt:
                print('Was interrupted. Hope, the battery is full enough ...')

            print('Start driving at %d' % self.env.now)
            trip_duration = 2
            yield self.env.timeout(trip_duration)

    def charge(self, duration):
        yield self.env.timeout(duration)
```

```python
env = simpy.Environment()
car = Car(env)
env.process(driver(env, car))
env.run(until=15)
```

Вывод:

```
Start parking and charging at 0
Was interrupted. Hope, the battery is full enough ...
Start driving at 3
Start parking and charging at 5
Start driving at 10
Start parking and charging at 12
```

!!! info "Паттерн try/except"
    Прерывание — это исключение `simpy.Interrupt`. Процесс, который может быть прерван, оборачивает свой `yield` в блок `try/except`. Если прерывание произошло — управление переходит в `except`, и процесс может корректно завершить текущее действие.

---

## Ресурсы (Resource)

**Ресурсы** — один из самых важных элементов SimPy. Ресурс — это объект с ограниченной ёмкостью, который процессы должны **запрашивать** (request) и **освобождать** (release).

### Базовое использование

```python
def car(env, name, bcs, driving_time, charge_duration):
    # Едем к зарядной станции
    yield env.timeout(driving_time)

    print('%s arriving at %d' % (name, env.now))
    with bcs.request() as req:
        yield req  # Ждём свободного места

        print('%s starting to charge at %s' % (name, env.now))
        yield env.timeout(charge_duration)
        print('%s leaving the bcs at %s' % (name, env.now))
```

```python
env = simpy.Environment()
bcs = simpy.Resource(env, capacity=2)  # Станция на 2 места

for i in range(4):
    env.process(car(env, 'Car %d' % i, bcs, i*2, 5))

env.run()
```

Вывод:

```
Car 0 arriving at 0
Car 0 starting to charge at 0
Car 1 arriving at 2
Car 1 starting to charge at 2
Car 2 arriving at 4
Car 0 leaving the bcs at 5
Car 2 starting to charge at 5
Car 3 arriving at 6
Car 1 leaving the bcs at 7
Car 3 starting to charge at 7
Car 2 leaving the bcs at 10
Car 3 leaving the bcs at 12
```

!!! tip "Контекстный менеджер (with)"
    Конструкция `with bcs.request() as req` автоматически освобождает ресурс при выходе из блока `with`. Это безопаснее, чем ручной `release()` — ресурс освобождается даже при исключении.

### Ручной request/release

Можно управлять ресурсами и вручную:

```python
def resource_user(env, resource):
    request = resource.request()   # Генерируем запрос
    yield request                  # Ждём доступа
    print('Resource request', env.now)
    yield env.timeout(1)           # Работаем
    resource.release(request)      # Освобождаем
    print('Resource release', env.now)
```

### Статистика ресурса

SimPy предоставляет полезные атрибуты для мониторинга ресурсов:

| Атрибут | Описание |
|---|---|
| `res.count` | Текущее число занятых слотов |
| `res.capacity` | Общая ёмкость ресурса |
| `res.users` | Список текущих пользователей |
| `res.queue` | Очередь ожидающих запросов |

```python
def print_stats(res):
    print(f'{res.count} of {res.capacity} slots are allocated.')
    print(f'  Users: {res.users}')
    print(f'  Queued events: {res.queue}')

def user(res):
    print_stats(res)
    with res.request() as req:
        yield req
        print_stats(res)
    print_stats(res)
```

```python
env = simpy.Environment()
res = simpy.Resource(env, capacity=1)
procs = [env.process(user(res)), env.process(user(res))]
env.run()
```

Вывод:

```
0 of 1 slots are allocated.
  Users: []
  Queued events: []
1 of 1 slots are allocated.
  Users: [<Request() object at 0x...>]
  Queued events: []
1 of 1 slots are allocated.
  Users: [<Request() object at 0x...>]
  Queued events: [<Request() object at 0x...>]
0 of 1 slots are allocated.
  Users: []
  Queued events: [<Request() object at 0x...>]
1 of 1 slots are allocated.
  Users: [<Request() object at 0x...>]
  Queued events: []
0 of 1 slots are allocated.
  Users: []
  Queued events: []
```

---

## Приоритетные ресурсы

Обычный `Resource` обслуживает запросы в порядке поступления (FIFO). Но иногда нужны **приоритеты** — например, VIP-клиент должен обслуживаться раньше обычного.

`PriorityResource` позволяет назначать приоритет каждому запросу:

```python
def resource_user(name, env, resource, wait, prio):
    yield env.timeout(wait)
    with resource.request(priority=prio) as req:
        print(f'{name} requesting at {env.now} with priority={prio}')
        yield req
        print(f'{name} got resource at {env.now}')
        yield env.timeout(3)
```

```python
env = simpy.Environment()
res = simpy.PriorityResource(env, capacity=1)
p1 = env.process(resource_user(1, env, res, wait=0, prio=0))
p2 = env.process(resource_user(2, env, res, wait=1, prio=0))
p3 = env.process(resource_user(3, env, res, wait=2, prio=-1))
env.run()
```

Вывод:

```
1 requesting at 0 with priority=0
1 got resource at 0
2 requesting at 1 with priority=0
3 requesting at 2 with priority=-1
3 got resource at 3
2 got resource at 6
```

!!! warning "Чем меньше число — тем выше приоритет!"
    В SimPy приоритет задаётся числом: **меньшее значение = более высокий приоритет**. В примере выше процесс 3 с `priority=-1` обслуживается раньше процесса 2 с `priority=0`, хотя пришёл позже.

---

## Вытесняющие ресурсы

`PreemptiveResource` идёт ещё дальше — процесс с более высоким приоритетом может **вытеснить** текущего пользователя ресурса:

```python
def resource_user(name, env, resource, wait, prio):
    yield env.timeout(wait)
    with resource.request(priority=prio) as req:
        print(f'{name} requesting at {env.now} with priority={prio}')
        yield req
        print(f'{name} got resource at {env.now}')
        try:
            yield env.timeout(3)
        except simpy.Interrupt as interrupt:
            by = interrupt.cause.by
            usage = env.now - interrupt.cause.usage_since
            print(f'{name} got preempted by {by} at {env.now}'
                  f' after {usage}')
```

```python
env = simpy.Environment()
res = simpy.PreemptiveResource(env, capacity=1)
p1 = env.process(resource_user(1, env, res, wait=0, prio=0))
p2 = env.process(resource_user(2, env, res, wait=1, prio=0))
p3 = env.process(resource_user(3, env, res, wait=2, prio=-1))
env.run()
```

Вывод:

```
1 requesting at 0 with priority=0
1 got resource at 0
2 requesting at 1 with priority=0
3 requesting at 2 with priority=-1
1 got preempted by <Process(resource_user) object at 0x...> at 2 after 2
3 got resource at 2
2 got resource at 5
```

!!! info "preempt=True vs preempt=False"
    При использовании `PreemptiveResource` можно указать параметр `preempt` в запросе:

    - `preempt=True` (по умолчанию) — процесс **может вытеснить** текущего пользователя, если имеет более высокий приоритет.
    - `preempt=False` — процесс получит приоритет в очереди, но **не будет вытеснять** текущего пользователя.

    ```python
    def user(name, env, res, prio, preempt):
        with res.request(priority=prio, preempt=preempt) as req:
            try:
                print(f'{name} requesting at {env.now}')
                yield req
                print(f'{name} got resource at {env.now}')
                yield env.timeout(3)
            except simpy.Interrupt:
                print(f'{name} got preempted at {env.now}')
    ```

    Пример: процесс B с `preempt=False` и более высоким приоритетом не вытесняет A, а просто ждёт в очереди с приоритетом:

    ```python
    env = simpy.Environment()
    res = simpy.PreemptiveResource(env, capacity=1)
    A = env.process(user('A', env, res, prio=0, preempt=True))
    env.run(until=1)  # A получает ресурс

    B = env.process(user('B', env, res, prio=-2, preempt=False))
    C = env.process(user('C', env, res, prio=-1, preempt=True))
    env.run()
    ```

    ```
    A requesting at 0
    A got resource at 0
    B requesting at 1
    C requesting at 1
    B got resource at 3
    C got resource at 6
    ```

    B не вытесняет A (хотя имеет более высокий приоритет), потому что `preempt=False`. Зато B обслуживается раньше C благодаря более высокому приоритету в очереди.

---

## Контейнеры (Container)

**Container** — это ресурс, моделирующий **непрерывную ёмкость**: бак с топливом, аккумулятор, склад с однородным товаром. В отличие от `Resource` (дискретные слоты), `Container` работает с **уровнем** (level).

| Атрибут/Метод | Описание |
|---|---|
| `capacity` | Максимальная ёмкость контейнера |
| `level` | Текущий уровень заполнения |
| `put(amount)` | Добавить `amount` в контейнер (событие) |
| `get(amount)` | Извлечь `amount` из контейнера (событие) |

### Пример: автозаправка

```python
class GasStation:
    def __init__(self, env):
        self.fuel_dispensers = simpy.Resource(env, capacity=2)
        self.gas_tank = simpy.Container(env, init=100, capacity=1000)
        self.mon_proc = env.process(self.monitor_tank(env))

    def monitor_tank(self, env):
        while True:
            if self.gas_tank.level < 100:
                print(f'Calling tanker at {env.now}')
                env.process(tanker(env, self))
            yield env.timeout(15)

def tanker(env, gas_station):
    yield env.timeout(10)  # 10 минут на дорогу
    print(f'Tanker arriving at {env.now}')
    amount = gas_station.gas_tank.capacity - gas_station.gas_tank.level
    yield gas_station.gas_tank.put(amount)

def car(name, env, gas_station):
    print(f'Car {name} arriving at {env.now}')
    with gas_station.fuel_dispensers.request() as req:
        yield req
        print(f'Car {name} starts refueling at {env.now}')
        yield gas_station.gas_tank.get(40)
        yield env.timeout(5)
        print(f'Car {name} done refueling at {env.now}')

def car_generator(env, gas_station):
    for i in range(4):
        env.process(car(i, env, gas_station))
        yield env.timeout(5)
```

```python
env = simpy.Environment()
gas_station = GasStation(env)
car_gen = env.process(car_generator(env, gas_station))
env.run(35)
```

Вывод:

```
Car 0 arriving at 0
Car 0 starts refueling at 0
Car 1 arriving at 5
Car 0 done refueling at 5
Car 1 starts refueling at 5
Car 2 arriving at 10
Car 1 done refueling at 10
Car 2 starts refueling at 10
Calling tanker at 15
Car 3 arriving at 15
Car 3 starts refueling at 15
Tanker arriving at 25
Car 2 done refueling at 30
Car 3 done refueling at 30
```

!!! note "Блокировка при нехватке ресурса"
    Обрати внимание: `yield gas_station.gas_tank.get(40)` **блокирует** процесс, если в баке недостаточно топлива. Машины 2 и 3 ждут, пока танкер не привезёт бензин. Это моделируется автоматически — никакого ручного кода ожидания!

---

## Хранилища (Store)

**Store** — это ресурс для хранения **дискретных объектов** (в отличие от Container, который хранит «однородную жидкость»). Идеально подходит для паттерна **производитель-потребитель**.

```python
def producer(env, store):
    for i in range(100):
        yield env.timeout(2)
        yield store.put(f'spam {i}')
        print(f'Produced spam at', env.now)

def consumer(name, env, store):
    while True:
        yield env.timeout(1)
        print(name, 'requesting spam at', env.now)
        item = yield store.get()
        print(name, 'got', item, 'at', env.now)
```

```python
env = simpy.Environment()
store = simpy.Store(env, capacity=2)
prod = env.process(producer(env, store))
consumers = [env.process(consumer(i, env, store)) for i in range(2)]
env.run(until=5)
```

Вывод:

```
0 requesting spam at 1
1 requesting spam at 1
Produced spam at 2
0 got spam 0 at 2
0 requesting spam at 3
Produced spam at 4
1 got spam 1 at 4
```

!!! tip "Store vs Container vs Resource"
    - **Resource** — дискретные слоты (парковочные места, кассиры). Процесс занимает слот и освобождает его.
    - **Container** — однородная ёмкость (бензин, электричество, деньги). Можно добавлять и забирать произвольные количества.
    - **Store** — хранилище именованных объектов (товары, заказы, сообщения). Каждый объект уникален.

---

## Параллельные процессы

SimPy легко поддерживает **параллельное выполнение** нескольких процессов. Все процессы работают в одной среде и координируются через общее модельное время.

```python
def clock(env, name, tick):
    while True:
        print(name, env.now)
        yield env.timeout(tick)
```

```python
env = simpy.Environment()
env.process(clock(env, 'fast', 0.5))
env.process(clock(env, 'slow', 1))
env.run(until=2.1)
```

Вывод:

```
fast 0
slow 0
fast 0.5
slow 1
fast 1.0
fast 1.5
slow 2
fast 2.0
```

!!! note "Как это работает внутри?"
    SimPy использует **очередь событий** (event queue). Когда процесс делает `yield env.timeout(tick)`, в очередь добавляется событие «разбудить этот процесс через `tick` единиц времени». Среда обрабатывает события в хронологическом порядке. Если два события запланированы на одно время — они выполняются в порядке добавления в очередь (FIFO).

---

## Возврат значений

Процессы в SimPy могут **возвращать значения** — как через `return`, так и через параметр `value` у Timeout.

### Через параметр value у Timeout

```python
def example(env):
    value = yield env.timeout(1, value=42)
    print('now=%d, value=%d' % (env.now, value))
```

```python
env = simpy.Environment()
p = env.process(example(env))
env.run()
```

```
now=1, value=42
```

### Через return

```python
def my_proc(env):
    yield env.timeout(1)
    return 'Armen'
```

```python
env = simpy.Environment()
proc = env.process(my_proc(env))
env.run(until=proc)
```

Результат: `'Armen'`

!!! tip "Зачем возвращать значения?"
    Это полезно, когда один процесс ожидает результата другого. Например, процесс-клиент отправляет запрос процессу-серверу и ждёт ответа. Возвращаемое значение — это ответ сервера.

---

## Пошаговое выполнение

Иногда нужно выполнять симуляцию **по одному шагу** — например, для отладки или интеграции с внешней системой. SimPy предоставляет для этого два метода:

| Метод | Описание |
|---|---|
| `env.peek()` | Возвращает время следующего события (без выполнения) |
| `env.step()` | Выполняет одно следующее событие |

```python
def my_proc(env):
    while True:
        yield env.timeout(1)
        print('Armen at %d' % env.now)
```

```python
env = simpy.Environment()
env.process(my_proc(env))
until = 10
while env.peek() < until:
    env.step()
```

Вывод:

```
Armen at 1
Armen at 2
Armen at 3
Armen at 4
Armen at 5
Armen at 6
Armen at 7
Armen at 8
Armen at 9
```

!!! info "Когда использовать?"
    Пошаговое выполнение полезно для:

    - **Отладки** — можно остановиться после каждого события и проверить состояние.
    - **Интеграции** — если SimPy-модель встроена в другую систему, которая управляет тактами.
    - **Визуализации** — обновлять картинку после каждого шага.

---

## Мониторинг процессов

SimPy позволяет узнать, какой процесс сейчас активен, через атрибут `env.active_process`:

```python
def subfunc(env):
    print(env.active_process)

def my_proc(env):
    while True:
        print(env.active_process)
        subfunc(env)
        yield env.timeout(1)
```

```python
env = simpy.Environment()
p1 = env.process(my_proc(env))
print(env.active_process)   # None — между событиями
env.step()
print(env.active_process)   # None — между событиями
```

Вывод:

```
None
<Process(my_proc) object at 0x...>
<Process(my_proc) object at 0x...>
None
```

!!! note "active_process"
    `env.active_process` возвращает текущий активный процесс **только во время выполнения** обработчика события. Между событиями (в «управляющем» коде) он возвращает `None`.

---

## Sleep until woken up

Иногда процесс должен «заснуть» и ждать, пока его **явно разбудят**. Это паттерн **passivation/reactivation** — процесс переходит в пассивное состояние и активируется по внешнему событию.

В SimPy это реализуется через **события** (`env.event()`):

```python
from random import seed, randint
seed(23)

class EV:
    def __init__(self, env):
        self.env = env
        self.drive_proc = env.process(self.drive(env))
        self.bat_ctrl_proc = env.process(self.bat_ctrl(env))
        self.bat_ctrl_reactivate = env.event()

    def drive(self, env):
        while True:
            # Едем 20-40 минут
            yield env.timeout(randint(20, 40))

            # Паркуемся на 1-6 часов
            print('Start parking at', env.now)
            self.bat_ctrl_reactivate.succeed()         # "разбудить" контроллер
            self.bat_ctrl_reactivate = env.event()     # новое событие для следующего раза
            yield env.timeout(randint(60, 360))
            print('Stop parking at', env.now)

    def bat_ctrl(self, env):
        while True:
            print('Bat. ctrl. passivating at', env.now)
            yield self.bat_ctrl_reactivate             # "заснуть"
            print('Bat. ctrl. reactivated at', env.now)

            # Умная зарядка...
            yield env.timeout(randint(30, 90))
```

```python
env = simpy.Environment()
ev = EV(env)
env.run(until=150)
```

Вывод:

```
Bat. ctrl. passivating at 0
Start parking at 29
Bat. ctrl. reactivated at 29
Bat. ctrl. passivating at 60
Stop parking at 131
```

!!! tip "Как работает паттерн?"
    1. `bat_ctrl` делает `yield self.bat_ctrl_reactivate` — «засыпает», ожидая события.
    2. `drive` вызывает `self.bat_ctrl_reactivate.succeed()` — «будит» контроллер.
    3. После пробуждения создаётся **новое** событие (`env.event()`), чтобы контроллер мог заснуть снова в следующей итерации.

---

## Ожидание завершения другого процесса

SimPy поддерживает **составные условия ожидания** — можно ждать выполнения нескольких событий одновременно.

### AND-условие (оба должны завершиться)

Оператор `&` — ждём, пока **оба** события не произойдут:

```python
class EV:
    def __init__(self, env):
        self.env = env
        self.drive_proc = env.process(self.drive(env))

    def drive(self, env):
        while True:
            yield env.timeout(randint(20, 40))

            print('Start parking at', env.now)
            charging = env.process(self.bat_ctrl(env))
            parking = env.timeout(randint(60, 360))
            yield charging & parking  # Ждём И зарядки, И окончания парковки
            print('Stop parking at', env.now)

    def bat_ctrl(self, env):
        print('Bat. ctrl. started at', env.now)
        yield env.timeout(randint(30, 90))
        print('Bat. ctrl. done at', env.now)
```

```python
env = simpy.Environment()
ev = EV(env)
env.run(until=310)
```

Вывод:

```
Start parking at 21
Bat. ctrl. started at 21
Bat. ctrl. done at 101
Stop parking at 186
Start parking at 207
Bat. ctrl. started at 207
Bat. ctrl. done at 279
```

Машина уезжает только когда **и зарядка завершена, и время парковки истекло**.

### OR-условие (достаточно одного)

Оператор `|` — ждём, пока **хотя бы одно** событие произойдёт:

```python
class EV:
    def __init__(self, env):
        self.env = env
        self.drive_proc = env.process(self.drive(env))

    def drive(self, env):
        while True:
            yield env.timeout(randint(20, 40))

            print('Start parking at', env.now)
            charging = env.process(self.bat_ctrl(env))
            parking = env.timeout(360)
            yield charging | parking  # Ждём ИЛИ зарядки, ИЛИ окончания парковки
            if not charging.triggered:
                charging.interrupt('Need to go!')
            print('Stop parking at', env.now)

    def bat_ctrl(self, env):
        print('Bat. ctrl. started at', env.now)
        try:
            yield env.timeout(randint(60, 90))
            print('Bat. ctrl. done at', env.now)
        except simpy.Interrupt as i:
            print('Bat. ctrl. interrupted at', env.now, 'msg:', i.cause)
```

!!! warning "AND vs OR"
    - `yield A & B` — процесс продолжится, когда **оба** события A и B произойдут (аналог `asyncio.gather`).
    - `yield A | B` — процесс продолжится, когда **любое** из событий A или B произойдёт (аналог `asyncio.wait` с `FIRST_COMPLETED`).

    Это очень мощный механизм для моделирования сложной логики: «жди, пока ИЛИ закончится ремонт, ИЛИ пройдёт 2 часа — что раньше».

---

## Реальное время

По умолчанию SimPy работает в **виртуальном времени** — симуляция выполняется максимально быстро. Но иногда нужно привязать модельное время к **реальным часам** — например, для демонстрации или управления реальным оборудованием.

### Обычная среда (мгновенно)

```python
import time

def example(env):
    start = time.perf_counter()
    yield env.timeout(1)
    end = time.perf_counter()
    print('Duration of one simulation time unit: %.2fs' % (end - start))

env = simpy.Environment()
proc = env.process(example(env))
env.run(until=proc)
```

```
Duration of one simulation time unit: 0.00s
```

### RealtimeEnvironment

```python
import simpy.rt

env = simpy.rt.RealtimeEnvironment(factor=0.1)
proc = env.process(example(env))
env.run(until=proc)
```

```
Duration of one simulation time unit: 0.10s
```

Параметр `factor` задаёт масштаб: `factor=0.1` означает, что 1 единица модельного времени = 0.1 секунды реального времени.

!!! info "strict-режим"
    По умолчанию `RealtimeEnvironment` работает в **strict-режиме**: если симуляция не успевает за реальным временем (обработка события занимает больше, чем отведённый промежуток), выбрасывается `RuntimeError`.

    ```python
    def slow_proc(env):
        time.sleep(0.02)    # Слишком медленная обработка
        yield env.timeout(1)

    env = simpy.rt.RealtimeEnvironment(factor=0.01)
    proc = env.process(slow_proc(env))
    try:
        env.run(until=proc)
        print('Everything alright')
    except RuntimeError:
        print('Simulation is too slow')
    ```

    ```
    Simulation is too slow
    ```

    Чтобы отключить strict-режим (симуляция продолжится, даже если отстаёт):

    ```python
    env = simpy.rt.RealtimeEnvironment(factor=0.01, strict=False)
    ```

---

**Подведём итог.** SimPy — это мощный и элегантный инструмент для дискретно-событийного моделирования на Python. Его ключевые возможности:

- **Процессы** на основе генераторов Python (`yield`) — интуитивное описание поведения во времени;
- **Ресурсы** (`Resource`, `PriorityResource`, `PreemptiveResource`) — моделирование ограниченных ёмкостей с очередями;
- **Контейнеры** (`Container`) и **хранилища** (`Store`) — для однородных и дискретных ресурсов;
- **Прерывания** (`Interrupt`) — динамическое взаимодействие между процессами;
- **Составные условия** (`&`, `|`) — гибкая логика ожидания;
- **Реальное время** (`RealtimeEnvironment`) — привязка к настоящим часам.

Всё это доступно в нескольких строках Python-кода — без GUI, но с полным контролем и возможностью интеграции с любыми библиотеками Python-экосистемы.
