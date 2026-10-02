# Part 77: Real-Time Features with SignalR
## Steps 761-770: Live Updates, Chat, Live Order Tracking, WebSockets

---

## Step 761: SignalR Client Setup

```csharp
// ============================================
// SignalR Hub Connection
// ============================================

// Install: Microsoft.AspNetCore.SignalR.Client

public class SignalRConnectionService
{
    private HubConnection? _connection;
    private readonly ITokenService _tokens;
    
    public event EventHandler<ConnectionState>? StateChanged;
    
    public bool IsConnected =>
        _connection?.State == HubConnectionState.Connected;
    
    public SignalRConnectionService(ITokenService tokens)
    {
        _tokens = tokens;
    }
    
    public async Task ConnectAsync(string hubUrl, CancellationToken ct = default)
    {
        if (_connection != null) await DisconnectAsync();
        
        _connection = new HubConnectionBuilder()
            .WithUrl(hubUrl, options =>
            {
                options.AccessTokenProvider = async () =>
                    await _tokens.GetAccessTokenAsync();
            })
            .WithAutomaticReconnect(new[] {
                TimeSpan.Zero,
                TimeSpan.FromSeconds(2),
                TimeSpan.FromSeconds(5),
                TimeSpan.FromSeconds(15)
            })
            .Build();
        
        _connection.Reconnecting += _ =>
        {
            StateChanged?.Invoke(this, ConnectionState.Reconnecting);
            return Task.CompletedTask;
        };
        
        _connection.Reconnected += _ =>
        {
            StateChanged?.Invoke(this, ConnectionState.Connected);
            return Task.CompletedTask;
        };
        
        _connection.Closed += _ =>
        {
            StateChanged?.Invoke(this, ConnectionState.Disconnected);
            return Task.CompletedTask;
        };
        
        await _connection.StartAsync(ct);
        StateChanged?.Invoke(this, ConnectionState.Connected);
    }
    
    public Task OnAsync<T>(string method, Action<T> handler)
    {
        _connection?.On<T>(method, handler);
        return Task.CompletedTask;
    }
    
    public Task InvokeAsync(string method, object? arg = null, CancellationToken ct = default)
        => _connection?.InvokeAsync(method, arg, ct) ?? Task.CompletedTask;
    
    public async Task DisconnectAsync()
    {
        if (_connection != null)
        {
            await _connection.StopAsync();
            await _connection.DisposeAsync();
            _connection = null;
        }
    }
}

public enum ConnectionState { Connected, Reconnecting, Disconnected }
```

---

## Step 762: Live Order Tracking

```csharp
// ============================================
// Real-Time Order Status Updates
// ============================================

public class LiveOrderTracker : IDisposable
{
    private readonly SignalRConnectionService _signalR;
    private string? _currentOrderId;
    
    public event EventHandler<OrderStatusUpdate>? StatusUpdated;
    public event EventHandler<RiderLocation>? RiderMoved;
    public event EventHandler<string>? MessageReceived;
    
    public LiveOrderTracker(SignalRConnectionService signalR)
    {
        _signalR = signalR;
        
        _signalR.OnAsync<OrderStatusUpdate>("OrderStatusChanged",
            update => StatusUpdated?.Invoke(this, update));
        
        _signalR.OnAsync<RiderLocation>("RiderLocationUpdated",
            loc => RiderMoved?.Invoke(this, loc));
        
        _signalR.OnAsync<string>("OrderMessage",
            msg => MessageReceived?.Invoke(this, msg));
    }
    
    public async Task TrackOrderAsync(string orderId)
    {
        if (_currentOrderId != null)
            await _signalR.InvokeAsync("LeaveOrderGroup", _currentOrderId);
        
        _currentOrderId = orderId;
        await _signalR.InvokeAsync("JoinOrderGroup", orderId);
    }
    
    public async Task StopTrackingAsync()
    {
        if (_currentOrderId != null)
        {
            await _signalR.InvokeAsync("LeaveOrderGroup", _currentOrderId);
            _currentOrderId = null;
        }
    }
    
    public void Dispose()
        => StopTrackingAsync().ConfigureAwait(false);
}

public record OrderStatusUpdate(
    string OrderId,
    string Status,
    string Message,
    DateTime UpdatedAt,
    int EstimatedMinutes);

public record RiderLocation(
    string RiderId,
    double Latitude,
    double Longitude,
    double Heading,
    int EstimatedMinutes);
```

---

## Step 763: Live Order Tracking ViewModel

```csharp
// ============================================
// Live Tracking ViewModel with Map Updates
// ============================================

public partial class LiveTrackingViewModel : ObservableObject, IDisposable
{
    private readonly LiveOrderTracker _tracker;
    private readonly ISignalRConnectionService _signalR;
    
    [ObservableProperty] private string _status = "กำลังยืนยันออเดอร์";
    [ObservableProperty] private string _statusMessage = "";
    [ObservableProperty] private int _estimatedMinutes = 30;
    [ObservableProperty] private double _riderLatitude;
    [ObservableProperty] private double _riderLongitude;
    [ObservableProperty] private bool _isRiderVisible;
    [ObservableProperty] private string _connectionStatus = "";
    
    public LiveTrackingViewModel(LiveOrderTracker tracker, SignalRConnectionService signalR)
    {
        _tracker = tracker;
        
        _tracker.StatusUpdated += OnStatusUpdated;
        _tracker.RiderMoved += OnRiderMoved;
        
        signalR.StateChanged += OnConnectionStateChanged;
    }
    
    public async Task StartTrackingAsync(string orderId)
    {
        if (!_signalR.IsConnected)
            await _signalR.ConnectAsync("https://api.myapp.com/hubs/orders");
        
        await _tracker.TrackOrderAsync(orderId);
    }
    
    private void OnStatusUpdated(object? sender, OrderStatusUpdate update)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            Status = TranslateStatus(update.Status);
            StatusMessage = update.Message;
            EstimatedMinutes = update.EstimatedMinutes;
            
            if (update.Status == "Delivered")
                IsRiderVisible = false;
        });
    }
    
    private void OnRiderMoved(object? sender, RiderLocation loc)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            RiderLatitude = loc.Latitude;
            RiderLongitude = loc.Longitude;
            EstimatedMinutes = loc.EstimatedMinutes;
            IsRiderVisible = true;
        });
    }
    
    private void OnConnectionStateChanged(object? sender, ConnectionState state)
    {
        ConnectionStatus = state switch
        {
            ConnectionState.Connected => "",
            ConnectionState.Reconnecting => "กำลังเชื่อมต่อใหม่...",
            ConnectionState.Disconnected => "ออฟไลน์",
            _ => ""
        };
    }
    
    private static string TranslateStatus(string status) => status switch
    {
        "Placed" => "ได้รับออเดอร์แล้ว",
        "Confirmed" => "ร้านยืนยันออเดอร์",
        "Preparing" => "กำลังปรุงอาหาร",
        "ReadyForPickup" => "รออาหารให้ไรเดอร์",
        "PickedUp" => "ไรเดอร์รับอาหารแล้ว",
        "Delivered" => "จัดส่งสำเร็จ",
        _ => status
    };
    
    public void Dispose()
    {
        _tracker.StatusUpdated -= OnStatusUpdated;
        _tracker.RiderMoved -= OnRiderMoved;
        _tracker.Dispose();
    }
}
```

---

## Step 764: Live Chat System

```csharp
// ============================================
// In-App Chat with SignalR
// ============================================

public class ChatService
{
    private readonly SignalRConnectionService _signalR;
    private readonly SQLiteAsyncConnection _db;
    
    public event EventHandler<ChatMessage>? MessageReceived;
    public event EventHandler<string>? UserTyping;
    
    public ChatService(SignalRConnectionService signalR, SQLiteAsyncConnection db)
    {
        _signalR = signalR;
        _db = db;
        _db.CreateTableAsync<ChatMessage>().Wait();
        
        _signalR.OnAsync<ChatMessage>("ReceiveMessage", OnMessageReceived);
        _signalR.OnAsync<string>("UserTyping", userId => UserTyping?.Invoke(this, userId));
    }
    
    public async Task JoinRoomAsync(string roomId)
        => await _signalR.InvokeAsync("JoinRoom", roomId);
    
    public async Task SendMessageAsync(string roomId, string text)
    {
        var message = new ChatMessage
        {
            Id = Guid.NewGuid().ToString(),
            RoomId = roomId,
            Text = text,
            SentAt = DateTime.UtcNow,
            IsFromMe = true,
            Status = MessageStatus.Sending
        };
        
        // Optimistic: save locally first
        await _db.InsertAsync(message);
        MessageReceived?.Invoke(this, message);
        
        try
        {
            await _signalR.InvokeAsync("SendMessage", new { roomId, text });
            message.Status = MessageStatus.Sent;
            await _db.UpdateAsync(message);
        }
        catch
        {
            message.Status = MessageStatus.Failed;
            await _db.UpdateAsync(message);
        }
    }
    
    public async Task NotifyTypingAsync(string roomId)
        => await _signalR.InvokeAsync("NotifyTyping", roomId);
    
    private async void OnMessageReceived(ChatMessage message)
    {
        message.IsFromMe = false;
        message.Status = MessageStatus.Delivered;
        await _db.InsertOrReplaceAsync(message);
        
        MainThread.BeginInvokeOnMainThread(() =>
            MessageReceived?.Invoke(this, message));
    }
    
    public Task<List<ChatMessage>> GetHistoryAsync(string roomId)
        => _db.Table<ChatMessage>()
              .Where(m => m.RoomId == roomId)
              .OrderBy(m => m.SentAt)
              .ToListAsync();
}

public enum MessageStatus { Sending, Sent, Delivered, Read, Failed }

[Table("chat_messages")]
public class ChatMessage
{
    [PrimaryKey] public string Id { get; set; } = "";
    public string RoomId { get; set; } = "";
    public string Text { get; set; } = "";
    public DateTime SentAt { get; set; }
    public bool IsFromMe { get; set; }
    public MessageStatus Status { get; set; }
    [Ignore] public bool ShowTick => IsFromMe;
}
```

---

## Step 765: Chat ViewModel & XAML

```csharp
// Chat ViewModel
public partial class ChatViewModel : ObservableObject, IDisposable
{
    private readonly ChatService _chat;
    private readonly string _roomId;
    private CancellationTokenSource? _typingCts;
    
    [ObservableProperty] private ObservableCollection<ChatMessage> _messages = new();
    [ObservableProperty] private string _inputText = "";
    [ObservableProperty] private bool _isTypingVisible;
    [ObservableProperty] private bool _isSending;
    
    public ChatViewModel(ChatService chat, string roomId)
    {
        _chat = chat;
        _roomId = roomId;
        
        _chat.MessageReceived += OnMessageReceived;
        _chat.UserTyping += OnUserTyping;
    }
    
    public async Task InitializeAsync()
    {
        await _chat.JoinRoomAsync(_roomId);
        var history = await _chat.GetHistoryAsync(_roomId);
        Messages = new ObservableCollection<ChatMessage>(history);
    }
    
    [RelayCommand]
    private async Task SendAsync()
    {
        if (string.IsNullOrWhiteSpace(InputText)) return;
        
        IsSending = true;
        var text = InputText;
        InputText = "";
        
        try { await _chat.SendMessageAsync(_roomId, text); }
        finally { IsSending = false; }
    }
    
    partial void OnInputTextChanged(string value)
    {
        if (!string.IsNullOrEmpty(value))
        {
            _typingCts?.Cancel();
            _typingCts = new CancellationTokenSource();
            _ = NotifyTypingDebounced(_typingCts.Token);
        }
    }
    
    private async Task NotifyTypingDebounced(CancellationToken ct)
    {
        await Task.Delay(500, ct);
        if (!ct.IsCancellationRequested)
            await _chat.NotifyTypingAsync(_roomId);
    }
    
    private void OnMessageReceived(object? sender, ChatMessage msg)
        => Messages.Add(msg);
    
    private void OnUserTyping(object? sender, string userId)
    {
        IsTypingVisible = true;
        Dispatcher.Dispatch(async () =>
        {
            await Task.Delay(3000);
            IsTypingVisible = false;
        });
    }
    
    public void Dispose()
    {
        _chat.MessageReceived -= OnMessageReceived;
        _chat.UserTyping -= OnUserTyping;
    }
}
```

---

## Step 766: Live Restaurant Dashboard

```csharp
// ============================================
// Restaurant-Side Live Dashboard
// ============================================

public class RestaurantDashboard
{
    private readonly SignalRConnectionService _signalR;
    
    public event EventHandler<NewOrderEvent>? NewOrderArrived;
    public event EventHandler<int>? QueueLengthChanged;
    public event EventHandler<RevenueUpdate>? RevenueUpdated;
    
    public RestaurantDashboard(SignalRConnectionService signalR)
    {
        _signalR = signalR;
        
        _signalR.OnAsync<NewOrderEvent>("NewOrder",
            e => NewOrderArrived?.Invoke(this, e));
        _signalR.OnAsync<int>("QueueLength",
            n => QueueLengthChanged?.Invoke(this, n));
        _signalR.OnAsync<RevenueUpdate>("RevenueUpdate",
            r => RevenueUpdated?.Invoke(this, r));
    }
    
    public Task JoinRestaurantRoomAsync(string restaurantId)
        => _signalR.InvokeAsync("JoinRestaurant", restaurantId);
    
    public Task AcceptOrderAsync(string orderId)
        => _signalR.InvokeAsync("AcceptOrder", orderId);
    
    public Task MarkOrderReadyAsync(string orderId)
        => _signalR.InvokeAsync("MarkOrderReady", orderId);
}

public record NewOrderEvent(
    string OrderId, string CustomerName,
    List<string> Items, decimal Total,
    DateTime OrderedAt);

public record RevenueUpdate(
    decimal TodayTotal, int TodayOrders,
    decimal HourlyRate);
```

---

## Step 767: Presence System

```csharp
// ============================================
// Online Presence (Who's Online)
// ============================================

public class PresenceService
{
    private readonly SignalRConnectionService _signalR;
    private readonly Dictionary<string, UserPresence> _presences = new();
    
    public event EventHandler<UserPresence>? UserWentOnline;
    public event EventHandler<string>? UserWentOffline;
    
    public IReadOnlyDictionary<string, UserPresence> OnlineUsers => _presences;
    
    public PresenceService(SignalRConnectionService signalR)
    {
        _signalR = signalR;
        
        _signalR.OnAsync<UserPresence>("UserOnline", presence =>
        {
            _presences[presence.UserId] = presence;
            UserWentOnline?.Invoke(this, presence);
        });
        
        _signalR.OnAsync<string>("UserOffline", userId =>
        {
            _presences.Remove(userId);
            UserWentOffline?.Invoke(this, userId);
        });
    }
    
    public Task JoinPresenceAsync(string userId)
        => _signalR.InvokeAsync("RegisterPresence", userId);
    
    public bool IsOnline(string userId)
        => _presences.ContainsKey(userId);
}

public record UserPresence(
    string UserId,
    string DisplayName,
    DateTime ConnectedAt,
    string? AvatarUrl);
```

---

## Step 768: Real-Time Notifications

```csharp
// ============================================
// Server-Pushed Notifications via SignalR
// ============================================

public class PushNotificationHub
{
    private readonly SignalRConnectionService _signalR;
    private readonly LocalNotificationService _localNotifs;
    
    public PushNotificationHub(SignalRConnectionService signalR,
        LocalNotificationService localNotifs)
    {
        _signalR = signalR;
        _localNotifs = localNotifs;
        
        _signalR.OnAsync<ServerNotification>("PushNotification",
            notification => OnNotificationReceived(notification));
    }
    
    private void OnNotificationReceived(ServerNotification notification)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            // Show in-app banner if app is foreground
            WeakReferenceMessenger.Default.Send(notification);
            
            // Show local notification if app is background
            if (!Application.Current!.Windows.Any(w => w.IsActivated))
                _localNotifs.Show(notification.Title, notification.Body,
                    notification.Data);
        });
    }
}

public class ServerNotification
{
    public string Id { get; set; } = "";
    public string Title { get; set; } = "";
    public string Body { get; set; } = "";
    public string Type { get; set; } = "";
    public Dictionary<string, string> Data { get; set; } = new();
}

public class LocalNotificationService
{
    public void Show(string title, string body, Dictionary<string, string> data)
    {
        // Platform implementation
        // Android: NotificationManager
        // iOS: UNUserNotificationCenter
    }
}
```

---

## Step 769: WebSocket Fallback

```csharp
// ============================================
// WebSocket Fallback for SignalR
// ============================================

public class HubConnectionFactory
{
    public static HubConnection CreateWithFallback(
        string hubUrl,
        Func<Task<string?>> accessTokenProvider)
    {
        return new HubConnectionBuilder()
            .WithUrl(hubUrl, options =>
            {
                options.AccessTokenProvider = accessTokenProvider;
                
                // Try transports in order: WebSockets → Server-Sent Events → Long Polling
                options.Transports =
                    HttpTransportType.WebSockets |
                    HttpTransportType.ServerSentEvents |
                    HttpTransportType.LongPolling;
                
                // Skip SSL verification in DEBUG
#if DEBUG
                options.HttpMessageHandlerFactory = _ =>
                    new HttpClientHandler
                    {
                        ServerCertificateCustomValidationCallback = (_, _, _, _) => true
                    };
#endif
            })
            .WithAutomaticReconnect(new RetryPolicy())
            .ConfigureLogging(logging =>
            {
#if DEBUG
                logging.SetMinimumLevel(LogLevel.Debug);
#endif
            })
            .Build();
    }
}

public class RetryPolicy : IRetryPolicy
{
    private static readonly TimeSpan[] _delays =
    {
        TimeSpan.Zero,
        TimeSpan.FromSeconds(2),
        TimeSpan.FromSeconds(5),
        TimeSpan.FromSeconds(15),
        TimeSpan.FromSeconds(30),
        TimeSpan.FromMinutes(1),
    };
    
    public TimeSpan? NextRetryDelay(RetryContext ctx)
    {
        if (ctx.PreviousRetryCount >= _delays.Length) return null; // Stop retrying
        return _delays[ctx.PreviousRetryCount];
    }
}
```

---

## Step 770: SignalR Tests

```csharp
// ============================================
// SignalR Integration Tests
// ============================================

[TestFixture]
public class SignalRChatTests
{
    private FakeChatService _fakeChat = null!;
    private ChatViewModel _vm = null!;
    
    [SetUp]
    public void SetUp()
    {
        _fakeChat = new FakeChatService();
        _vm = new ChatViewModel(_fakeChat, "room-123");
    }
    
    [Test]
    public async Task SendMessage_AddsToCollection()
    {
        _vm.InputText = "สวัสดี";
        await _vm.SendCommand.ExecuteAsync(null);
        
        Assert.That(_vm.Messages.Count, Is.EqualTo(1));
        Assert.That(_vm.Messages[0].Text, Is.EqualTo("สวัสดี"));
        Assert.That(_vm.Messages[0].IsFromMe, Is.True);
    }
    
    [Test]
    public async Task ReceiveMessage_AddsToCollection()
    {
        await _vm.InitializeAsync();
        
        _fakeChat.SimulateReceive(new ChatMessage
        {
            Id = "msg-1",
            RoomId = "room-123",
            Text = "ออเดอร์พร้อมแล้ว",
            IsFromMe = false
        });
        
        Assert.That(_vm.Messages.Any(m => m.Text == "ออเดอร์พร้อมแล้ว"), Is.True);
    }
    
    [Test]
    public void TypingIndicator_ShowsWhenUserTypes()
    {
        _fakeChat.SimulateTyping("user-456");
        Assert.That(_vm.IsTypingVisible, Is.True);
    }
}

public class FakeChatService : ChatService
{
    public FakeChatService() : base(null!, null!) { }
    
    public void SimulateReceive(ChatMessage msg)
        => MessageReceived?.Invoke(this, msg);
    
    public void SimulateTyping(string userId)
        => UserTyping?.Invoke(this, userId);
}
```

---

## สรุป Part 77

ใน Part 77 เราได้เรียนรู้:

1. **SignalR Client** - HubConnection, automatic reconnect, state events
2. **Live Order Tracking** - Group join/leave, status updates, rider location
3. **Tracking ViewModel** - Map pin updates, status translation to Thai
4. **Chat System** - Rooms, optimistic message send, typing notification
5. **Chat ViewModel** - Input debouncing, history loading, typing indicator
6. **Restaurant Dashboard** - New orders, queue length, revenue updates
7. **Presence System** - Online/offline events, presence dictionary
8. **Real-Time Notifications** - Server-push to in-app banner or local notif
9. **WebSocket Fallback** - Transport negotiation order, custom retry policy
10. **SignalR Tests** - Fake service, message receive, typing simulation

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 77 | Steps 761-770*
