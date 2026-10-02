# Part 57: Advanced State Management
## Steps 561-570: Redux Pattern, Flux, State Machines

---

## Step 561: Redux-Style State Management

```csharp
// ============================================
// Redux Pattern for MAUI
// ============================================

// State (immutable record)
public record AppState(
    UserState User,
    ProductState Products,
    CartState Cart,
    OrderState Orders)
{
    public static AppState Initial => new(
        UserState.Guest,
        ProductState.Empty,
        CartState.Empty,
        OrderState.Empty);
}

public record UserState(
    int? UserId, string? Name, string? Email,
    bool IsAuthenticated, bool IsLoading, string? Error)
{
    public static UserState Guest => new(null, null, null, false, false, null);
}

public record ProductState(
    List<ProductDto> Items, bool IsLoading, string? Error,
    string? SearchQuery, int Page)
{
    public static ProductState Empty => new(new(), false, null, null, 1);
}

public record CartState(List<CartItem> Items)
{
    public static CartState Empty => new(new());
    public decimal Total => Items.Sum(i => i.UnitPrice * i.Quantity);
    public int ItemCount => Items.Sum(i => i.Quantity);
}

public record OrderState(List<OrderDto> Items, bool IsLoading, string? Error)
{
    public static OrderState Empty => new(new(), false, null);
}

public record CartItem(int ProductId, string Name, decimal UnitPrice, int Quantity);
```

---

## Step 562: Actions & Reducers

```csharp
// ============================================
// Actions & Reducers
// ============================================

// Actions (discriminated union via abstract record)
public abstract record AppAction;

// User actions
public record LoginAction(string Email, string Password) : AppAction;
public record LoginSuccessAction(int UserId, string Name, string Email) : AppAction;
public record LoginFailureAction(string Error) : AppAction;
public record LogoutAction : AppAction;

// Product actions
public record LoadProductsAction(string? Query = null) : AppAction;
public record LoadProductsSuccessAction(List<ProductDto> Products) : AppAction;
public record LoadProductsFailureAction(string Error) : AppAction;

// Cart actions
public record AddToCartAction(int ProductId, string Name, decimal Price) : AppAction;
public record RemoveFromCartAction(int ProductId) : AppAction;
public record UpdateQuantityAction(int ProductId, int Quantity) : AppAction;
public record ClearCartAction : AppAction;

// Root reducer
public static class RootReducer
{
    public static AppState Reduce(AppState state, AppAction action) => state with
    {
        User = UserReducer.Reduce(state.User, action),
        Products = ProductReducer.Reduce(state.Products, action),
        Cart = CartReducer.Reduce(state.Cart, action),
        Orders = OrderReducer.Reduce(state.Orders, action)
    };
}

public static class UserReducer
{
    public static UserState Reduce(UserState state, AppAction action) => action switch
    {
        LoginAction => state with { IsLoading = true, Error = null },
        LoginSuccessAction a => new UserState(a.UserId, a.Name, a.Email, true, false, null),
        LoginFailureAction a => state with { IsLoading = false, Error = a.Error },
        LogoutAction => UserState.Guest,
        _ => state
    };
}

public static class CartReducer
{
    public static CartState Reduce(CartState state, AppAction action) => action switch
    {
        AddToCartAction a => AddItem(state, a),
        RemoveFromCartAction a => state with
        {
            Items = state.Items.Where(i => i.ProductId != a.ProductId).ToList()
        },
        UpdateQuantityAction a => UpdateQuantity(state, a),
        ClearCartAction => CartState.Empty,
        _ => state
    };
    
    private static CartState AddItem(CartState state, AddToCartAction a)
    {
        var existing = state.Items.FirstOrDefault(i => i.ProductId == a.ProductId);
        if (existing != null)
        {
            return state with
            {
                Items = state.Items.Select(i =>
                    i.ProductId == a.ProductId
                        ? i with { Quantity = i.Quantity + 1 }
                        : i).ToList()
            };
        }
        
        return state with
        {
            Items = state.Items.Append(new CartItem(a.ProductId, a.Name, a.Price, 1)).ToList()
        };
    }
    
    private static CartState UpdateQuantity(CartState state, UpdateQuantityAction a)
    {
        if (a.Quantity <= 0)
            return state with { Items = state.Items.Where(i => i.ProductId != a.ProductId).ToList() };
        
        return state with
        {
            Items = state.Items.Select(i =>
                i.ProductId == a.ProductId
                    ? i with { Quantity = a.Quantity }
                    : i).ToList()
        };
    }
}
```

---

## Step 563: Redux Store

```csharp
// ============================================
// Store with Middleware
// ============================================

public class Store<TState>
{
    private TState _state;
    private readonly Func<TState, AppAction, TState> _reducer;
    private readonly List<Action<TState>> _subscribers = new();
    private readonly List<Func<Store<TState>, AppAction, Func<AppAction, Task>, Task>> _middleware;
    
    public TState State => _state;
    
    public Store(
        TState initialState,
        Func<TState, AppAction, TState> reducer,
        List<Func<Store<TState>, AppAction, Func<AppAction, Task>, Task>>? middleware = null)
    {
        _state = initialState;
        _reducer = reducer;
        _middleware = middleware ?? new();
    }
    
    public async Task DispatchAsync(AppAction action)
    {
        if (_middleware.Count == 0)
        {
            ApplyAction(action);
            return;
        }
        
        // Build middleware chain
        Func<AppAction, Task> next = async a =>
        {
            await Task.CompletedTask;
            ApplyAction(a);
        };
        
        foreach (var mw in _middleware.AsEnumerable().Reverse())
        {
            var current = next;
            var captured = mw;
            next = a => captured(this, a, current);
        }
        
        await next(action);
    }
    
    private void ApplyAction(AppAction action)
    {
        _state = _reducer(_state, action);
        foreach (var sub in _subscribers.ToList())
            sub(_state);
    }
    
    public IDisposable Subscribe(Action<TState> subscriber)
    {
        _subscribers.Add(subscriber);
        return new Subscription(() => _subscribers.Remove(subscriber));
    }
    
    private record Subscription(Action Dispose) : IDisposable
    {
        void IDisposable.Dispose() => Dispose();
    }
}

// Thunk middleware (async actions)
public static class ThunkMiddleware
{
    public static Func<Store<AppState>, AppAction, Func<AppAction, Task>, Task> Create()
    {
        return async (store, action, next) =>
        {
            if (action is AsyncAction asyncAction)
            {
                await asyncAction.ExecuteAsync(store.DispatchAsync);
                return;
            }
            await next(action);
        };
    }
}

public abstract record AsyncAction : AppAction
{
    public abstract Task ExecuteAsync(Func<AppAction, Task> dispatch);
}

// Logger middleware
public static class LoggerMiddleware
{
    public static Func<Store<AppState>, AppAction, Func<AppAction, Task>, Task> Create()
    {
        return async (store, action, next) =>
        {
            System.Diagnostics.Debug.WriteLine($"[Redux] Action: {action.GetType().Name}");
            await next(action);
            System.Diagnostics.Debug.WriteLine($"[Redux] New state: {store.State}");
        };
    }
}
```

---

## Step 564: Store ViewModel Integration

```csharp
// ============================================
// ViewModel using Redux Store
// ============================================

public partial class CartViewModel : ObservableObject, IDisposable
{
    private readonly Store<AppState> _store;
    private readonly IDisposable _subscription;
    
    [ObservableProperty] private List<CartItem> _items = new();
    [ObservableProperty] private decimal _total;
    [ObservableProperty] private int _itemCount;
    
    public CartViewModel(Store<AppState> store)
    {
        _store = store;
        _subscription = store.Subscribe(OnStateChanged);
        OnStateChanged(store.State);
    }
    
    private void OnStateChanged(AppState state)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            Items = state.Cart.Items;
            Total = state.Cart.Total;
            ItemCount = state.Cart.ItemCount;
        });
    }
    
    [RelayCommand]
    private async Task RemoveItemAsync(int productId)
        => await _store.DispatchAsync(new RemoveFromCartAction(productId));
    
    [RelayCommand]
    private async Task UpdateQuantityAsync((int id, int qty) args)
        => await _store.DispatchAsync(new UpdateQuantityAction(args.id, args.qty));
    
    [RelayCommand]
    private async Task ClearCartAsync()
        => await _store.DispatchAsync(new ClearCartAction());
    
    public void Dispose() => _subscription.Dispose();
}

// Async thunk action
public record LoginThunk(string Email, string Password) : AsyncAction
{
    public override async Task ExecuteAsync(Func<AppAction, Task> dispatch)
    {
        await dispatch(new LoginAction(Email, Password));
        
        try
        {
            // Call API
            var result = await LoginApiAsync(Email, Password);
            await dispatch(new LoginSuccessAction(result.UserId, result.Name, result.Email));
        }
        catch (Exception ex)
        {
            await dispatch(new LoginFailureAction(ex.Message));
        }
    }
    
    private static Task<LoginResult> LoginApiAsync(string email, string password)
        => throw new NotImplementedException();
}

public record LoginResult(int UserId, string Name, string Email);
```

---

## Step 565: State Machine

```csharp
// ============================================
// Stateful Machine Pattern
// ============================================

public enum OrderFlowState
{
    Cart, AddressSelection, PaymentSelection,
    Review, Processing, Success, Failed
}

public enum OrderFlowTrigger
{
    ProceedToAddress, ProceedToPayment, ReviewOrder,
    ConfirmOrder, PaymentSuccess, PaymentFailed,
    BackToCart, BackToAddress, Retry
}

public class OrderFlowStateMachine
{
    private OrderFlowState _current = OrderFlowState.Cart;
    
    private static readonly Dictionary<(OrderFlowState, OrderFlowTrigger), OrderFlowState> _transitions = new()
    {
        [(OrderFlowState.Cart, OrderFlowTrigger.ProceedToAddress)] = OrderFlowState.AddressSelection,
        [(OrderFlowState.AddressSelection, OrderFlowTrigger.ProceedToPayment)] = OrderFlowState.PaymentSelection,
        [(OrderFlowState.AddressSelection, OrderFlowTrigger.BackToCart)] = OrderFlowState.Cart,
        [(OrderFlowState.PaymentSelection, OrderFlowTrigger.ReviewOrder)] = OrderFlowState.Review,
        [(OrderFlowState.PaymentSelection, OrderFlowTrigger.BackToAddress)] = OrderFlowState.AddressSelection,
        [(OrderFlowState.Review, OrderFlowTrigger.ConfirmOrder)] = OrderFlowState.Processing,
        [(OrderFlowState.Review, OrderFlowTrigger.BackToAddress)] = OrderFlowState.AddressSelection,
        [(OrderFlowState.Processing, OrderFlowTrigger.PaymentSuccess)] = OrderFlowState.Success,
        [(OrderFlowState.Processing, OrderFlowTrigger.PaymentFailed)] = OrderFlowState.Failed,
        [(OrderFlowState.Failed, OrderFlowTrigger.Retry)] = OrderFlowState.Processing,
        [(OrderFlowState.Failed, OrderFlowTrigger.BackToCart)] = OrderFlowState.Cart,
    };
    
    public OrderFlowState Current => _current;
    
    public event Action<OrderFlowState, OrderFlowState>? StateChanged;
    
    public bool CanFire(OrderFlowTrigger trigger)
        => _transitions.ContainsKey((_current, trigger));
    
    public void Fire(OrderFlowTrigger trigger)
    {
        if (!_transitions.TryGetValue((_current, trigger), out var next))
            throw new InvalidOperationException(
                $"Cannot fire {trigger} from {_current}");
        
        var previous = _current;
        _current = next;
        StateChanged?.Invoke(previous, next);
    }
}

// ViewModel using state machine
public partial class CheckoutFlowViewModel : ObservableObject
{
    private readonly OrderFlowStateMachine _stateMachine = new();
    
    [ObservableProperty] private OrderFlowState _currentState = OrderFlowState.Cart;
    [ObservableProperty] private bool _isProcessing;
    
    public CheckoutFlowViewModel()
    {
        _stateMachine.StateChanged += (from, to) =>
        {
            CurrentState = to;
            IsProcessing = to == OrderFlowState.Processing;
        };
    }
    
    [RelayCommand(CanExecute = nameof(CanProceedToAddress))]
    private void ProceedToAddress() => _stateMachine.Fire(OrderFlowTrigger.ProceedToAddress);
    
    private bool CanProceedToAddress => _stateMachine.CanFire(OrderFlowTrigger.ProceedToAddress);
    
    [RelayCommand(CanExecute = nameof(CanConfirmOrder))]
    private async Task ConfirmOrderAsync()
    {
        _stateMachine.Fire(OrderFlowTrigger.ConfirmOrder);
        
        try
        {
            // await ProcessPaymentAsync();
            _stateMachine.Fire(OrderFlowTrigger.PaymentSuccess);
        }
        catch
        {
            _stateMachine.Fire(OrderFlowTrigger.PaymentFailed);
        }
    }
    
    private bool CanConfirmOrder => _stateMachine.CanFire(OrderFlowTrigger.ConfirmOrder);
}
```

---

## Step 566: Observable State

```csharp
// ============================================
// Observable State with ReactiveUI
// ============================================

public class ObservableAppState
{
    private readonly BehaviorSubject<UserState> _user = new(UserState.Guest);
    private readonly BehaviorSubject<CartState> _cart = new(CartState.Empty);
    
    // Observables
    public IObservable<UserState> UserStream => _user.AsObservable();
    public IObservable<CartState> CartStream => _cart.AsObservable();
    
    // Derived observables
    public IObservable<bool> IsAuthenticatedStream => 
        UserStream.Select(u => u.IsAuthenticated).DistinctUntilChanged();
    
    public IObservable<int> CartItemCountStream =>
        CartStream.Select(c => c.ItemCount).DistinctUntilChanged();
    
    // Mutations
    public void SetUser(UserState state) => _user.OnNext(state);
    public void UpdateCart(Func<CartState, CartState> update)
        => _cart.OnNext(update(_cart.Value));
    
    public void AddToCart(int productId, string name, decimal price)
    {
        UpdateCart(cart =>
        {
            var existing = cart.Items.FirstOrDefault(i => i.ProductId == productId);
            var items = existing != null
                ? cart.Items.Select(i => i.ProductId == productId
                    ? i with { Quantity = i.Quantity + 1 } : i).ToList()
                : cart.Items.Append(new CartItem(productId, name, price, 1)).ToList();
            
            return cart with { Items = items };
        });
    }
}

// ViewModel using observable state
public class CartBadgeViewModel : ObservableObject, IDisposable
{
    private readonly CompositeDisposable _disposables = new();
    
    [ObservableProperty] private int _itemCount;
    
    public CartBadgeViewModel(ObservableAppState state)
    {
        state.CartItemCountStream
            .ObserveOn(RxApp.MainThreadScheduler)
            .Subscribe(count => ItemCount = count)
            .DisposeWith(_disposables);
    }
    
    public void Dispose() => _disposables.Dispose();
}
```

---

## Step 567: Time-Travel Debugging

```csharp
// ============================================
// State History (Time-Travel Debug)
// ============================================

public class DebugStore<TState>
{
    private readonly Store<TState> _inner;
    private readonly List<(AppAction Action, TState State)> _history = new();
    private int _currentIndex = -1;
    
    public DebugStore(Store<TState> inner) => _inner = inner;
    
    public async Task DispatchAsync(AppAction action)
    {
        await _inner.DispatchAsync(action);
        
        // Trim future history if we're in past
        if (_currentIndex < _history.Count - 1)
            _history.RemoveRange(_currentIndex + 1, _history.Count - _currentIndex - 1);
        
        _history.Add((action, _inner.State));
        _currentIndex = _history.Count - 1;
    }
    
    public TState? TravelTo(int index)
    {
        if (index < 0 || index >= _history.Count) return default;
        _currentIndex = index;
        return _history[index].State;
    }
    
    public List<string> GetActionHistory()
        => _history.Select(h => h.Action.GetType().Name).ToList();
    
    public TState? Undo()
        => _currentIndex > 0 ? TravelTo(_currentIndex - 1) : default;
    
    public TState? Redo()
        => _currentIndex < _history.Count - 1 ? TravelTo(_currentIndex + 1) : default;
}
```

---

## Step 568: Undo/Redo

```csharp
// ============================================
// Undo/Redo for Forms
// ============================================

public class UndoStack<T>
{
    private readonly Stack<T> _undoStack = new();
    private readonly Stack<T> _redoStack = new();
    
    public bool CanUndo => _undoStack.Count > 0;
    public bool CanRedo => _redoStack.Count > 0;
    
    public void Push(T state)
    {
        _undoStack.Push(state);
        _redoStack.Clear();
    }
    
    public T? Undo(T current)
    {
        if (!CanUndo) return default;
        _redoStack.Push(current);
        return _undoStack.Pop();
    }
    
    public T? Redo(T current)
    {
        if (!CanRedo) return default;
        _undoStack.Push(current);
        return _redoStack.Pop();
    }
    
    public void Clear() { _undoStack.Clear(); _redoStack.Clear(); }
}

// Rich text editor with undo
public partial class EditorViewModel : ObservableObject
{
    private readonly UndoStack<string> _history = new();
    private string _text = string.Empty;
    
    [ObservableProperty] private string _displayText = string.Empty;
    [ObservableProperty] private bool _canUndo;
    [ObservableProperty] private bool _canRedo;
    
    public void UpdateText(string newText)
    {
        if (newText == _text) return;
        _history.Push(_text);
        _text = newText;
        DisplayText = newText;
        UpdateButtons();
    }
    
    [RelayCommand(CanExecute = nameof(CanUndo))]
    private void Undo()
    {
        var prev = _history.Undo(_text);
        if (prev != null) { _text = prev; DisplayText = prev; }
        UpdateButtons();
    }
    
    [RelayCommand(CanExecute = nameof(CanRedo))]
    private void Redo()
    {
        var next = _history.Redo(_text);
        if (next != null) { _text = next; DisplayText = next; }
        UpdateButtons();
    }
    
    private void UpdateButtons()
    {
        CanUndo = _history.CanUndo;
        CanRedo = _history.CanRedo;
    }
}
```

---

## Step 569: Persistent State

```csharp
// ============================================
// Persistent Application State
// ============================================

public class PersistentStateService
{
    private const string StateFile = "app_state.json";
    private readonly string _path;
    
    public PersistentStateService()
    {
        _path = Path.Combine(FileSystem.AppDataDirectory, StateFile);
    }
    
    public async Task SaveAsync<T>(T state)
    {
        var json = System.Text.Json.JsonSerializer.Serialize(state);
        await File.WriteAllTextAsync(_path, json);
    }
    
    public async Task<T?> LoadAsync<T>()
    {
        if (!File.Exists(_path)) return default;
        
        try
        {
            var json = await File.ReadAllTextAsync(_path);
            return System.Text.Json.JsonSerializer.Deserialize<T>(json);
        }
        catch { return default; }
    }
    
    public void Delete()
    {
        if (File.Exists(_path)) File.Delete(_path);
    }
}

// Hydrate store on startup
public class StoreHydrationService
{
    private readonly Store<AppState> _store;
    private readonly PersistentStateService _persistence;
    
    public StoreHydrationService(Store<AppState> store, PersistentStateService persistence)
    {
        _store = store;
        _persistence = persistence;
    }
    
    public async Task HydrateAsync()
    {
        var saved = await _persistence.LoadAsync<PersistedState>();
        if (saved == null) return;
        
        // Restore cart items
        foreach (var item in saved.Cart)
            await _store.DispatchAsync(new AddToCartAction(item.ProductId, item.Name, item.UnitPrice));
    }
    
    public async Task PersistAsync()
    {
        var state = new PersistedState(_store.State.Cart.Items);
        await _persistence.SaveAsync(state);
    }
}

public record PersistedState(List<CartItem> Cart);
```

---

## Step 570: State Testing

```csharp
// ============================================
// Testing Redux State
// ============================================

public class CartReducerTests
{
    [Fact]
    public void AddToCart_NewProduct_AddsItem()
    {
        var state = CartState.Empty;
        var action = new AddToCartAction(1, "สินค้า", 100m);
        
        var result = CartReducer.Reduce(state, action);
        
        result.Items.Should().HaveCount(1);
        result.Items[0].ProductId.Should().Be(1);
        result.Items[0].Quantity.Should().Be(1);
        result.Total.Should().Be(100m);
    }
    
    [Fact]
    public void AddToCart_ExistingProduct_IncreasesQuantity()
    {
        var state = new CartState(new List<CartItem>
        {
            new(1, "สินค้า", 100m, 2)
        });
        
        var result = CartReducer.Reduce(state, new AddToCartAction(1, "สินค้า", 100m));
        
        result.Items.Should().HaveCount(1);
        result.Items[0].Quantity.Should().Be(3);
        result.Total.Should().Be(300m);
    }
    
    [Fact]
    public void ClearCart_AlwaysEmpty()
    {
        var state = new CartState(new List<CartItem>
        {
            new(1, "A", 100m, 1),
            new(2, "B", 200m, 2)
        });
        
        var result = CartReducer.Reduce(state, new ClearCartAction());
        
        result.Items.Should().BeEmpty();
        result.Total.Should().Be(0);
    }
    
    [Fact]
    public void StateMachine_InvalidTransition_Throws()
    {
        var machine = new OrderFlowStateMachine();
        // Start in Cart state
        
        var act = () => machine.Fire(OrderFlowTrigger.ConfirmOrder);
        
        act.Should().Throw<InvalidOperationException>();
    }
    
    [Fact]
    public void StateMachine_ValidFlow_ReachesSuccess()
    {
        var machine = new OrderFlowStateMachine();
        
        machine.Fire(OrderFlowTrigger.ProceedToAddress);
        machine.Fire(OrderFlowTrigger.ProceedToPayment);
        machine.Fire(OrderFlowTrigger.ReviewOrder);
        machine.Fire(OrderFlowTrigger.ConfirmOrder);
        machine.Fire(OrderFlowTrigger.PaymentSuccess);
        
        machine.Current.Should().Be(OrderFlowState.Success);
    }
}
```

---

## สรุป Part 57

ใน Part 57 เราได้เรียนรู้:

1. **Redux State** - Immutable records, root state
2. **Actions & Reducers** - Pure functions
3. **Store with Middleware** - Thunk, Logger
4. **ViewModel Integration** - Subscribe, dispatch
5. **State Machine** - Order flow transitions
6. **Observable State** - ReactiveUI streams
7. **Time-Travel Debug** - History stack
8. **Undo/Redo** - UndoStack<T>
9. **Persistent State** - JSON serialization
10. **State Testing** - Reducer unit tests

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 57 | Steps 561-570*
