# Part 28: SignalR Real-time Communication
## Steps 271-280: Real-time Features

---

## Step 271: SignalR Overview

```csharp
// ============================================
// SignalR ใน .NET MAUI
// ============================================

/*
 * SignalR คืออะไร?
 * - Library สำหรับ real-time communication
 * - รองรับ WebSockets, Server-Sent Events, Long Polling
 * - สามารถ:
 *   - Server push data ไปยัง clients
 *   - Clients เรียก methods บน server
 *   - Broadcast ไปยัง clients หลายคน
 *   - Group messaging
 *
 * Use cases:
 * - Chat application
 * - Live notifications
 * - Real-time dashboards
 * - Collaborative editing
 * - Live tracking (delivery, GPS)
 * - Online gaming
 */

// Install NuGet:
// Microsoft.AspNetCore.SignalR.Client
```

---

## Step 272: SignalR Connection

```csharp
// ============================================
// SignalR Hub Connection
// ============================================

using Microsoft.AspNetCore.SignalR.Client;

public class ChatHubService : IAsyncDisposable
{
    private HubConnection? _connection;
    private readonly string _hubUrl;
    private readonly TokenManager _tokenManager;
    
    public event Action<ChatMessage>? MessageReceived;
    public event Action<string>? UserJoined;
    public event Action<string>? UserLeft;
    public event Action<string, bool>? UserTyping;
    public event Action<ConnectionState>? ConnectionStateChanged;
    
    public ConnectionState State => _connection?.State switch
    {
        HubConnectionState.Connected => ConnectionState.Connected,
        HubConnectionState.Connecting => ConnectionState.Connecting,
        HubConnectionState.Reconnecting => ConnectionState.Reconnecting,
        _ => ConnectionState.Disconnected
    };
    
    public ChatHubService(string hubUrl, TokenManager tokenManager)
    {
        _hubUrl = hubUrl;
        _tokenManager = tokenManager;
    }
    
    public async Task StartAsync(CancellationToken ct = default)
    {
        _connection = new HubConnectionBuilder()
            .WithUrl(_hubUrl, options =>
            {
                options.AccessTokenProvider = async () =>
                    await _tokenManager.GetValidAccessTokenAsync();
            })
            .WithAutomaticReconnect(new[] 
            { 
                TimeSpan.Zero, 
                TimeSpan.FromSeconds(2), 
                TimeSpan.FromSeconds(5), 
                TimeSpan.FromSeconds(10) 
            })
            .AddJsonProtocol()
            .Build();
        
        // Register handlers
        _connection.On<ChatMessage>("ReceiveMessage", msg =>
            MessageReceived?.Invoke(msg));
        
        _connection.On<string>("UserJoined", username =>
            UserJoined?.Invoke(username));
        
        _connection.On<string>("UserLeft", username =>
            UserLeft?.Invoke(username));
        
        _connection.On<string, bool>("UserTyping", (username, isTyping) =>
            UserTyping?.Invoke(username, isTyping));
        
        // Connection lifecycle
        _connection.Reconnecting += error =>
        {
            ConnectionStateChanged?.Invoke(ConnectionState.Reconnecting);
            return Task.CompletedTask;
        };
        
        _connection.Reconnected += connectionId =>
        {
            ConnectionStateChanged?.Invoke(ConnectionState.Connected);
            return Task.CompletedTask;
        };
        
        _connection.Closed += error =>
        {
            ConnectionStateChanged?.Invoke(ConnectionState.Disconnected);
            return Task.CompletedTask;
        };
        
        await _connection.StartAsync(ct);
        ConnectionStateChanged?.Invoke(ConnectionState.Connected);
    }
    
    public async Task StopAsync()
    {
        if (_connection != null)
        {
            await _connection.StopAsync();
            ConnectionStateChanged?.Invoke(ConnectionState.Disconnected);
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_connection != null)
            await _connection.DisposeAsync();
    }
}

public enum ConnectionState { Disconnected, Connecting, Connected, Reconnecting }
```

---

## Step 273: Chat Hub Methods

```csharp
// ============================================
// Sending Messages via Hub
// ============================================

public class ChatHubService : IAsyncDisposable
{
    // ... (continued from Step 272)
    
    public async Task JoinRoomAsync(string roomId)
    {
        if (_connection?.State != HubConnectionState.Connected) return;
        await _connection.InvokeAsync("JoinRoom", roomId);
    }
    
    public async Task LeaveRoomAsync(string roomId)
    {
        if (_connection?.State != HubConnectionState.Connected) return;
        await _connection.InvokeAsync("LeaveRoom", roomId);
    }
    
    public async Task SendMessageAsync(string roomId, string message)
    {
        if (_connection?.State != HubConnectionState.Connected)
            throw new InvalidOperationException("Not connected to hub");
        
        await _connection.InvokeAsync("SendMessage", roomId, message);
    }
    
    public async Task SendTypingIndicatorAsync(string roomId, bool isTyping)
    {
        if (_connection?.State != HubConnectionState.Connected) return;
        await _connection.SendAsync("Typing", roomId, isTyping); // Fire-and-forget
    }
    
    // Stream data from server
    public async IAsyncEnumerable<ChatMessage> GetMessageHistoryAsync(
        string roomId, 
        int pageSize = 20,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        if (_connection == null) yield break;
        
        var stream = _connection.StreamAsync<ChatMessage>(
            "GetMessageHistory", roomId, pageSize, ct);
        
        await foreach (var message in stream.WithCancellation(ct))
            yield return message;
    }
    
    // Upload file in chunks
    public async Task UploadFileAsync(string roomId, Stream fileStream, string fileName)
    {
        if (_connection == null) return;
        
        var channel = System.Threading.Channels.Channel.CreateBounded<FileChunk>(10);
        
        // Start upload
        var uploadTask = _connection.InvokeAsync("UploadFile", roomId, 
            channel.Reader.ReadAllAsync());
        
        // Stream file in chunks
        var buffer = new byte[32 * 1024]; // 32KB chunks
        int read;
        int chunkIndex = 0;
        
        while ((read = await fileStream.ReadAsync(buffer)) > 0)
        {
            await channel.Writer.WriteAsync(new FileChunk
            {
                FileName = fileName,
                Data = buffer[..read],
                Index = chunkIndex++
            });
        }
        
        channel.Writer.Complete();
        await uploadTask;
    }
}

public record ChatMessage(
    string Id,
    string SenderId,
    string SenderName,
    string Content,
    DateTimeOffset SentAt,
    string? ImageUrl = null
);

public record FileChunk(string FileName, byte[] Data, int Index);
```

---

## Step 274: Chat ViewModel

```csharp
// ============================================
// Chat ViewModel
// ============================================

public partial class ChatViewModel : ObservableObject, IQueryAttributable
{
    private readonly ChatHubService _hub;
    private string _roomId = string.Empty;
    private System.Timers.Timer? _typingTimer;
    
    [ObservableProperty] private ObservableCollection<ChatMessage> _messages = new();
    [ObservableProperty] private string _messageText = string.Empty;
    [ObservableProperty] private bool _isConnected;
    [ObservableProperty] private bool _isReconnecting;
    [ObservableProperty] private string _connectionStatus = "กำลังเชื่อมต่อ...";
    [ObservableProperty] private string _typingUsers = string.Empty;
    
    private readonly HashSet<string> _currentlyTyping = new();
    
    public ChatViewModel(ChatHubService hub)
    {
        _hub = hub;
        
        _hub.MessageReceived += OnMessageReceived;
        _hub.UserJoined += OnUserJoined;
        _hub.UserLeft += OnUserLeft;
        _hub.UserTyping += OnUserTyping;
        _hub.ConnectionStateChanged += OnConnectionStateChanged;
    }
    
    public void ApplyQueryAttributes(IDictionary<string, object> query)
    {
        if (query.TryGetValue("roomId", out var roomId))
            _roomId = roomId.ToString()!;
    }
    
    [RelayCommand]
    private async Task InitializeAsync()
    {
        try
        {
            await _hub.StartAsync();
            await _hub.JoinRoomAsync(_roomId);
            
            // Load message history
            await foreach (var msg in _hub.GetMessageHistoryAsync(_roomId))
                Messages.Insert(0, msg);
        }
        catch (Exception ex)
        {
            ConnectionStatus = $"เชื่อมต่อไม่ได้: {ex.Message}";
        }
    }
    
    [RelayCommand]
    private async Task SendAsync()
    {
        if (string.IsNullOrWhiteSpace(MessageText)) return;
        
        var text = MessageText;
        MessageText = string.Empty;
        
        try
        {
            await _hub.SendMessageAsync(_roomId, text);
        }
        catch
        {
            MessageText = text; // Restore on failure
        }
    }
    
    partial void OnMessageTextChanged(string value)
    {
        // Typing indicator
        if (!string.IsNullOrEmpty(value))
        {
            _ = _hub.SendTypingIndicatorAsync(_roomId, true);
            
            _typingTimer?.Stop();
            _typingTimer = new System.Timers.Timer(2000);
            _typingTimer.Elapsed += async (_, _) =>
            {
                await _hub.SendTypingIndicatorAsync(_roomId, false);
                _typingTimer?.Dispose();
            };
            _typingTimer.Start();
        }
    }
    
    private void OnMessageReceived(ChatMessage msg)
    {
        MainThread.BeginInvokeOnMainThread(() => Messages.Add(msg));
    }
    
    private void OnUserJoined(string username)
    {
        MainThread.BeginInvokeOnMainThread(() =>
            Messages.Add(new ChatMessage(
                Guid.NewGuid().ToString(), "system", "ระบบ",
                $"🟢 {username} เข้าร่วมห้องแชท", DateTimeOffset.Now)));
    }
    
    private void OnUserLeft(string username)
    {
        MainThread.BeginInvokeOnMainThread(() =>
            Messages.Add(new ChatMessage(
                Guid.NewGuid().ToString(), "system", "ระบบ",
                $"🔴 {username} ออกจากห้องแชท", DateTimeOffset.Now)));
    }
    
    private void OnUserTyping(string username, bool isTyping)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            if (isTyping) _currentlyTyping.Add(username);
            else _currentlyTyping.Remove(username);
            
            TypingUsers = _currentlyTyping.Count switch
            {
                0 => string.Empty,
                1 => $"{_currentlyTyping.First()} กำลังพิมพ์...",
                _ => $"{_currentlyTyping.Count} คนกำลังพิมพ์..."
            };
        });
    }
    
    private void OnConnectionStateChanged(ConnectionState state)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            IsConnected = state == ConnectionState.Connected;
            IsReconnecting = state == ConnectionState.Reconnecting;
            ConnectionStatus = state switch
            {
                ConnectionState.Connected => "เชื่อมต่อแล้ว",
                ConnectionState.Reconnecting => "กำลังเชื่อมต่อใหม่...",
                ConnectionState.Disconnected => "ขาดการเชื่อมต่อ",
                _ => "กำลังเชื่อมต่อ..."
            };
        });
    }
}
```

---

## Step 275: Chat Page XAML

```xml
<!-- ChatPage.xaml -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.ChatPage"
             Title="ห้องแชท">
    
    <ContentPage.Behaviors>
        <toolkit:EventToCommandBehavior
            EventName="Appearing"
            Command="{Binding InitializeCommand}" />
    </ContentPage.Behaviors>
    
    <Grid RowDefinitions="Auto,*,Auto">
        
        <!-- Connection Status Bar -->
        <Grid Grid.Row="0" BackgroundColor="{Binding IsConnected, Converter={StaticResource BoolToColorConverter}}"
              Padding="8,4">
            <Label Text="{Binding ConnectionStatus}" 
                   TextColor="White" HorizontalOptions="Center" />
        </Grid>
        
        <!-- Messages -->
        <CollectionView Grid.Row="1" ItemsSource="{Binding Messages}"
                        x:Name="MessagesCollection">
            <CollectionView.ItemTemplate>
                <DataTemplate x:DataType="models:ChatMessage">
                    <Grid Padding="8,4">
                        <!-- Message bubble -->
                        <Frame BackgroundColor="{AppThemeBinding Light=#E3F2FD, Dark=#1565C0}"
                               Padding="12,8" CornerRadius="12">
                            <StackLayout>
                                <Label Text="{Binding SenderName}" 
                                       FontAttributes="Bold" FontSize="12" />
                                <Label Text="{Binding Content}" FontSize="14" />
                                <Label Text="{Binding SentAt, StringFormat='{0:HH:mm}'}"
                                       FontSize="10" HorizontalOptions="End" TextColor="Gray" />
                            </StackLayout>
                        </Frame>
                    </Grid>
                </DataTemplate>
            </CollectionView.ItemTemplate>
        </CollectionView>
        
        <!-- Typing Indicator -->
        <Label Grid.Row="2" Text="{Binding TypingUsers}" 
               FontSize="12" TextColor="Gray" Padding="12,4"
               IsVisible="{Binding TypingUsers, Converter={StaticResource StringNotEmptyConverter}}" />
        
        <!-- Message Input -->
        <Grid Grid.Row="2" ColumnDefinitions="*,Auto" Padding="8">
            <Entry Grid.Column="0" Placeholder="พิมพ์ข้อความ..."
                   Text="{Binding MessageText}"
                   ReturnCommand="{Binding SendCommand}" />
            <Button Grid.Column="1" Text="ส่ง" 
                    Command="{Binding SendCommand}"
                    IsEnabled="{Binding IsConnected}" />
        </Grid>
        
    </Grid>
</ContentPage>
```

---

## Step 276: Real-time Dashboard

```csharp
// ============================================
// Real-time Analytics Dashboard
// ============================================

public class DashboardHubService : IAsyncDisposable
{
    private HubConnection? _connection;
    
    public event Action<DashboardStats>? StatsUpdated;
    public event Action<SaleEvent>? NewSale;
    public event Action<List<ChartPoint>>? ChartUpdated;
    
    public async Task ConnectAsync(string hubUrl)
    {
        _connection = new HubConnectionBuilder()
            .WithUrl(hubUrl)
            .WithAutomaticReconnect()
            .Build();
        
        _connection.On<DashboardStats>("UpdateStats", stats =>
            StatsUpdated?.Invoke(stats));
        
        _connection.On<SaleEvent>("NewSale", sale =>
            NewSale?.Invoke(sale));
        
        _connection.On<List<ChartPoint>>("UpdateChart", points =>
            ChartUpdated?.Invoke(points));
        
        await _connection.StartAsync();
    }
    
    public async Task SubscribeAsync(string[] metrics)
        => await _connection!.InvokeAsync("Subscribe", metrics);
    
    public async ValueTask DisposeAsync()
    {
        if (_connection != null) await _connection.DisposeAsync();
    }
}

public record DashboardStats(
    decimal TotalSales,
    int OrderCount,
    int ActiveUsers,
    decimal Revenue,
    DateTime UpdatedAt
);

public record SaleEvent(int OrderId, string Product, decimal Amount, DateTime At);
public record ChartPoint(DateTime Time, double Value);

// ============================================
// Dashboard ViewModel
// ============================================

public partial class DashboardViewModel : ObservableObject
{
    private readonly DashboardHubService _hub;
    
    [ObservableProperty] private decimal _totalSales;
    [ObservableProperty] private int _orderCount;
    [ObservableProperty] private int _activeUsers;
    [ObservableProperty] private ObservableCollection<SaleEvent> _recentSales = new();
    [ObservableProperty] private ObservableCollection<ChartPoint> _chartData = new();
    [ObservableProperty] private DateTime _lastUpdated;
    
    public DashboardViewModel(DashboardHubService hub)
    {
        _hub = hub;
        _hub.StatsUpdated += OnStatsUpdated;
        _hub.NewSale += OnNewSale;
        _hub.ChartUpdated += OnChartUpdated;
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        await _hub.ConnectAsync("https://api.example.com/dashboard-hub");
        await _hub.SubscribeAsync(["sales", "orders", "users", "revenue"]);
    }
    
    private void OnStatsUpdated(DashboardStats stats)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            TotalSales = stats.TotalSales;
            OrderCount = stats.OrderCount;
            ActiveUsers = stats.ActiveUsers;
            LastUpdated = stats.UpdatedAt;
        });
    }
    
    private void OnNewSale(SaleEvent sale)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            RecentSales.Insert(0, sale);
            if (RecentSales.Count > 50)
                RecentSales.RemoveAt(RecentSales.Count - 1);
        });
    }
    
    private void OnChartUpdated(List<ChartPoint> points)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            ChartData.Clear();
            foreach (var p in points) ChartData.Add(p);
        });
    }
}
```

---

## Step 277: Connection Resilience

```csharp
// ============================================
// Resilient SignalR Connection
// ============================================

public class ResilientHubConnection : IAsyncDisposable
{
    private HubConnection? _connection;
    private readonly string _hubUrl;
    private readonly SemaphoreSlim _connectLock = new(1, 1);
    private bool _disposed;
    
    public bool IsConnected => _connection?.State == HubConnectionState.Connected;
    
    public ResilientHubConnection(string hubUrl) => _hubUrl = hubUrl;
    
    private HubConnection BuildConnection()
    {
        return new HubConnectionBuilder()
            .WithUrl(_hubUrl)
            .WithAutomaticReconnect(new RetryPolicy())
            .Build();
    }
    
    public async Task EnsureConnectedAsync(CancellationToken ct = default)
    {
        if (IsConnected) return;
        
        await _connectLock.WaitAsync(ct);
        try
        {
            if (IsConnected) return;
            
            if (_connection == null)
                _connection = BuildConnection();
            
            if (_connection.State == HubConnectionState.Disconnected)
                await _connection.StartAsync(ct);
        }
        finally
        {
            _connectLock.Release();
        }
    }
    
    // Safe invoke with auto-reconnect
    public async Task InvokeAsync(string method, params object[] args)
    {
        await EnsureConnectedAsync();
        
        try
        {
            await _connection!.InvokeCoreAsync(method, args);
        }
        catch (HubException)
        {
            throw; // Server-side error - don't retry
        }
        catch
        {
            // Connection error - try to reconnect and retry once
            await Task.Delay(1000);
            await EnsureConnectedAsync();
            await _connection!.InvokeCoreAsync(method, args);
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (!_disposed)
        {
            _disposed = true;
            if (_connection != null)
                await _connection.DisposeAsync();
        }
    }
}

public class RetryPolicy : IRetryPolicy
{
    private static readonly TimeSpan[] Delays =
    {
        TimeSpan.Zero,
        TimeSpan.FromSeconds(1),
        TimeSpan.FromSeconds(5),
        TimeSpan.FromSeconds(15),
        TimeSpan.FromSeconds(30),
        TimeSpan.FromMinutes(1)
    };
    
    public TimeSpan? NextRetryDelay(RetryContext retryContext)
    {
        var index = (int)Math.Min(retryContext.PreviousRetryCount, Delays.Length - 1);
        return Delays[index];
    }
}
```

---

## Step 278: Live Notifications

```csharp
// ============================================
// Live Notification System
// ============================================

public class NotificationHubService
{
    private HubConnection? _connection;
    
    public event Action<AppNotification>? NotificationReceived;
    public event Action<int>? UnreadCountChanged;
    
    public async Task ConnectAsync(string userId, string token)
    {
        _connection = new HubConnectionBuilder()
            .WithUrl("https://api.example.com/notification-hub", opts =>
            {
                opts.AccessTokenProvider = () => Task.FromResult<string?>(token);
            })
            .WithAutomaticReconnect()
            .Build();
        
        _connection.On<AppNotification>("NewNotification", notif =>
        {
            NotificationReceived?.Invoke(notif);
            
            // Show local notification
            MainThread.BeginInvokeOnMainThread(async () =>
            {
                // Show in-app banner or local notification
                await ShowNotificationBannerAsync(notif);
            });
        });
        
        _connection.On<int>("UnreadCount", count =>
            UnreadCountChanged?.Invoke(count));
        
        await _connection.StartAsync();
    }
    
    private static async Task ShowNotificationBannerAsync(AppNotification notif)
    {
        // Show using CommunityToolkit Snackbar or Toast
        var snackbar = Snackbar.Make(
            $"{notif.Title}\n{notif.Body}",
            () => { /* Handle tap */ },
            "ดู",
            TimeSpan.FromSeconds(4));
        
        await snackbar.Show();
    }
    
    public async Task MarkReadAsync(string notificationId)
        => await _connection!.InvokeAsync("MarkRead", notificationId);
    
    public async Task MarkAllReadAsync()
        => await _connection!.InvokeAsync("MarkAllRead");
}

public record AppNotification(
    string Id,
    string Title,
    string Body,
    string? ImageUrl,
    string? ActionUrl,
    NotificationType Type,
    DateTimeOffset CreatedAt,
    bool IsRead = false
);

public enum NotificationType 
{ 
    Order, 
    Message, 
    Promotion, 
    System, 
    Alert 
}
```

---

## Step 279: Offline Queue

```csharp
// ============================================
// Offline Message Queue
// ============================================

public class OfflineMessageQueue
{
    private readonly Queue<PendingMessage> _queue = new();
    private readonly ChatHubService _hub;
    private readonly IConnectivity _connectivity;
    
    public OfflineMessageQueue(ChatHubService hub, IConnectivity connectivity)
    {
        _hub = hub;
        _connectivity = connectivity;
        _connectivity.ConnectivityChanged += OnConnectivityChanged;
    }
    
    public async Task QueueOrSendAsync(string roomId, string message)
    {
        if (_connectivity.NetworkAccess == NetworkAccess.Internet)
        {
            try
            {
                await _hub.SendMessageAsync(roomId, message);
                return;
            }
            catch { }
        }
        
        // Queue for later
        _queue.Enqueue(new PendingMessage(roomId, message, DateTimeOffset.Now));
    }
    
    private async void OnConnectivityChanged(object? sender, ConnectivityChangedEventArgs e)
    {
        if (e.NetworkAccess == NetworkAccess.Internet)
            await FlushQueueAsync();
    }
    
    private async Task FlushQueueAsync()
    {
        while (_queue.TryDequeue(out var pending))
        {
            try
            {
                await _hub.SendMessageAsync(pending.RoomId, pending.Message);
            }
            catch
            {
                // Re-queue if failed
                _queue.Enqueue(pending);
                break;
            }
        }
    }
    
    public int PendingCount => _queue.Count;
}

public record PendingMessage(string RoomId, string Message, DateTimeOffset QueuedAt);
```

---

## Step 280: SignalR Testing

```csharp
// ============================================
// Testing SignalR with Mock
// ============================================

public class MockChatHubService : ChatHubService
{
    public bool IsStarted { get; private set; }
    public List<(string RoomId, string Message)> SentMessages { get; } = new();
    
    public override Task StartAsync(CancellationToken ct = default)
    {
        IsStarted = true;
        return Task.CompletedTask;
    }
    
    public override Task SendMessageAsync(string roomId, string message)
    {
        SentMessages.Add((roomId, message));
        return Task.CompletedTask;
    }
    
    // Simulate received message
    public void SimulateIncomingMessage(ChatMessage message)
        => MessageReceived?.Invoke(message);
    
    public void SimulateUserJoined(string username)
        => UserJoined?.Invoke(username);
}

public class ChatViewModelTests
{
    [Fact]
    public async Task SendMessage_WhenConnected_AddsToSentMessages()
    {
        var mockHub = new MockChatHubService();
        var vm = new ChatViewModel(mockHub);
        await vm.InitializeCommand.ExecuteAsync(null);
        
        vm.MessageText = "Hello World";
        await vm.SendCommand.ExecuteAsync(null);
        
        mockHub.SentMessages.Should().ContainSingle(m => m.Message == "Hello World");
        vm.MessageText.Should().BeEmpty();
    }
    
    [Fact]
    public async Task ReceiveMessage_UpdatesCollection()
    {
        var mockHub = new MockChatHubService();
        var vm = new ChatViewModel(mockHub);
        await vm.InitializeCommand.ExecuteAsync(null);
        
        var testMessage = new ChatMessage("1", "user1", "TestUser", "Hi!", DateTimeOffset.Now);
        mockHub.SimulateIncomingMessage(testMessage);
        
        await Task.Delay(50); // Allow MainThread dispatch
        vm.Messages.Should().ContainSingle(m => m.Content == "Hi!");
    }
}
```

---

## สรุป Part 28

ใน Part 28 เราได้เรียนรู้:

1. **SignalR Connection** - HubConnectionBuilder, AutoReconnect
2. **Event Handlers** - On<T> สำหรับ receive messages
3. **Hub Methods** - InvokeAsync, SendAsync
4. **Streaming** - StreamAsync สำหรับ large data
5. **Chat ViewModel** - Typing indicators, connection status
6. **Chat UI** - XAML layout สำหรับ chat
7. **Real-time Dashboard** - Live stats, charts
8. **Retry Policy** - Exponential backoff
9. **Offline Queue** - Buffer messages when offline
10. **Testing** - Mock hub service

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 28 | Steps 271-280*
