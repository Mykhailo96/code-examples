## Account Tokenization API service

### API controller to tokenize and manage account data

```csharp
[ApiVersion("1.0")]
[Authorize]
[Route("api/accounts")]
[ProducesResponseType(typeof(void), StatusCodes.Status403Forbidden)]
public sealed class AccountController : AppController
{
    private readonly IAccountService _accountService;

    public AccountController(IAccountService accountService)
    {
        _accountService = accountService;
    }

    /// <summary>
    /// Create a new account.
    /// Required scopes: token.
    /// </summary>
    /// <param name="accountCreateRequestVM"></param>
    /// <returns></returns>
    [HttpPost]
    [RequiredScope(Scopes.Token)]
    [ProducesResponseType(typeof(AccountTokenizeResponseVM), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(BadRequestVM), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateAccount([FromBody] AccountCreateRequestVM accountCreateRequestVM)
    {
        var tokenizeResult = await _accountService.CreateAccountAsync(
            accountCreateRequestVM.AccountNumber,
            accountCreateRequestVM.ExpirationDate,
            accountCreateRequestVM.HolderEmail,
            ClientId.Value);

        return tokenizeResult.Match<IActionResult>(
            token => Ok(new AccountTokenizeResponseVM { Token = token }),
            failedToTokenize => InternalServerError(failedToTokenize.Message()),
            invalidAccountNumber => BadRequest(nameof(AccountCreateRequestVM.AccountNumber), invalidAccountNumber.Message()));
    }

    /// <summary>
    /// Update account details.
    /// Required scopes: token.
    /// </summary>
    /// <param name="accountUpdateRequestVM"></param>
    /// <returns></returns>
    [HttpPut]
    [RequiredScope(Scopes.Token)]
    [ProducesResponseType(typeof(AccountUpdateResponseVM), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(BadRequestVM), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(string), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateAccount([FromBody] AccountUpdateRequestVM accountUpdateRequestVM)
    {
        var updateAccountResult = await _accountService.UpdateAccountAsync(
            accountUpdateRequestVM.AccountNumber,
            accountUpdateRequestVM.ExpirationDate,
            accountUpdateRequestVM.HolderEmail,
            accountUpdateRequestVM.Token,
            ClientId.Value);

        return updateAccountResult.Match<IActionResult>(
            token => Ok(new AccountUpdateResponseVM { Token = token }),
            accountNotFound => NotFound(accountNotFound.Message()),
            accountAlreadyExists => BadRequest(nameof(AccountUpdateRequestVM.AccountNumber), accountAlreadyExists.Message()),
            failedToUpdateAccount => InternalServerError(failedToUpdateAccount.Message()),
            invalidAccountNumber => BadRequest(nameof(AccountUpdateRequestVM.AccountNumber), invalidAccountNumber.Message()));
    }

    /// <summary>
    /// Receive a token by corresponding account number.
    /// Required scopes: token.
    /// </summary>
    /// <param name="tokenFindRequestVM"></param>
    /// <returns></returns>
    [HttpPost("find")]
    [RequiredScope(Scopes.Token)]
    [ProducesResponseType(typeof(TokenFindResponseVM), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(BadRequestVM), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(string), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> FindToken([FromBody] TokenFindRequestVM tokenFindRequestVM)
    {
        var findTokenResult = await _accountService.FindTokenAsync(tokenFindRequestVM.AccountNumber, ClientId.Value);

        return findTokenResult.Match<IActionResult>(
            tokenData => Ok(new TokenFindResponseVM { Token = tokenData }),
            tokenNotFound => NotFound(tokenNotFound.Message()),
            failedToFindToken => InternalServerError(failedToFindToken.Message()));
    }

    /// <summary>
    /// Delete account by corresponding token.
    /// Required scopes: token.
    /// </summary>
    /// <param name="accountDeleteRequestVM"></param>
    /// <returns></returns>
    [HttpDelete]
    [RequiredScope(Scopes.Token)]
    [ProducesResponseType(typeof(void), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(BadRequestVM), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(string), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DisableAccount([FromBody] AccountDeleteRequestVM accountDeleteRequestVM)
    {
        var accountDisableResult = await _accountService.DisableAccountAsync(accountDeleteRequestVM.Token, ClientId.Value);

        return accountDisableResult.Match<IActionResult>(
            success => Ok(),
            accountNotFound => NotFound(accountNotFound.Message()));
    }
}
```

### Account tokenization service with some business logic

```csharp
internal sealed class TokenizationService : ITokenizationService
{
    ...
    
    public async Task<OneOf<
        CreateAccountResultDTO,
        FailedToTokenizeError,
        InvalidCardBrandError>>
    CreateAccountAsync(
        string accountNumber,
        string expirationDate,
        string holderEmail,
        short clientId,
        string token = null)
    {
        var visibleAccountNumberParts = GetVisibleAccountNumberParts(accountNumber);
    
        var findTokenResult = await FindTokenAsync(
            accountNumber,
            visibleAccountNumberParts.FirstEight,
            visibleAccountNumberParts.LastFour,
            clientId);
    
        if (!findTokenResult.Success) return new FailedToTokenizeError();
    
        var encryptedAccountDetails = await GetEncryptedAccountAsync(accountNumber, findTokenResult.Value);
        if (!encryptedAccountDetails.Success) return new FailedToTokenizeError();
    
        var accountExists = findTokenResult.Value is not null;
        if (accountExists)
        {
            await _mediator.Send(new AccountUpdateCommand(
                findTokenResult.Value.AccountId,
                visibleAccountNumberParts.FirstEight,
                visibleAccountNumberParts.LastFour,
                encryptedAccountDetails.Value.IV,
                encryptedAccountDetails.Value.AccountNumber,
                expirationDate,
                holderEmail));
    
            return new CreateAccountResultDTO()
            {
                AccountId = findTokenResult.Value.AccountId,
                Token = findTokenResult.Value.Token,
                NewAccountCreated = false
            };
        }
    
        var cardBinIdResult = await _cardBinService.GetCardBinIdAsync(visibleAccountNumberParts.FirstEight);
        if (!cardBinIdResult.Success) return new FailedToTokenizeError();
        if (!cardBinIdResult.Value.HasValue) return new InvalidCardBrandError();
    
        var newtoken = GenerateToken();
    
        var accountId = await _mediator.Send(new AccountCreateCommand(
            visibleAccountNumberParts.FirstEight,
            visibleAccountNumberParts.LastFour,
            newtoken,
            encryptedAccountDetails.Value.AccountNumber,
            encryptedAccountDetails.Value.IV,
            encryptedAccountDetails.Value.DataEncryptionKeyId,
            expirationDate,
            holderEmail,
            clientId,
            cardBinIdResult.Value.Value,
            token));
    
        return new CreateAccountResultDTO()
        {
            AccountId = accountId,
            Token = newtoken,
            NewAccountCreated = true
        };
    
    ...
}
```

## Integration Event handler. A background app that is subscribed to multiple topics to receive and hanlde async events from other services/apps

### Extend DI with own Chain of Responsibility

```csharp
services.AddChain()
    .AddHandler<AccountCreatedHandler>()
    .AddHandler<AccountUpdatedHandler>()
    .AddHandler<AccountDeletedHandler>();
```

### Custom Handler Registry and Manager to register event handlers
```csharp
public static class Extensions
{
    public static ChainConfiguration AddChain(this IServiceCollection services)
    {
        HandlerRegistry registry = new();
    
        services.AddSingleton<IHandlerManager>(serviceProvider =>
            new HandlerManager(serviceProvider, registry));
    
        return new ChainConfiguration(services, registry);
    }
}

internal sealed class HandlerManager : IHandlerManager
{
    private readonly IServiceProvider _serviceProvider;
    private readonly HandlerRegistry _handlerRegistry;

    public HandlerManager(IServiceProvider serviceProvider, HandlerRegistry handlerRegistry)
    {
        _serviceProvider = serviceProvider;
        _handlerRegistry = handlerRegistry;
    }

    public async Task<bool> HandleAsync(JsonDocument eventToHandle)
    {
        foreach (var handlerType in _handlerRegistry)
        {
            if (_serviceProvider.GetService(handlerType) is not IHandler handler) continue;

            if (handler.CanHandle(eventToHandle))
            {
                return await handler.HandleAsync(eventToHandle);
            }
        }

        return false;
    }
}
```

### Event Handler. Receive an event, check its type and process if required.
```csharp
internal sealed class AccountCreatedHandler : IHandler
{
    private readonly ILogService _logService;
    private readonly IRegistryService _registryService;

    public AccountCreatedHandler(ILogService logService, IRegistryService registryService)
    {
        _logService = logService;
        _registryService = registryService;
    }

    public bool CanHandle(JsonDocument eventToHandle)
    {
        var eventTypeExists = eventToHandle.RootElement.TryGetProperty(IntegrationEventFields.EventType, out var eventType);

        return eventTypeExists && eventType.ToString() == IntegrationEventTypes.AccountCreated;
    }

    public async Task<bool> HandleAsync(JsonDocument eventToHandle)
    {
        var integrationEvent = JsonSerializer.Deserialize<AccountCreatedIntegrationEvent>(eventToHandle);

        var registerResult = await _registryService.RegisterAsync(
            integrationEvent.ClientId,
            integrationEvent.AccountId);

        if (registerResult.Value is FailedToTokenizeError)
        {
            _logService.LogError("Cannot process integration event", obj: integrationEvent);
        }

        return registerResult.Match(
            _ => true,
            failedToCreate => false);
    }
}
```
