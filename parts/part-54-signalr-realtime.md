# Part 54: SignalR & Real-Time Features
## Steps 531-540: Live Chat, Notifications, Collaboration

---

## Step 531: SignalR Hub Client

```csharp
// ============================================
// SignalR Client Setup
// ============================================

// dotnet add package Microsoft.AspNetCore.SignalR.Client

public class SignalRConnection : IAsyncDisposable
{
    private HubConnection? _connection;
    private readonly string _hubUrl;
    private readonly AuthTokenService _tokens;
    private CancellationTokenSource? _reconnectCts;
    
    public event Action<string>? Connected;
    public event Action? Disconnected;
    public event Action<string>? Reconnecting;
    
    public HubConnectionState State => _connection?.State ?? HubConnectionState.Disconnected;
    
    public SignalRConnection(string hubUrl, AuthTokenService tokens)
    {
        _hubUrl = hubUrl;
        _tokens = tokens;
    }
    
    public async Task StartAsync(CancellationToken ct = default)
    {
        _connection = new HubConnectionBuilder()
            .WithUrl(_hubUrl, options =>
            {
                options.AccessTokenProvider = async () =>
                    await _tokens.GetValidTokenAsync();
            })
            .WithAutomaticReconnect(new RetryPolicy())
            .AddJsonProtocol()
            .Build();
        
        _connection.Reconnecting += error =>
        {
            Reconnecting?.Invoke(error?.Message ?? "Reconnecting...");
            return Task.CompletedTask;
        };
        
        _connection.Reconnected += connectionId =>
        {
            Connected?.Invoke(connectionId ?? string.Empty);
            return Task.CompletedTask;
        };
        
        _connection.Closed += error =>
        {
            Disconnected?.Invoke();
            return Task.CompletedTask;
        };
        
        await _connection.StartAsync(ct);
        Connected?.Invoke(_connection.ConnectionId ?? string.Empty);
    }
    
    public IDisposable On<T>(string method, Action<T> handler)
        => _connection!.On(method, handler);
    
    public Task InvokeAsync(string method, params object[] args)
        => _connection!.InvokeAsync(method, args);
    
    public Task<TResult> InvokeAsync<TResult>(string method, params object[] args)
        => _connection!.InvokeAsync<TResult>(method, args);
    
    public async ValueTask DisposeAsync()
    {
        _reconnectCts?.Cancel();
        if (_connection != null)
            await _connection.DisposeAsync();
    }
}

// Exponential backoff retry policy
public class RetryPolicy : IRetryPolicy
{
    private static readonly TimeSpan[] _delays =
    {
        TimeSpan.Zero,
        TimeSpan.FromSeconds(2),
        TimeSpan.FromSeconds(10),
        TimeSpan.FromSeconds(30),
        TimeSpan.FromMinutes(1),
        null! // stop retrying
    };
    
    public TimeSpan? NextRetryDelay(RetryContext context)
    {
        var idx = (int)context.PreviousRetryCount;
        return idx < _delays.Length ? _delays[idx] : null;
    }
}
```

---

## Step 532: Live Chat

```csharp
// ============================================
// Live Chat Implementation
// ============================================

public partial class ChatViewModel : ObservableObject, IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    private readonly ICurrentUserService _user;
    private readonly IDisposable _messageSubscription;
    private readonly IDisposable _typingSubscription;
    
    [ObservableProperty] private ObservableCollection<ChatMessage> _messages = new();
    [ObservableProperty] private string _messageText = string.Empty;
    [ObservableProperty] private bool _isConnected;
    [ObservableProperty] private string? _typingUser;
    [ObservableProperty] private bool _isSending;
    
    public string RoomId { get; set; } = string.Empty;
    
    public ChatViewModel(SignalRConnection signalR, ICurrentUserService user)
    {
        _signalR = signalR;
        _user = user;
        
        _messageSubscription = _signalR.On<ChatMessage>("ReceiveMessage", OnMessageReceived);
        _typingSubscription = _signalR.On<TypingNotification>("UserTyping", OnUserTyping);
        
        _signalR.Connected += _ => IsConnected = true;
        _signalR.Disconnected += () => IsConnected = false;
    }
    
    public async Task JoinRoomAsync()
    {
        await _signalR.StartAsync();
        await _signalR.InvokeAsync("JoinRoom", RoomId);
        
        // Load history
        var history = await _signalR.InvokeAsync<List<ChatMessage>>(
            "GetHistory", RoomId, 50);
        
        foreach (var msg in history)
            Messages.Add(msg);
    }
    
    [RelayCommand]
    private async Task SendMessageAsync()
    {
        if (string.IsNullOrWhiteSpace(MessageText)) return;
        
        IsSending = true;
        var text = MessageText;
        MessageText = string.Empty;
        
        try
        {
            await _signalR.InvokeAsync("SendMessage", RoomId, text);
        }
        catch
        {
            MessageText = text; // restore on failure
        }
        finally { IsSending = false; }
    }
    
    private CancellationTokenSource? _typingCts;
    
    partial void OnMessageTextChanged(string value)
    {
        // Notify typing
        _typingCts?.Cancel();
        _typingCts = new CancellationTokenSource();
        
        if (!string.IsNullOrEmpty(value))
        {
            _ = _signalR.InvokeAsync("NotifyTyping", RoomId, true);
            
            Task.Delay(3000, _typingCts.Token).ContinueWith(t =>
            {
                if (!t.IsCanceled)
                    _ = _signalR.InvokeAsync("NotifyTyping", RoomId, false);
            });
        }
    }
    
    private void OnMessageReceived(ChatMessage message)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            Messages.Add(message);
        });
    }
    
    private void OnUserTyping(TypingNotification notification)
    {
        if (notification.UserId == _user.Current?.UserId) return;
        
        MainThread.BeginInvokeOnMainThread(() =>
        {
            TypingUser = notification.IsTyping ? notification.UserName : null;
        });
    }
    
    public async ValueTask DisposeAsync()
    {
        _messageSubscription.Dispose();
        _typingSubscription.Dispose();
        await _signalR.InvokeAsync("LeaveRoom", RoomId);
        await _signalR.DisposeAsync();
    }
}

public record ChatMessage(
    string Id, int UserId, string UserName,
    string Text, DateTime SentAt, bool IsOwn);

public record TypingNotification(int UserId, string UserName, bool IsTyping);
```

---

## Step 533: Real-Time Order Tracking

```csharp
// ============================================
// Order Status Real-Time Updates
// ============================================

public partial class OrderTrackingViewModel : ObservableObject, IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    private IDisposable? _statusSubscription;
    private IDisposable? _locationSubscription;
    
    [ObservableProperty] private OrderStatus _currentStatus;
    [ObservableProperty] private Location? _riderLocation;
    [ObservableProperty] private string? _statusMessage;
    [ObservableProperty] private DateTime? _estimatedArrival;
    [ObservableProperty] private int _progressPercent;
    
    public int OrderId { get; set; }
    
    public OrderTrackingViewModel(SignalRConnection signalR) => _signalR = signalR;
    
    public async Task StartTrackingAsync()
    {
        await _signalR.StartAsync();
        
        _statusSubscription = _signalR.On<OrderStatusUpdate>(
            "OrderStatusChanged", OnStatusChanged);
        
        _locationSubscription = _signalR.On<RiderLocationUpdate>(
            "RiderLocationUpdated", OnLocationUpdated);
        
        await _signalR.InvokeAsync("TrackOrder", OrderId);
    }
    
    private void OnStatusChanged(OrderStatusUpdate update)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            CurrentStatus = update.Status;
            StatusMessage = GetStatusMessage(update.Status);
            EstimatedArrival = update.EstimatedArrival;
            ProgressPercent = GetProgress(update.Status);
        });
    }
    
    private void OnLocationUpdated(RiderLocationUpdate update)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            RiderLocation = new Location(update.Latitude, update.Longitude);
        });
    }
    
    private static string GetStatusMessage(OrderStatus status) => status switch
    {
        OrderStatus.Confirmed => "ร้านค้ารับออเดอร์แล้ว",
        OrderStatus.Preparing => "กำลังเตรียมอาหาร",
        OrderStatus.PickedUp => "ไรเดอร์รับอาหารแล้ว",
        OrderStatus.OnTheWay => "กำลังจัดส่ง",
        OrderStatus.Delivered => "ส่งแล้ว 🎉",
        _ => "กำลังดำเนินการ"
    };
    
    private static int GetProgress(OrderStatus status) => status switch
    {
        OrderStatus.Confirmed => 20,
        OrderStatus.Preparing => 40,
        OrderStatus.PickedUp => 60,
        OrderStatus.OnTheWay => 80,
        OrderStatus.Delivered => 100,
        _ => 0
    };
    
    public async ValueTask DisposeAsync()
    {
        _statusSubscription?.Dispose();
        _locationSubscription?.Dispose();
        await _signalR.DisposeAsync();
    }
}

public record OrderStatusUpdate(int OrderId, OrderStatus Status, DateTime? EstimatedArrival);
public record RiderLocationUpdate(int OrderId, double Latitude, double Longitude);
```

---

## Step 534: Collaborative Editing

```csharp
// ============================================
// Real-Time Collaborative Form Editing
// ============================================

public partial class CollaborativeEditViewModel : ObservableObject, IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    private readonly ICurrentUserService _user;
    private readonly string _documentId;
    
    [ObservableProperty] private string _title = string.Empty;
    [ObservableProperty] private string _content = string.Empty;
    [ObservableProperty] private ObservableCollection<ActiveEditor> _activeEditors = new();
    [ObservableProperty] private bool _isLocked;
    [ObservableProperty] private string? _lockedBy;
    
    private bool _isRemoteChange;
    
    public CollaborativeEditViewModel(
        SignalRConnection signalR, ICurrentUserService user, string documentId)
    {
        _signalR = signalR;
        _user = user;
        _documentId = documentId;
    }
    
    public async Task InitializeAsync()
    {
        await _signalR.StartAsync();
        
        _signalR.On<string>("TitleChanged", text =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                _isRemoteChange = true;
                Title = text;
                _isRemoteChange = false;
            });
        });
        
        _signalR.On<string>("ContentChanged", text =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                _isRemoteChange = true;
                Content = text;
                _isRemoteChange = false;
            });
        });
        
        _signalR.On<ActiveEditor>("EditorJoined", editor =>
            MainThread.BeginInvokeOnMainThread(() => ActiveEditors.Add(editor)));
        
        _signalR.On<int>("EditorLeft", userId =>
            MainThread.BeginInvokeOnMainThread(() =>
            {
                var editor = ActiveEditors.FirstOrDefault(e => e.UserId == userId);
                if (editor != null) ActiveEditors.Remove(editor);
            }));
        
        _signalR.On<DocumentLock>("DocumentLocked", l =>
            MainThread.BeginInvokeOnMainThread(() =>
            {
                IsLocked = l.IsLocked;
                LockedBy = l.LockedByName;
            }));
        
        await _signalR.InvokeAsync("JoinDocument", _documentId);
    }
    
    private CancellationTokenSource? _titleDebounce;
    
    partial void OnTitleChanged(string value)
    {
        if (_isRemoteChange) return;
        
        _titleDebounce?.Cancel();
        _titleDebounce = new CancellationTokenSource();
        
        Task.Delay(500, _titleDebounce.Token).ContinueWith(t =>
        {
            if (!t.IsCanceled)
                _ = _signalR.InvokeAsync("UpdateTitle", _documentId, value);
        });
    }
    
    public async ValueTask DisposeAsync()
    {
        await _signalR.InvokeAsync("LeaveDocument", _documentId);
        await _signalR.DisposeAsync();
    }
}

public record ActiveEditor(int UserId, string Name, string Color);
public record DocumentLock(bool IsLocked, int? LockedByUserId, string? LockedByName);
```

---

## Step 535: Live Dashboard

```csharp
// ============================================
// Real-Time Dashboard
// ============================================

public partial class LiveDashboardViewModel : ObservableObject, IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    
    [ObservableProperty] private int _activeUsers;
    [ObservableProperty] private decimal _todayRevenue;
    [ObservableProperty] private int _pendingOrders;
    [ObservableProperty] private ObservableCollection<LiveOrder> _recentOrders = new();
    [ObservableProperty] private ObservableCollection<DataPoint> _revenueChart = new();
    
    public LiveDashboardViewModel(SignalRConnection signalR) => _signalR = signalR;
    
    public async Task StartAsync()
    {
        await _signalR.StartAsync();
        
        _signalR.On<int>("ActiveUsersUpdated", count =>
            MainThread.BeginInvokeOnMainThread(() => ActiveUsers = count));
        
        _signalR.On<decimal>("RevenueUpdated", amount =>
            MainThread.BeginInvokeOnMainThread(() =>
            {
                TodayRevenue = amount;
                RevenueChart.Add(new DataPoint(DateTime.Now, amount));
                if (RevenueChart.Count > 60) RevenueChart.RemoveAt(0);
            }));
        
        _signalR.On<int>("PendingOrdersChanged", count =>
            MainThread.BeginInvokeOnMainThread(() => PendingOrders = count));
        
        _signalR.On<LiveOrder>("NewOrder", order =>
            MainThread.BeginInvokeOnMainThread(() =>
            {
                RecentOrders.Insert(0, order);
                if (RecentOrders.Count > 20) RecentOrders.RemoveAt(20);
            }));
        
        await _signalR.InvokeAsync("SubscribeDashboard");
        
        // Load initial data
        var snapshot = await _signalR.InvokeAsync<DashboardSnapshot>("GetSnapshot");
        MainThread.BeginInvokeOnMainThread(() =>
        {
            ActiveUsers = snapshot.ActiveUsers;
            TodayRevenue = snapshot.TodayRevenue;
            PendingOrders = snapshot.PendingOrders;
        });
    }
    
    public async ValueTask DisposeAsync()
    {
        await _signalR.InvokeAsync("UnsubscribeDashboard");
        await _signalR.DisposeAsync();
    }
}

public record LiveOrder(int Id, string CustomerName, decimal Amount, DateTime At);
public record DashboardSnapshot(int ActiveUsers, decimal TodayRevenue, int PendingOrders);
public record DataPoint(DateTime Time, decimal Value);
```

---

## Step 536: Presence System

```csharp
// ============================================
// User Presence (Online/Offline/Away)
// ============================================

public class PresenceService : IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    private System.Timers.Timer? _heartbeatTimer;
    
    public event Action<UserPresenceUpdate>? PresenceChanged;
    
    public PresenceService(SignalRConnection signalR) => _signalR = signalR;
    
    public async Task StartAsync()
    {
        _signalR.On<UserPresenceUpdate>("PresenceChanged", update =>
        {
            MainThread.BeginInvokeOnMainThread(() => PresenceChanged?.Invoke(update));
        });
        
        // Start heartbeat
        _heartbeatTimer = new System.Timers.Timer(30_000); // 30s
        _heartbeatTimer.Elapsed += async (_, _) =>
            await _signalR.InvokeAsync("Heartbeat");
        _heartbeatTimer.Start();
        
        await _signalR.InvokeAsync("SetPresence", "online");
    }
    
    public async Task SetAwayAsync()
        => await _signalR.InvokeAsync("SetPresence", "away");
    
    public async Task<List<UserPresence>> GetTeamPresenceAsync(int teamId)
        => await _signalR.InvokeAsync<List<UserPresence>>("GetTeamPresence", teamId);
    
    public async ValueTask DisposeAsync()
    {
        _heartbeatTimer?.Dispose();
        await _signalR.InvokeAsync("SetPresence", "offline");
        await _signalR.DisposeAsync();
    }
}

public record UserPresence(int UserId, string Name, string Status, DateTime LastSeen);
public record UserPresenceUpdate(int UserId, string Name, string Status);
```

---

## Step 537: Stream-Based SignalR

```csharp
// ============================================
// Server-Side Streaming
// ============================================

public class StreamingService
{
    private readonly SignalRConnection _signalR;
    
    public StreamingService(SignalRConnection signalR) => _signalR = signalR;
    
    public async IAsyncEnumerable<StockPrice> StreamPricesAsync(
        string[] symbols, [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        var channel = await _signalR.InvokeAsync<System.Threading.Channels.ChannelReader<StockPrice>>(
            "StreamStockPrices", symbols);
        
        await foreach (var price in channel.ReadAllAsync(ct))
            yield return price;
    }
    
    // ViewModel using streaming
    public partial class StockViewModel : ObservableObject
    {
        private readonly StreamingService _streaming;
        
        [ObservableProperty]
        private ObservableCollection<StockPrice> _prices = new();
        
        private CancellationTokenSource? _streamCts;
        
        public StockViewModel(StreamingService streaming) => _streaming = streaming;
        
        public async Task StartStreamingAsync(string[] symbols)
        {
            _streamCts = new CancellationTokenSource();
            
            await foreach (var price in _streaming.StreamPricesAsync(
                symbols, _streamCts.Token))
            {
                var existing = Prices.FirstOrDefault(p => p.Symbol == price.Symbol);
                if (existing != null)
                {
                    var idx = Prices.IndexOf(existing);
                    Prices[idx] = price;
                }
                else
                {
                    Prices.Add(price);
                }
            }
        }
        
        public void StopStreaming() => _streamCts?.Cancel();
    }
}

public record StockPrice(string Symbol, decimal Price, decimal Change, DateTime At);
```

---

## Step 538: Notification Hub

```csharp
// ============================================
// In-App Notification System via SignalR
// ============================================

public partial class NotificationHubViewModel : ObservableObject, IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    
    [ObservableProperty] private ObservableCollection<InAppNotification> _notifications = new();
    [ObservableProperty] private int _unreadCount;
    
    public NotificationHubViewModel(SignalRConnection signalR) => _signalR = signalR;
    
    public async Task ConnectAsync()
    {
        await _signalR.StartAsync();
        
        _signalR.On<InAppNotification>("NewNotification", notification =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                Notifications.Insert(0, notification);
                UnreadCount++;
                
                // Show local notification banner
                ShowBanner(notification);
            });
        });
        
        // Load unread
        var unread = await _signalR.InvokeAsync<List<InAppNotification>>(
            "GetUnreadNotifications");
        
        MainThread.BeginInvokeOnMainThread(() =>
        {
            foreach (var n in unread)
                Notifications.Add(n);
            UnreadCount = unread.Count;
        });
    }
    
    [RelayCommand]
    private async Task MarkAllReadAsync()
    {
        var ids = Notifications.Where(n => !n.IsRead).Select(n => n.Id).ToList();
        await _signalR.InvokeAsync("MarkAsRead", ids);
        
        foreach (var n in Notifications.Where(n => !n.IsRead))
            n.IsRead = true;
        
        UnreadCount = 0;
    }
    
    private static void ShowBanner(InAppNotification notification)
    {
        // Use a custom overlay or CommunityToolkit.Maui Toast
        MainThread.BeginInvokeOnMainThread(async () =>
        {
            await Shell.Current.DisplaySnackbar(
                notification.Message,
                notification.Action != null ? "ดู" : null,
                async () => await NavigateToNotificationAsync(notification));
        });
    }
    
    private static Task NavigateToNotificationAsync(InAppNotification n)
        => n.Action != null
            ? Shell.Current.GoToAsync(n.Action)
            : Task.CompletedTask;
    
    public async ValueTask DisposeAsync() => await _signalR.DisposeAsync();
}

public class InAppNotification : ObservableObject
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public string Type { get; set; } = string.Empty;
    public string? Action { get; set; }
    public DateTime CreatedAt { get; set; }
    
    private bool _isRead;
    public bool IsRead
    {
        get => _isRead;
        set => SetProperty(ref _isRead, value);
    }
}
```

---

## Step 539: Multiplayer Game Feature

```csharp
// ============================================
// Simple Real-Time Quiz / Game
// ============================================

public partial class QuizGameViewModel : ObservableObject, IAsyncDisposable
{
    private readonly SignalRConnection _signalR;
    
    [ObservableProperty] private QuizQuestion? _currentQuestion;
    [ObservableProperty] private int _score;
    [ObservableProperty] private int _timeLeft;
    [ObservableProperty] private ObservableCollection<PlayerScore> _leaderboard = new();
    [ObservableProperty] private GameState _state = GameState.Waiting;
    [ObservableProperty] private string? _selectedAnswer;
    [ObservableProperty] private bool _hasAnswered;
    
    private System.Timers.Timer? _countdownTimer;
    
    public string RoomCode { get; set; } = string.Empty;
    
    public QuizGameViewModel(SignalRConnection signalR) => _signalR = signalR;
    
    public async Task JoinGameAsync()
    {
        await _signalR.StartAsync();
        
        _signalR.On<QuizQuestion>("QuestionStarted", q =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                CurrentQuestion = q;
                SelectedAnswer = null;
                HasAnswered = false;
                TimeLeft = q.TimeLimit;
                State = GameState.InProgress;
                StartCountdown();
            });
        });
        
        _signalR.On<QuizResult>("QuestionEnded", result =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                StopCountdown();
                if (SelectedAnswer == result.CorrectAnswer) Score += result.Points;
                State = GameState.ShowingResult;
            });
        });
        
        _signalR.On<List<PlayerScore>>("LeaderboardUpdated", scores =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                Leaderboard.Clear();
                foreach (var s in scores) Leaderboard.Add(s);
            });
        });
        
        _signalR.On<object>("GameEnded", _ =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                State = GameState.Ended;
                StopCountdown();
            });
        });
        
        await _signalR.InvokeAsync("JoinGame", RoomCode);
    }
    
    [RelayCommand]
    private async Task AnswerAsync(string answer)
    {
        if (HasAnswered) return;
        
        HasAnswered = true;
        SelectedAnswer = answer;
        
        await _signalR.InvokeAsync("SubmitAnswer", RoomCode,
            CurrentQuestion!.Id, answer, TimeLeft);
    }
    
    private void StartCountdown()
    {
        _countdownTimer = new System.Timers.Timer(1000);
        _countdownTimer.Elapsed += (_, _) =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                TimeLeft--;
                if (TimeLeft <= 0) StopCountdown();
            });
        };
        _countdownTimer.Start();
    }
    
    private void StopCountdown() { _countdownTimer?.Stop(); _countdownTimer?.Dispose(); }
    
    public async ValueTask DisposeAsync() => await _signalR.DisposeAsync();
}

public record QuizQuestion(string Id, string Text, List<string> Options, int TimeLimit);
public record QuizResult(string QuestionId, string CorrectAnswer, int Points);
public record PlayerScore(string Name, int Score, int Rank);
public enum GameState { Waiting, InProgress, ShowingResult, Ended }
```

---

## Step 540: SignalR Reconnection & Resilience

```csharp
// ============================================
// Resilient SignalR with State Recovery
// ============================================

public abstract class ResilientHubClient<TState> : IAsyncDisposable
    where TState : class
{
    protected readonly SignalRConnection Connection;
    private TState? _lastKnownState;
    
    protected ResilientHubClient(SignalRConnection connection)
    {
        Connection = connection;
        Connection.Reconnected += OnReconnectedAsync;
    }
    
    private async void OnReconnectedAsync(string connectionId)
    {
        // Re-subscribe and restore state
        await RegisterHandlersAsync();
        await RestoreStateAsync(_lastKnownState);
    }
    
    protected void SaveState(TState state) => _lastKnownState = state;
    
    protected abstract Task RegisterHandlersAsync();
    protected abstract Task RestoreStateAsync(TState? lastState);
    
    public async ValueTask DisposeAsync() => await Connection.DisposeAsync();
}

// Concrete: Chat with reconnect
public class ResilientChatClient : ResilientHubClient<ChatState>
{
    public string RoomId { get; }
    public event Action<ChatMessage>? MessageReceived;
    
    public ResilientChatClient(SignalRConnection connection, string roomId)
        : base(connection) => RoomId = roomId;
    
    protected override async Task RegisterHandlersAsync()
    {
        Connection.On<ChatMessage>("ReceiveMessage", msg =>
        {
            SaveState(new ChatState(msg.Id));
            MessageReceived?.Invoke(msg);
        });
        
        await Connection.InvokeAsync("JoinRoom", RoomId);
    }
    
    protected override async Task RestoreStateAsync(ChatState? lastState)
    {
        if (lastState == null) return;
        
        // Fetch missed messages since last known
        var missed = await Connection.InvokeAsync<List<ChatMessage>>(
            "GetMessagesSince", RoomId, lastState.LastMessageId);
        
        foreach (var msg in missed)
            MessageReceived?.Invoke(msg);
    }
    
    public Task SendAsync(string text)
        => Connection.InvokeAsync("SendMessage", RoomId, text);
}

public record ChatState(string LastMessageId);
```

---

## สรุป Part 54

ใน Part 54 เราได้เรียนรู้:

1. **SignalR Hub Client** - Retry policy, auto-reconnect
2. **Live Chat** - Typing indicators, message history
3. **Order Tracking** - Real-time status + rider location
4. **Collaborative Editing** - Multi-user document
5. **Live Dashboard** - Revenue, orders, active users
6. **Presence System** - Online/offline/away heartbeat
7. **Server Streaming** - ChannelReader<T> stock prices
8. **Notification Hub** - In-app banners via SignalR
9. **Multiplayer Quiz** - Real-time game with countdown
10. **Resilient Client** - State recovery on reconnect

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 54 | Steps 531-540*
