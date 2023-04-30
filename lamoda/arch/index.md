# Некоторые приёмы проектирования приложений

Я иногда придумываю архитектурные решения уровня приложения и хочу поделиться
приёмами, которые упрощают такую работу.

## ООП

ООП хорошо прежде всего тем, что на этом уровне абстракции мы привыкли думать в
повседневной жизни. Почти всё о чём мы размышляем в быту мы выражаем через
объекты и их поведение. Именно поэтому спроектировать систему как взаимодействие
объектов может быть хорошей идеей.

### Данные и методы

Для описания типов объектов удобно использовать UML-диаграмму классов.

```puml
interface ExternalEvent {
    + DomainEvent() DomainEvent
}

interface DomainEvent {
    + FindNotificationTempaltes() []NotificationTemplate
}
interface NotificationTemplate {
    + RequiredParams() []string
    + Render(params map[string]string) Notification
}

class SMSTemplate {
    - params []string
    - text string
    - useTransliteration bool
}

interface Notification {
    + Send()
}

class SMS {
    + cscomClient CuscomClient
}

ExternalEvent -- DomainEvent
DomainEvent -- NotificationTemplate
NotificationTemplate <-- SMSTemplate
NotificationTemplate -- Notification
Notification <-- SMS
```

У нас есть ещё требование послать уведомление не абы когда, а в определённое
время, поэтому понадобится расписание. Почему отдельный объект, а не
Notification? Потому что данные, которые будут управлять алгоритмом планирования
отсутствуют в нотификации.

```puml
class Schedule {
    - schedule map[string]PrimeTime
    + BestTime(country string, transport string, userLocation time.Location) time.Time
}

class PrimeTime {
    - form int
    - to int
    + Delay(now time.Time, l time.Location) time.Duration
}

Schedule *-- PrimeTime
```

Такие диаграммы связывают данные и поведение. В процессе проектирования мы не
потеряем данные, которые нам необходимы для реализации поведения. Наши
архитектурные границы не пройдут через какой-то класс. Абстракции не протекут.

Например, мы видим, что расписание не зависит от доменного события. Для
планирования отправки нужен userLocation и кто-то должен его предоставить.

Или, ставя задачу на доменное событие OrderConfirmed, возникнет вопрос — откуда
взять данные?

```puml
interface ExternalEvent {
    + DomainEvent() DomainEvent
}

class OrderManagementUpdateV1 {
    + Region *string
    + CountryIso string
    + Delivery Delivery
}

interface DomainEvent {
    + FindNotificationTempaltes() []NotificationTemplate
}

class OrderConfirmed {
    - country string
    - trigger int
    - shipmentTypes []string
    - shipping_methods []string
    - buiseness_models []string
    - orderType string
}

ExternalEvent -- DomainEvent
ExternalEvent <-- OrderManagementUpdateV1
DomainEvent <-- OrderConfirmed
```

### Жизненный цикл программы

Чтобы придумать и описать как работает система, удобно использовать диаграммы
последовательности.

Начать лучше с общей схемы работы системы,

```puml
Processor -> ExternalEvent: DomainEvent()
Processor -> DomainEvent: FindNotificationPlans()
Processor -> NotificationPlan: NextTemplate()
Processor -> NotificationTemplate: RequiredParams()
Processor -> Processor: GetParams()
Processor -> NotificationTemplate: Render()
Processor -> AddressAPI: TzGet()
Processor -> Schedule: BestTime()
database Db
Processor -> Db: Insert()
Sender -> Db: Get()
Sender -> Notification: Send()
Notification -> Cuscom: Send()
```

Такая диаграмма не содержит подробностей реализации, но даёт общее представление
о том, как будет работать система и не упустить важные детали. К ней удобно
обращаться в процессе разработки, менять её по мере погружения в проблему
бизнеса.

Разглядывая такую схему, можно прийти к выводам, что компонента у нас всего 4:

- EventDetector, который порождает доменные события
- Основной флоу: выбрать план, шаблон, отрендерить, запланировать
- DataProvider
- Sender

## На сегодня всё

В UML есть ещё много полезных при проектировании диаграмм, но эти две наверное
самые простые и ценные. С помощью них удобно думать и общаться.

- [plantuml дока](https://plantuml.com/ru/)
- [planttext](https://www.planttext.com/)