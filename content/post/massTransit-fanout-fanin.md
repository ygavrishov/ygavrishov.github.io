---
title: "Паттерн Fan-Out/Fan-In с использованием MassTransit"
date: 2025-08-17
draft: false
tags: ["Saga", "Patterns", "MassTransit", ".NET"]
---

В предыдущем [примере](../mass-transit-saga) реализации саги оркестратор дожидается ответа от каждого сервиса прежде чем отправить команду следующему. Это надежный подход, который позволяет четко контролировать процесс выполнения, в результате все операции выполняются строго последовательно. Однако иногда результаты шага не требуются на следующем, а значит шаги можно запускать параллельно и тем самым сильно сокращать общее время выполнения операции.

Для таких ситуаций в распреденной архитектуре прижился термин fan-out fan-in, то есть поток выполнения сначала разветвляется, а потом вновь соединяется в одну нить.

{{< figure src="/images/diagrams/massTransit-fanout-fanin/state-diagram.svg" width="600" >}}

## Применение Fan-Out/Fan-In

Если применить Fan-Out/Fan-In к исходной реализации саги, то все команды и события, пересылаемые между сервисами, остаются прежними, однако нам не требуется отдельное состояние на каждый шаг, выполняемый в вызываемом сервисе, достаточно начального состояния и двух конечных

```C# {lineNos=inline}
public class PurchaseStateMachine : MassTransitStateMachine<PurchaseState>
{
    public State ReservationRequested { get; private set; }
    public State Failed { get; private set; }
    public State Completed { get; private set; }
    //...
}
```

Мы по-прежнему хотим выполнять компенсационные действия как и в последовательной реализации саги, а значит нам надо запоминать, какие из параллельных операций выполнились и с каким статусом, поэтому в хранимое состояние саги добавляем новые поля:

```C# {lineNos=inline}
public class PurchaseState : SagaStateMachineInstance
{
    //...
    public StepResult PaymentStepResult { get; set; }
    public StepResult BookingStepResult { get; set; }
    public StepResult TicketGenerationStepRsult { get; set; }
    //...
}
```
где перечисление `StepResult` имеет следующий вид:
```C# {lineNos=inline}
public enum StepResult { Pending, Success, Failed }
```

Теперь когда оркестратор получает запрос на проведение операции, он отправляет сразу три команды и переходит в следующее состояние:

```C# {lineNos=inline}
Initially(
    When(ReservationRequestedEvent)
        .Then(ctx =>
        {
            ctx.Saga.RowNumber = ctx.Message.RowNumber;
            ctx.Saga.SeatNumber = ctx.Message.SeatNumber;
            ctx.Saga.Amount = Random.Shared.Next(100, 500);
        })
        .Send(new Uri("queue:booking-service"), ctx =>
            new ReserveSeatCommand(ctx.Saga.CorrelationId, ctx.Message.RowNumber, ctx.Message.SeatNumber))
        .Send(new Uri("queue:payment-service"), ctx =>
            new ProcessPaymentCommand(ctx.Saga.CorrelationId, ctx.Saga.Amount))
        .Send(new Uri("queue:document-service"), ctx =>
            new GenerateTicketCommand(ctx.Saga.CorrelationId))
        .TransitionTo(ReservationRequested)
);
```

Когда же приходят ответы от вызываемых сервисов, то нужно заполнить результат и проверить, все ли сервисы отработали. Если да, то надо перевести сагу в финальное состояние:

```C# {lineNos=inline}
During(ReservationRequested,
    When(SeatReservedEvent)
        .Then(ctx => ctx.Saga.BookingStepResult = StepResult.Success)
        .ThenAsync(CheckIfAllCompletedOrFailed),
    When(PaymentProcessedEvent)
        .Then(ctx => ctx.Saga.PaymentStepResult = StepResult.Success)
        .ThenAsync(CheckIfAllCompletedOrFailed),
    When(TicketGeneratedEvent)
        .Then(ctx => ctx.Saga.TicketGenerationStepRsult = StepResult.Success)
        .ThenAsync(CheckIfAllCompletedOrFailed),
    When(SeatReservationFailedEvent)
        .Then(ctx => ctx.Saga.BookingStepResult = StepResult.Failed)
        .ThenAsync(CheckIfAllCompletedOrFailed),
    When(PaymentFailedEvent)
        .Then(ctx => ctx.Saga.PaymentStepResult = StepResult.Failed)
        .ThenAsync(CheckIfAllCompletedOrFailed),
    When(TicketGenerationFailedEvent)
        .Then(ctx => ctx.Saga.TicketGenerationStepRsult = StepResult.Failed)
        .ThenAsync(CheckIfAllCompletedOrFailed)
);
```
Метод `CheckIfAllCompletedOrFailed` как раз и занимается проверкой результатов и переводом в финальное состояние:
```C# {lineNos=inline}
private async Task CheckIfAllCompletedOrFailed(BehaviorContext<PurchaseState> context)
{
    var saga = context.Saga;
    if (saga.AllStepsAreCompleted)
    {
        if (saga.AllStepsAreSuccessful)
        {
            saga.CurrentState = Completed.Name;
            Console.WriteLine($"Order {saga.CorrelationId} was processed successfully.");
        }
        else
        {
            await PerformCompensations(context);
            saga.CurrentState = Failed.Name;
            Console.WriteLine($"Order {saga.CorrelationId} processing failed.");
        }
    }
}
```
Компенсацию проводим только там, где сервис отработал и компенсация поддерживается, в нашем случае это PaymentService и BookingService:
```C# {lineNos=inline}
private async Task PerformCompensations(BehaviorContext<PurchaseState> context)
{
    var saga = context.Saga;

    if (saga.PaymentStepResult == StepResult.Success)
        await context.Send(new Uri("queue:payment-service"), new RefundPaymentCommand(saga.CorrelationId));

    if (saga.BookingStepResult == StepResult.Success)
        await context.Send(new Uri("queue:booking-service"), new ReleaseSeatCommand(saga.CorrelationId));
}

```
Все остальное можно оставить неизменным, и этого достаточно, чтобы работа выполнялась параллельно.

## Composite Event
К слову MassTransit имеет интересную конструкцию `Composite Event`, которая позволяет выполнить действие, когда получены несколько указанных событий. Например:
```C# {lineNos=inline}
CompositeEvent(() => OrderReady, x => x.ReadyEventStatus, SubmitOrder, OrderAccepted);
```
Этот код генерирует событие `OrderReady` в тот момент, когда получены все события из списка: `SubmitOrder`, `OrderAccepted`. Порядок событий не имеет значения, `OrderReady` будет порожден, когда придет последнее из событий. Для хранения информации о том, какие события уже получены, используется поле, указанное вторым параметром - `ReadyEventStatus`, это integer поле, которое используется как битовая маска.

Это удобная конструкция, и возникает соблазн использовать ее для реализации Fan-Out/Fan-In, однако этот подход позволяет реагировать только на события об успехе от вызываемых сервисов, а значит выполнить компенсации будет проблематично.

## Заключение
Fan-Out/Fan-In - отличная вещь, которая позволяет сократить общее время выполнения саги, запустив некоторые шаги параллельно. Полный код демо в [репозитории](https://github.com/ygavrishov/massTransit-patterns).