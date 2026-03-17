# Example: Bank Account Rich Domain Model (Class-Based)

Below is an example of how the skill should reason when implementing a rich domain model using the class-based approach with private setters, Result pattern, and EF Core interceptors.

## When to use this approach

- EF Core change tracking is the persistence mechanism
- The aggregate has state-machine behavior (status transitions)
- Business operations may fail for expected reasons (insufficient funds, account closed)
- Domain events need automatic publishing via interceptor

## Ubiquitous Language

- Bounded Context: `Banking`
- Aggregate Root: `Account`
- Value Objects: `AccountNumber`, `Money`, `Currency`
- Entities: `Transaction`
- Enums: `AccountStatus`, `AccountType`
- Domain Events: `AccountCreated`, `AccountActivated`, `AccountSuspended`, `AccountClosed`, `DepositProcessed`, `WithdrawalProcessed`
- Integration Events: `AccountCreatedIntegrationEvent`

## Key invariant rules

- An account can only process deposits/withdrawals when Active
- Deposit and withdrawal amounts must be positive
- Currency must match between operation amount and account balance
- An account cannot be closed with a non-zero balance
- A closed account cannot be activated or suspended
- Account creation goes through a factory method that ensures initial state and raises `AccountCreated` event

## Aggregate root: Account

```csharp
namespace Banking.Domain.Accounts;

public class Account : AggregateRoot<Guid>
{
    public AccountNumber AccountNumber { get; } = null!;
    public Guid CustomerId { get; }
    public Money Balance { get; private set; } = null!;
    public AccountStatus Status { get; private set; }
    public AccountType Type { get; private set; }
    public DateTime? ClosedAt { get; private set; }

    private Account() { } // EF Core

    private Account(AccountNumber accountNumber, Guid customerId, AccountType type, Currency currency)
    {
        Guard.AgainstNullOrEmpty(accountNumber, nameof(accountNumber));
        Guard.AgainstEmptyGuid(customerId, nameof(customerId));

        Id = Guid.NewGuid();
        AccountNumber = accountNumber;
        CustomerId = customerId;
        Type = type;
        Balance = Money.Zero(currency);
        Status = AccountStatus.PendingActivation;
    }

    public static Account CreateNew(Guid customerId, AccountType type, Currency currency)
    {
        var account = new Account(AccountNumber.Generate(), customerId, type, currency);
        account.AddDomainEvent(new AccountCreatedEvent(account.Id, customerId, account.AccountNumber, type.ToString(), DateTime.UtcNow));
        return account;
    }

    public Result Activate()
    {
        if (Status == AccountStatus.Active)
            return Result.Failure("Account is already active");
        if (Status == AccountStatus.Closed)
            return Result.Failure("Cannot activate a closed account");

        Status = AccountStatus.Active;
        AddDomainEvent(new AccountActivatedEvent(Id, CustomerId, AccountNumber.Value, DateTime.UtcNow));
        return Result.Success();
    }

    public Result Close(string reason)
    {
        Guard.AgainstNullOrEmpty(reason, nameof(reason));
        if (Status == AccountStatus.Closed)
            return Result.Failure("Account is already closed");
        if (!Balance.IsZero)
            return Result.Failure("Cannot close account with non-zero balance");

        Status = AccountStatus.Closed;
        ClosedAt = DateTime.UtcNow;
        AddDomainEvent(new AccountClosedEvent(Id, AccountNumber, CustomerId, reason));
        return Result.Success();
    }

    public Result<Transaction> Deposit(Money amount, string description, string reference)
    {
        if (Status != AccountStatus.Active)
            return Result<Transaction>.Failure("Account must be active to process deposits");
        if (amount.Amount <= 0)
            return Result<Transaction>.Failure("Deposit amount must be positive");
        if (amount.Currency != Balance.Currency)
            return Result<Transaction>.Failure("Currency mismatch");

        Balance = Balance.Add(amount);
        var transaction = Transaction.CreateDeposit(Id, amount, description, reference);
        AddDomainEvent(new DepositProcessedEvent(Id, amount, Balance, transaction.Id));
        return Result<Transaction>.Success(transaction);
    }

    public Result<Transaction> Withdraw(Money amount, string description, string reference)
    {
        if (Status != AccountStatus.Active)
            return Result<Transaction>.Failure("Account must be active to process withdrawals");
        if (amount.Amount <= 0)
            return Result<Transaction>.Failure("Withdrawal amount must be positive");
        if (amount.Currency != Balance.Currency)
            return Result<Transaction>.Failure("Currency mismatch");
        if (Balance.Amount < amount.Amount)
            return Result<Transaction>.Failure("Insufficient funds");

        Balance = Balance.Subtract(amount);
        var transaction = Transaction.CreateWithdrawal(Id, amount, description, reference);
        AddDomainEvent(new WithdrawalProcessedEvent(Id, amount, Balance, transaction.Id));
        return Result<Transaction>.Success(transaction);
    }
}
```

## Key differences from immutable record approach

| Aspect | Immutable records | Class-based rich model |
|--------|------------------|----------------------|
| State change | Returns new instance via `with` | Mutates internal state via private setters |
| Failure signaling | Throws `DomainRuleViolation` | Returns `Result` / `Result<T>` |
| Event collection | `ImmutableArray<IDomainEvent>` in record | `List<IDomainEvent>` in base class |
| EF Core mapping | Explicit mapping layer | `OwnsOne` + private constructor |
| Creation | Public constructor or factory | Private constructor + static factory only |
| Identity equality | By value (record default) | By `Id` field (base class override) |

## Application layer usage

```csharp
public class CreateAccountCommandHandler
{
    private readonly IAccountRepository _accountRepository;

    public async Task<Result<AccountDto>> Handle(CreateAccountCommand request, CancellationToken ct)
    {
        var account = Account.CreateNew(request.CustomerId, request.AccountType, new Currency(request.Currency));
        await _accountRepository.AddAsync(account, ct);
        // Domain events published automatically by EF interceptor on SaveChanges
        return Result<AccountDto>.Success(AccountDto.FromDomain(account));
    }
}
```

## Infrastructure: EF Core mapping

```csharp
modelBuilder.Entity<Account>(entity =>
{
    entity.HasKey(a => a.Id);
    entity.OwnsOne(a => a.AccountNumber, an =>
    {
        an.Property(x => x.Value).HasColumnName("AccountNumber").HasMaxLength(10).IsRequired();
    });
    entity.OwnsOne(a => a.Balance, money =>
    {
        money.Property(m => m.Amount).HasColumnName("Balance").HasPrecision(18, 2);
        money.OwnsOne(m => m.Currency, c => c.Property(x => x.Code).HasColumnName("Currency").HasMaxLength(3));
    });
    entity.Ignore(a => a.DomainEvents);
});
```

## Unit test example

```csharp
[Test]
public void Account_Deposit_Should_Raise_DepositProcessedEvent()
{
    var account = Account.CreateNew(Guid.NewGuid(), AccountType.Checking, Currency.USD);
    account.Activate();

    var result = account.Deposit(new Money(100, Currency.USD), "Test deposit", "REF123");

    Assert.That(result.IsSuccess, Is.True);
    Assert.That(account.DomainEvents.OfType<DepositProcessedEvent>().Count(), Is.EqualTo(1));
}

[Test]
public void Account_Withdraw_InsufficientFunds_Should_Return_Failure()
{
    var account = Account.CreateNew(Guid.NewGuid(), AccountType.Checking, Currency.USD);
    account.Activate();

    var result = account.Withdraw(new Money(100, Currency.USD), "Test withdrawal", "REF456");

    Assert.That(result.IsSuccess, Is.False);
    Assert.That(result.Error, Does.Contain("Insufficient funds"));
}
```

## Review criteria

- Private setters protect all mutable state
- Factory method is the only public creation path
- Result pattern used for expected business failures; Guard clauses for programming errors
- Domain events accumulated in base class, cleared after publication
- EF Core maps value objects via `OwnsOne`, ignores `DomainEvents`
- No ORM attributes in domain classes
- Integration events still use Transactional Outbox (interceptor handles domain events only)
