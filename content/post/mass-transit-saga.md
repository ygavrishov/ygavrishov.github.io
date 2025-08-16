---
title: "Паттерн Saga c использованием MassTransit"
date: 2025-08-15
draft: false
tags: ["Saga", "Patterns", "MassTransit", ".NET"]
---
В этой статье кратко о том, что такое Saga, и как с ней работать в MassTransit на платформе .NET.

<!-- Впервые метафора саги была введена в бородатом 1987 году в статье (https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) исследователей Принстоного университета как средство выполнения долгоживущих транзакций в пределах одной БД, но по-настоящему концепция заиграла новыми красками в эру микросервисов. -->

Преимущества микросервисов оборачиваются и некоторой сложностью: поскольку каждый микросервис имеет свою базу данных, то у нас нет возможности атомарно выполнить транзакцию, касающуюся нескольких микросервисов. На помощь приходит паттерн Saga, которая позволяет разбить долго живущую транзакцию (long-living transaction, LLT) на несколько простых шагов. Каждый из шагов выполняется атомарно и может выполняться в разных сервисах. Сага считается завершенной, если все шаги завершились успешно. Если один из шагов не был успешен, то для ранее выполненных шагов возможно выполнить компенсационные транзакции.

Пример: покупка билета на концерт.

Пусть мы имеем следующие микросервисы:
* Booking Service - резервирование места и отмена бронирования
* Payment Service - платежи
* Document Service - формирование различных документов, в том числе и билетов

Чтобы купить билет в такой системе нам потребуется скоординировать действия всех трех сервисов, при этом отслеживать прогресс выполнения и в случае необходимости выполнять компенсации. Вот как это может выглядеть с применением паттерна Saga:

![Пример архитектуры](/images/diagrams/mass-transit-saga/sequence-diagram.svg)

Для реализации подобной схемы с нуля требуется написать изрядное количество кода, при этом учитывая массу нюансов. Для платформы .NET не так много готовых реализаций саг, в отличие например от Java. Но к счастью есть MassTransit, который помогает избежать написания рутинного кода и, более того, имеет механизмы для решения типовых проблем в такой схеме взаимодействия.

Для реализации саги с MassTransit потребуется сделать следующее:
* описать, что будет представлять собой хранимое состояние саги (реализация интерфейса SagaStateMachineInstance)
* описать сообщения, используемые сагой
* описать граф переходов между состояниями (наследник MassTransitStateMachine<>)
* зарегистрировать сагу в DI контейнере

### Хранимое состояние

Чтобы сага завелась, класс состояния саги должен реализовывать интерфейс `SagaStateMachineInstance`. Название этого и других интерфейса в MassTransit не содержит префикса `I` по историческим причинам.

```C# {lineNos=inline}
public class PurchaseState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }

    public int RowNumber { get; set; }
    public int SeatNumber { get; set; }
    public decimal Amount { get; set; }

    public string CurrentState { get; set; }
}
```
Самое важное свойство здесь это `CorrelationId`, которое является ключом для адресации состояния саги. Любое обрабатываемое сагой сообщение сопоставляется с конкретным экземпляром при помощи этого ключа. Кроме того, класс содержит поля, переданные во входной команде, а также поле Amount (стоимость) билета. Все эти поля сохраняются и в микросервисах, где эти данные обрабатываются (Seat, Row в Booking Service, Amount в PaymentService), но для удобства отладки и мониторинга удобно сохранять это и в состоянии саги. Кроме того, некоторые поля отправляются в свой сервис не сразу, а значит должны где-то быть сохранены.

### Команды и события
Чтобы сага работала, по выбранному транспорту (шине) должны циркулировать сообщения, которые приводят сагу в действие. Их условно можно разделить на две группы:
* **Команды**: сообщения, которые приходят на вход микросервиса и принуждают его к выполнению определенного действия
* **События**: сообщения, которые информируют остальных участников системе о том, что в результате работы микросервиса что-то произошло, например успешно забронировано место или генерация документа прошла с ошибкой.
В этом демо Orchestrator порождает команды для остальных микросервисов и слушает их события в своей шине.

```C# {lineNos=inline}
// Commands
public record PurchaseTicketCommand(Guid OrderId, int RowNumber, int SeatNumber);
public record ReserveSeatCommand(Guid OrderId, int RowNumber, int SeatNumber);
public record ReleaseSeatCommand(Guid OrderId);
public record ProcessPaymentCommand(Guid OrderId, decimal Amount);
public record RefundPaymentCommand(Guid OrderId);
public record GenerateTicketCommand(Guid OrderId);

// Events
public record SeatReserved(Guid OrderId, int RowNumber, int SeatNumber);
public record SeatReservationFailed(Guid OrderId);

public record PaymentProcessed(Guid OrderId);
public record PaymentFailed(Guid OrderId);

public record TicketGenerated(Guid OrderId);
public record TicketGenerationFailed(Guid OrderId);
```
Команда `PurchaseTicketCommand` приходит на вход самому Orchestrator, остальные команды он отправляет сам нижележащим микросервисам.

Эти сообщения должны быть доступны всем микросервисам, участвующим в саге, поэтому они должны быть в отдельном проекте и могут быть оформлены в виде NuGet пакета.


### Машина состояний

Самая важная часть логики описывается в классе, наследуемом от `MassTransitStateMachine<TState>`, по сути Крис, автор библиотеки, реализовал свой собственный DSL для описания состояний и переходов и выделил его в отдельный NuGet пакет `Automatonymous`, а после применил его в проекте MassTransit.

```C# {lineNos=inline}
public class PurchaseStateMachine : MassTransitStateMachine<PurchaseState>
{
    // допустимые состояния саги
    public State ReservationRequested { get; private set; }
    public State Reserved { get; private set; }
    public State Paid { get; private set; }
    public State TicketGenerated { get; private set; }
    public State Failed { get; private set; }

    //на какие события сага реагирует
    public Event<PurchaseTicketCommand> ReservationRequestedEvent { get; private set; }
    public Event<SeatReserved> SeatReservedEvent { get; private set; }
    public Event<SeatReservationFailed> SeatReservationFailedEvent { get; private set; }
    public Event<PaymentProcessed> PaymentProcessedEvent { get; private set; }
    public Event<PaymentFailed> PaymentFailedEvent { get; private set; }
    public Event<TicketGenerated> TicketGeneratedEvent { get; private set; }
    public Event<TicketGenerationFailed> TicketGenerationFailedEvent { get; private set; }

    public PurchaseStateMachine()
    {
        //указывает, какое поле хранит текущее состояние саги
        InstanceState(x => x.CurrentState);

        //описывает, как сага может быть создана
        Initially(
            //по событию "запрошено бронирование места"
            When(ReservationRequestedEvent)
                .Then(ctx =>
                {
                    //задает начальные значения для полей состояния саги
                    ctx.Saga.RowNumber = ctx.Message.RowNumber;
                    ctx.Saga.SeatNumber = ctx.Message.SeatNumber;
                    ctx.Saga.Amount = Random.Shared.Next(100, 500);
                })
                //пересылает сообщение одному из сервисов на выполнение
                .Send(new Uri("queue:booking-service"), ctx =>
                    new ReserveSeatCommand(ctx.Saga.CorrelationId, ctx.Message.RowNumber, ctx.Message.SeatNumber))
                //переводит сагу в новое состояние
                .TransitionTo(ReservationRequested)
        );

        During(ReservationRequested,
            When(SeatReservedEvent)
                .Send(new Uri("queue:payment-service"), ctx =>
                    new ProcessPaymentCommand(ctx.Saga.CorrelationId, ctx.Saga.Amount))
                .TransitionTo(Reserved)
        );

        //описывает переход из конкретного состояния
        During(ReservationRequested,
            When(SeatReservationFailedEvent)
                .Then(ctx =>
                {
                    Console.WriteLine($"Seat reservation failed for Order {ctx.Message.OrderId}");
                })
                .TransitionTo(Failed)
                .Finalize()
        );

        //сокращено для читаемости. Полный код в репозитории
    }
}
```

### Регистрация саги

Чтобы вся механика саги заработала, надо ее зарегистрировать и сконфигурировать. В этом примере сага работает по шине RabbitMQ и сохраняет состояние в PostgreSQL.

```C# {lineNos=inline}
services.AddMassTransit(x =>
{
    //настраивает автоматическое именование очередей RabbitMQ в стиле my-favorite-service
    x.SetKebabCaseEndpointNameFormatter();

    //регистрирует собственно state machine
    x.AddSagaStateMachine<PurchaseStateMachine, PurchaseState>()
    //определяет хранение состояния при помощи EF Core в PostgreSQL
    .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;

            r.AddDbContext<DbContext, TicketDbContext>((provider, builder) =>
            {
                builder.UseNpgsql(hostContext.Configuration.GetConnectionString("DefaultConnection"));
            });
        });

    //определяет RabbitMQ как транспорт для саги
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        //задает конкретную очередь как поставщика сообщений для саги
        cfg.ReceiveEndpoint("ticket-purchase-saga", e =>
        {
            e.ConfigureSaga<PurchaseState>(context);
        });
    });
});
```

### Заключение

В заметке описан самый базовый сценарий использования саги в MassTransit. Когда дело доходит до практики, то всплывает масса нюансов, но к счастью, большинство из них покрывается возможностями библиотеки, надо только копнуть документацию поглубже.

Полный код демо выложен в [репозитории](https://github.com/ygavrishov/massTransit-patterns).