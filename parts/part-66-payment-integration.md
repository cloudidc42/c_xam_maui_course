# Part 66: Payment Integration
## Steps 651-660: PromptPay QR, Stripe, In-App Purchase, Wallet

---

## Step 651: Payment Domain Model

```csharp
// ============================================
// Payment Domain Model
// ============================================

public enum PaymentMethod { Cash, PromptPay, CreditCard, Wallet, TrueWallet, LinePay }
public enum PaymentStatus { Pending, Verifying, Completed, Failed, Refunded }

public record PaymentInfo(
    int Id, int OrderId, PaymentMethod Method,
    decimal Amount, string Currency,
    PaymentStatus Status, string? TransactionId,
    DateTime CreatedAt, DateTime? CompletedAt);

public record PaymentRequest(
    int OrderId, decimal Amount,
    PaymentMethod Method, string? CardToken = null);

public record PaymentResult(bool Success, string? TransactionId, string? Error);

// Payment aggregate
public class Payment : AggregateRoot
{
    public int OrderId { get; private set; }
    public PaymentMethod Method { get; private set; }
    public decimal Amount { get; private set; }
    public string Currency { get; private set; } = "THB";
    public PaymentStatus Status { get; private set; }
    public string? TransactionId { get; private set; }
    public string? FailureReason { get; private set; }
    
    private Payment() { }
    
    public static Payment Create(int orderId, decimal amount, PaymentMethod method)
    {
        if (amount <= 0) throw new DomainException("Amount must be positive");
        
        var payment = new Payment
        {
            OrderId = orderId, Amount = amount, Method = method,
            Status = PaymentStatus.Pending
        };
        
        payment.AddDomainEvent(new PaymentInitiated(payment.Id, orderId, amount, method));
        return payment;
    }
    
    public void Verify(string transactionId)
    {
        if (Status != PaymentStatus.Pending)
            throw new DomainException($"Cannot verify payment in {Status} state");
        
        Status = PaymentStatus.Verifying;
        TransactionId = transactionId;
    }
    
    public void Complete()
    {
        if (Status != PaymentStatus.Verifying)
            throw new DomainException($"Cannot complete payment in {Status} state");
        
        Status = PaymentStatus.Completed;
        AddDomainEvent(new PaymentCompleted(Id, OrderId, Amount, TransactionId!));
    }
    
    public void Fail(string reason)
    {
        if (Status == PaymentStatus.Completed)
            throw new DomainException("Cannot fail a completed payment");
        
        Status = PaymentStatus.Failed;
        FailureReason = reason;
        AddDomainEvent(new PaymentFailed(Id, OrderId, reason));
    }
    
    public void Refund()
    {
        if (Status != PaymentStatus.Completed)
            throw new DomainException("Only completed payments can be refunded");
        
        Status = PaymentStatus.Refunded;
        AddDomainEvent(new PaymentRefunded(Id, OrderId, Amount));
    }
}

// Domain events
public record PaymentInitiated(int PaymentId, int OrderId, decimal Amount, PaymentMethod Method) : IDomainEvent;
public record PaymentCompleted(int PaymentId, int OrderId, decimal Amount, string TransactionId) : IDomainEvent;
public record PaymentFailed(int PaymentId, int OrderId, string Reason) : IDomainEvent;
public record PaymentRefunded(int PaymentId, int OrderId, decimal Amount) : IDomainEvent;
```

---

## Step 652: PromptPay QR Code Generator

```csharp
// ============================================
// PromptPay QR Code
// ============================================

public class PromptPayQrGenerator
{
    // PromptPay EMVCo format
    public string GeneratePayload(string promptPayId, decimal amount)
    {
        var isPhone = IsPhoneNumber(promptPayId);
        var cleanId = CleanId(promptPayId, isPhone);
        
        // Format: 00 02 01 | 0102 11 | 2937...ID | 5303764 | 5405{amount} | 5802TH | 6304{CRC}
        var payload = new StringBuilder();
        
        // Payload Format Indicator
        payload.Append("000201");
        
        // Point of Initiation: 12 = dynamic
        payload.Append("010212");
        
        // Merchant Account Info
        var accountInfo = new StringBuilder();
        accountInfo.Append("0016A000000677010111"); // PromptPay AID
        
        var idTag = isPhone ? "01" : "02"; // phone vs tax ID/national ID
        accountInfo.Append($"{idTag}{cleanId.Length:D2}{cleanId}");
        
        payload.Append($"29{accountInfo.Length:D2}{accountInfo}");
        
        // Transaction Currency: 764 = THB
        payload.Append("5303764");
        
        // Amount
        var amountStr = amount.ToString("F2");
        payload.Append($"54{amountStr.Length:D2}{amountStr}");
        
        // Country Code
        payload.Append("5802TH");
        
        // CRC placeholder
        payload.Append("6304");
        
        // Calculate CRC16
        var crc = CalculateCrc16(payload.ToString());
        payload.Append(crc.ToString("X4"));
        
        return payload.ToString();
    }
    
    private bool IsPhoneNumber(string id) => id.StartsWith("0") && id.Length == 10;
    
    private string CleanId(string id, bool isPhone)
    {
        if (isPhone)
        {
            // Convert to international format: 0891234567 -> 0066891234567
            return "0066" + id[1..];
        }
        // Tax ID or National ID: remove dashes
        return id.Replace("-", "");
    }
    
    private ushort CalculateCrc16(string data)
    {
        ushort crc = 0xFFFF;
        foreach (var c in data)
        {
            crc ^= (ushort)(c << 8);
            for (int i = 0; i < 8; i++)
            {
                if ((crc & 0x8000) != 0)
                    crc = (ushort)((crc << 1) ^ 0x1021);
                else
                    crc <<= 1;
            }
        }
        return crc;
    }
}

// QR ViewModel
public partial class PromptPayViewModel : ObservableObject
{
    private readonly PromptPayQrGenerator _qrGen;
    private readonly IOrderService _orders;
    private CancellationTokenSource? _pollCts;
    
    [ObservableProperty] private string? _qrPayload;
    [ObservableProperty] private ImageSource? _qrImage;
    [ObservableProperty] private decimal _amount;
    [ObservableProperty] private bool _isPolling;
    [ObservableProperty] private string _statusText = "รอการชำระเงิน...";
    [ObservableProperty] private bool _isPaid;
    
    public PromptPayViewModel(PromptPayQrGenerator qrGen, IOrderService orders)
    {
        _qrGen = qrGen;
        _orders = orders;
    }
    
    public async Task GenerateAsync(int orderId, decimal amount, string promptPayId)
    {
        Amount = amount;
        QrPayload = _qrGen.GeneratePayload(promptPayId, amount);
        QrImage = await GenerateQrImageAsync(QrPayload);
        StartPolling(orderId);
    }
    
    private async Task<ImageSource> GenerateQrImageAsync(string payload)
    {
        // Use ZXing.Net.Maui or BarcodeScanner.Mobile
        // For now, return placeholder
        return ImageSource.FromFile("qr_placeholder.png");
    }
    
    private void StartPolling(int orderId)
    {
        _pollCts = new CancellationTokenSource();
        IsPolling = true;
        
        _ = Task.Run(async () =>
        {
            while (!_pollCts.Token.IsCancellationRequested && !IsPaid)
            {
                await Task.Delay(3000, _pollCts.Token);
                
                var status = await _orders.GetPaymentStatusAsync(orderId);
                if (status == PaymentStatus.Completed)
                {
                    MainThread.BeginInvokeOnMainThread(() =>
                    {
                        IsPaid = true;
                        StatusText = "ชำระเงินสำเร็จ! ✓";
                        IsPolling = false;
                    });
                    return;
                }
            }
        }, _pollCts.Token);
    }
    
    public void StopPolling() => _pollCts?.Cancel();
}
```

---

## Step 653: Credit Card Payment (Stripe)

```csharp
// ============================================
// Stripe Payment Integration
// ============================================

public interface IStripeService
{
    Task<string> CreatePaymentIntentAsync(decimal amount, string currency, string? customerId);
    Task<PaymentResult> ConfirmPaymentAsync(string paymentIntentId, string cardToken);
    Task<string> TokenizeCardAsync(string number, string expMonth, string expYear, string cvc);
}

public class StripeService : IStripeService
{
    private readonly HttpClient _http;
    private readonly string _secretKey;
    
    public StripeService(HttpClient http, IConfiguration config)
    {
        _http = http;
        _secretKey = config["Stripe:SecretKey"]!;
        _http.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", _secretKey);
    }
    
    public async Task<string> CreatePaymentIntentAsync(
        decimal amount, string currency, string? customerId)
    {
        var form = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["amount"] = ((int)(amount * 100)).ToString(),
            ["currency"] = currency.ToLower(),
            ["payment_method_types[]"] = "card",
            ["customer"] = customerId ?? string.Empty
        }.Where(p => !string.IsNullOrEmpty(p.Value)));
        
        var response = await _http.PostAsync("https://api.stripe.com/v1/payment_intents", form);
        response.EnsureSuccessStatusCode();
        
        var result = await response.Content
            .ReadFromJsonAsync<JsonElement>();
        return result.GetProperty("client_secret").GetString()!;
    }
    
    public async Task<PaymentResult> ConfirmPaymentAsync(
        string paymentIntentId, string paymentMethodId)
    {
        var form = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["payment_method"] = paymentMethodId
        });
        
        var response = await _http.PostAsync(
            $"https://api.stripe.com/v1/payment_intents/{paymentIntentId}/confirm", form);
        
        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadFromJsonAsync<JsonElement>();
            return new PaymentResult(false, null,
                error.GetProperty("error").GetProperty("message").GetString());
        }
        
        var result = await response.Content.ReadFromJsonAsync<JsonElement>();
        var status = result.GetProperty("status").GetString();
        
        return status == "succeeded"
            ? new PaymentResult(true, paymentIntentId, null)
            : new PaymentResult(false, null, $"Payment status: {status}");
    }
    
    public Task<string> TokenizeCardAsync(
        string number, string expMonth, string expYear, string cvc)
        => Task.FromResult("tok_placeholder"); // Use Stripe SDK for actual tokenization
}

// Card Input ViewModel
public partial class CardInputViewModel : ObservableObject
{
    [ObservableProperty] private string _cardNumber = string.Empty;
    [ObservableProperty] private string _expiryMonth = string.Empty;
    [ObservableProperty] private string _expiryYear = string.Empty;
    [ObservableProperty] private string _cvc = string.Empty;
    [ObservableProperty] private string _cardHolder = string.Empty;
    [ObservableProperty] private bool _isProcessing;
    [ObservableProperty] private string? _error;
    
    private readonly IStripeService _stripe;
    private readonly IPaymentService _payments;
    
    public CardInputViewModel(IStripeService stripe, IPaymentService payments)
    {
        _stripe = stripe;
        _payments = payments;
    }
    
    public string FormattedCardNumber
    {
        get
        {
            var digits = new string(CardNumber.Where(char.IsDigit).ToArray());
            return string.Join(" ", Enumerable.Range(0, digits.Length / 4 + 1)
                .Select(i => digits.Skip(i * 4).Take(4).ToArray())
                .Where(g => g.Length > 0)
                .Select(g => new string(g)));
        }
    }
    
    [RelayCommand(CanExecute = nameof(CanPay))]
    private async Task PayAsync(int orderId)
    {
        IsProcessing = true;
        Error = null;
        
        try
        {
            var token = await _stripe.TokenizeCardAsync(
                CardNumber, ExpiryMonth, ExpiryYear, Cvc);
            
            var result = await _payments.PayWithCardAsync(orderId, token);
            
            if (result.Success)
                WeakReferenceMessenger.Default.Send(new PaymentSucceededMessage(orderId));
            else
                Error = result.Error;
        }
        catch (Exception ex)
        {
            Error = ex.Message;
        }
        finally { IsProcessing = false; }
    }
    
    private bool CanPay()
        => CardNumber.Length >= 16 && ExpiryMonth.Length == 2
            && ExpiryYear.Length == 2 && Cvc.Length >= 3
            && !IsProcessing;
}

public record PaymentSucceededMessage(int OrderId);
```

---

## Step 654: In-App Wallet

```csharp
// ============================================
// In-App Wallet
// ============================================

public class WalletService
{
    private readonly SQLiteConnection _db;
    private readonly IStructuredLogger _logger;
    
    public WalletService(SQLiteConnection db, IStructuredLogger logger)
    {
        _db = db;
        _logger = logger;
        _db.CreateTable<WalletTransaction>();
    }
    
    public decimal GetBalance(int userId)
    {
        var credits = _db.ExecuteScalar<decimal>(
            "SELECT COALESCE(SUM(Amount),0) FROM WalletTransaction WHERE UserId=? AND Amount>0", userId);
        var debits = _db.ExecuteScalar<decimal>(
            "SELECT COALESCE(SUM(ABS(Amount)),0) FROM WalletTransaction WHERE UserId=? AND Amount<0", userId);
        return credits - debits;
    }
    
    public void TopUp(int userId, decimal amount, string reference)
    {
        if (amount <= 0) throw new DomainException("Amount must be positive");
        
        _db.Insert(new WalletTransaction
        {
            UserId = userId, Amount = amount,
            Type = WalletTransactionType.TopUp,
            Reference = reference, Note = "เติมเงิน",
            Timestamp = DateTime.UtcNow
        });
        
        _logger.Info("Wallet top-up", new() {
            ["user_id"] = userId, ["amount"] = (object)amount });
    }
    
    public void Debit(int userId, decimal amount, string orderId, string note)
    {
        var balance = GetBalance(userId);
        if (balance < amount)
            throw new DomainException($"Insufficient balance: {balance:F2} < {amount:F2}");
        
        _db.Insert(new WalletTransaction
        {
            UserId = userId, Amount = -amount,
            Type = WalletTransactionType.Payment,
            Reference = orderId, Note = note,
            Timestamp = DateTime.UtcNow
        });
    }
    
    public void Refund(int userId, decimal amount, string orderId)
    {
        _db.Insert(new WalletTransaction
        {
            UserId = userId, Amount = amount,
            Type = WalletTransactionType.Refund,
            Reference = orderId, Note = "คืนเงิน",
            Timestamp = DateTime.UtcNow
        });
    }
    
    public List<WalletTransaction> GetHistory(int userId, int page = 1, int pageSize = 20)
        => _db.Table<WalletTransaction>()
            .Where(t => t.UserId == userId)
            .OrderByDescending(t => t.Timestamp)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToList();
}

public class WalletTransaction
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public int UserId { get; set; }
    public decimal Amount { get; set; }
    public WalletTransactionType Type { get; set; }
    public string Reference { get; set; } = string.Empty;
    public string Note { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
}

public enum WalletTransactionType { TopUp, Payment, Refund, Cashback, Promo }
```

---

## Step 655: Payment Page UI

```xml
<!-- Features/Payment/PaymentPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyFoodDelivery.Features.Payment.PaymentPage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             Title="เลือกวิธีชำระเงิน">
    
    <Grid RowDefinitions="*,Auto">
        <ScrollView Grid.Row="0">
            <VerticalStackLayout Padding="16" Spacing="16">
                
                <!-- Order Summary -->
                <Frame BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                       CornerRadius="16" Padding="16">
                    <Grid ColumnDefinitions="*,Auto">
                        <VerticalStackLayout>
                            <Label Text="ยอดที่ต้องชำระ" FontSize="14" TextColor="#8E8E93" />
                            <Label Text="{Binding TotalAmount}" FontSize="28"
                                   FontAttributes="Bold" TextColor="#007AFF" />
                        </VerticalStackLayout>
                        <VerticalStackLayout Grid.Column="1" HorizontalOptions="End">
                            <Label Text="{Binding ItemCount, StringFormat='{0} รายการ'}"
                                   FontSize="13" TextColor="#8E8E93" HorizontalOptions="End" />
                            <Label Text="{Binding RestaurantName}"
                                   FontSize="15" HorizontalOptions="End" />
                        </VerticalStackLayout>
                    </Grid>
                </Frame>
                
                <!-- Wallet Balance -->
                <Frame IsVisible="{Binding WalletBalance, Converter={StaticResource GreaterThanZeroConverter}}"
                       BackgroundColor="#F0FFF4" CornerRadius="16" Padding="16">
                    <Grid ColumnDefinitions="Auto,*,Auto">
                        <Label Grid.Column="0" Text="💰" FontSize="28" />
                        <VerticalStackLayout Grid.Column="1" Margin="12,0">
                            <Label Text="กระเป๋าเงิน" FontAttributes="Bold" />
                            <Label Text="{Binding WalletBalance, StringFormat='ยอดคงเหลือ: ฿{0:F2}'}"
                                   FontSize="13" TextColor="#8E8E93" />
                        </VerticalStackLayout>
                        <CheckBox Grid.Column="2" IsChecked="{Binding UseWallet}"
                                  Color="#007AFF" />
                    </Grid>
                </Frame>
                
                <!-- Payment Methods -->
                <Label Text="วิธีชำระเงิน" FontSize="16" FontAttributes="Bold" />
                
                <CollectionView ItemsSource="{Binding PaymentMethods}"
                                SelectionMode="Single"
                                SelectedItem="{Binding SelectedMethod}">
                    <CollectionView.ItemTemplate>
                        <DataTemplate>
                            <Frame Margin="0,0,0,8" BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                                   CornerRadius="12" Padding="16">
                                <Grid ColumnDefinitions="48,*,Auto">
                                    <Frame Grid.Column="0" Padding="0"
                                           WidthRequest="48" HeightRequest="48"
                                           CornerRadius="8">
                                        <Image Source="{Binding Icon}"
                                               Aspect="AspectFit" />
                                    </Frame>
                                    <VerticalStackLayout Grid.Column="1" Margin="12,0"
                                                         VerticalOptions="Center">
                                        <Label Text="{Binding Name}" FontSize="15" FontAttributes="Bold" />
                                        <Label Text="{Binding Description}" FontSize="13" TextColor="#8E8E93"
                                               IsVisible="{Binding Description, Converter={StaticResource NotNullConverter}}" />
                                    </VerticalStackLayout>
                                    <Image Grid.Column="2" Source="checkmark.png"
                                           HeightRequest="24" WidthRequest="24"
                                           IsVisible="{Binding IsSelected}" />
                                </Grid>
                            </Frame>
                        </DataTemplate>
                    </CollectionView.ItemTemplate>
                </CollectionView>
                
                <!-- Coupon -->
                <Frame BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                       CornerRadius="16" Padding="12">
                    <Grid ColumnDefinitions="*,Auto">
                        <Entry Placeholder="ใส่คูปอง..." Text="{Binding CouponCode}"
                               BackgroundColor="Transparent" />
                        <Button Grid.Column="1" Text="ใช้" WidthRequest="60"
                                BackgroundColor="#007AFF" TextColor="White"
                                CornerRadius="8" HeightRequest="36"
                                Command="{Binding ApplyCouponCommand}" />
                    </Grid>
                </Frame>
                
                <!-- Discount applied -->
                <Label Text="{Binding DiscountText}" IsVisible="{Binding HasDiscount}"
                       TextColor="#34C759" FontSize="14" />
                
            </VerticalStackLayout>
        </ScrollView>
        
        <!-- Pay Button -->
        <Frame Grid.Row="1" Padding="16,8,16,34" BackgroundColor="White" HasShadow="True">
            <Button Text="{Binding PayButtonText}"
                    BackgroundColor="#007AFF" TextColor="White"
                    CornerRadius="12" HeightRequest="50"
                    Command="{Binding ProcessPaymentCommand}"
                    IsEnabled="{Binding CanPay}" />
        </Frame>
    </Grid>
</ContentPage>
```

---

## Step 656: Payment Processing ViewModel

```csharp
// ============================================
// Payment Processing
// ============================================

public partial class PaymentViewModel : PageViewModel
{
    private readonly IPaymentService _payments;
    private readonly WalletService _wallet;
    private readonly INavigationService _nav;
    private readonly IDialogService _dialogs;
    
    [ObservableProperty] private decimal _totalAmount;
    [ObservableProperty] private decimal _walletBalance;
    [ObservableProperty] private bool _useWallet;
    [ObservableProperty] private string? _couponCode;
    [ObservableProperty] private decimal _discountAmount;
    [ObservableProperty] private PaymentMethodItem? _selectedMethod;
    [ObservableProperty] private List<PaymentMethodItem> _paymentMethods = new();
    [ObservableProperty] private bool _isProcessing;
    [ObservableProperty] private int _orderId;
    
    public string PayButtonText
        => IsProcessing ? "กำลังดำเนินการ..." : $"ชำระ ฿{FinalAmount:F2}";
    
    public decimal FinalAmount
        => Math.Max(0, TotalAmount - DiscountAmount - (UseWallet ? Math.Min(WalletBalance, TotalAmount - DiscountAmount) : 0));
    
    public bool CanPay => !IsProcessing && SelectedMethod != null;
    public bool HasDiscount => DiscountAmount > 0;
    public string DiscountText => $"ส่วนลด -฿{DiscountAmount:F2}";
    
    public PaymentViewModel(
        IPaymentService payments, WalletService wallet,
        INavigationService nav, IDialogService dialogs)
    {
        _payments = payments;
        _wallet = wallet;
        _nav = nav;
        _dialogs = dialogs;
    }
    
    public override async Task OnAppearingAsync()
    {
        WalletBalance = _wallet.GetBalance(GetCurrentUserId());
        PaymentMethods = BuildPaymentMethods();
        SelectedMethod = PaymentMethods.First();
    }
    
    private List<PaymentMethodItem> BuildPaymentMethods()
    {
        return new List<PaymentMethodItem>
        {
            new(PaymentMethod.PromptPay, "PromptPay", "โอนผ่าน QR Code", "promptpay.png"),
            new(PaymentMethod.CreditCard, "บัตรเครดิต/เดบิต", "Visa, Mastercard", "creditcard.png"),
            new(PaymentMethod.Cash, "เงินสด", "ชำระเมื่อได้รับ", "cash.png"),
            new(PaymentMethod.TrueWallet, "TrueMoney Wallet", null, "truemoney.png"),
        };
    }
    
    [RelayCommand]
    private async Task ApplyCouponAsync()
    {
        if (string.IsNullOrWhiteSpace(CouponCode)) return;
        
        var result = await _payments.ValidateCouponAsync(CouponCode, TotalAmount);
        if (result.IsValid)
            DiscountAmount = result.DiscountAmount;
        else
            await _dialogs.ShowAlertAsync("ไม่สำเร็จ", result.Error ?? "คูปองไม่ถูกต้อง", "ตกลง");
    }
    
    [RelayCommand(CanExecute = nameof(CanPay))]
    private async Task ProcessPaymentAsync()
    {
        IsProcessing = true;
        
        try
        {
            var request = new PaymentRequest(
                OrderId, FinalAmount, SelectedMethod!.Method);
            
            if (SelectedMethod.Method == PaymentMethod.PromptPay)
            {
                await _nav.GoToAsync<PromptPayPage>(new { orderId = OrderId, amount = FinalAmount });
                return;
            }
            
            if (SelectedMethod.Method == PaymentMethod.CreditCard)
            {
                await _nav.GoToAsync<CardInputPage>(new { orderId = OrderId, amount = FinalAmount });
                return;
            }
            
            var result = await _payments.ProcessPaymentAsync(request);
            
            if (result.Success)
            {
                await _nav.GoToAsync<OrderTrackingPage>(new { orderId = OrderId });
            }
            else
            {
                await _dialogs.ShowAlertAsync("ชำระเงินไม่สำเร็จ", result.Error!, "ลองอีกครั้ง");
            }
        }
        finally { IsProcessing = false; }
    }
    
    private int GetCurrentUserId() => 1; // hook to auth
}

public record PaymentMethodItem(
    PaymentMethod Method, string Name, string? Description, string Icon)
{
    public bool IsSelected { get; set; }
}
```

---

## Step 657: Payment Security

```csharp
// ============================================
// Payment Security
// ============================================

public class PaymentSecurityService
{
    private readonly byte[] _encryptionKey;
    
    public PaymentSecurityService()
    {
        // Retrieve from secure storage
        var key = SecureStorage.Default.GetAsync("payment_key").GetAwaiter().GetResult();
        if (key == null)
        {
            var newKey = GenerateKey();
            SecureStorage.Default.SetAsync("payment_key", Convert.ToBase64String(newKey))
                .GetAwaiter().GetResult();
            _encryptionKey = newKey;
        }
        else
        {
            _encryptionKey = Convert.FromBase64String(key);
        }
    }
    
    public string EncryptCardData(string plainText)
    {
        using var aes = System.Security.Cryptography.Aes.Create();
        aes.Key = _encryptionKey;
        aes.GenerateIV();
        
        using var encryptor = aes.CreateEncryptor();
        var plainBytes = System.Text.Encoding.UTF8.GetBytes(plainText);
        var encrypted = encryptor.TransformFinalBlock(plainBytes, 0, plainBytes.Length);
        
        var result = new byte[aes.IV.Length + encrypted.Length];
        aes.IV.CopyTo(result, 0);
        encrypted.CopyTo(result, aes.IV.Length);
        
        return Convert.ToBase64String(result);
    }
    
    public string DecryptCardData(string cipherText)
    {
        var bytes = Convert.FromBase64String(cipherText);
        
        using var aes = System.Security.Cryptography.Aes.Create();
        aes.Key = _encryptionKey;
        
        var iv = bytes[..16];
        var cipher = bytes[16..];
        
        aes.IV = iv;
        using var decryptor = aes.CreateDecryptor();
        var decrypted = decryptor.TransformFinalBlock(cipher, 0, cipher.Length);
        
        return System.Text.Encoding.UTF8.GetString(decrypted);
    }
    
    public bool ValidateLuhn(string cardNumber)
    {
        var digits = cardNumber.Where(char.IsDigit).Select(c => c - '0').ToArray();
        var sum = 0;
        var isSecond = false;
        
        for (int i = digits.Length - 1; i >= 0; i--)
        {
            var d = digits[i];
            if (isSecond) { d *= 2; if (d > 9) d -= 9; }
            sum += d;
            isSecond = !isSecond;
        }
        
        return sum % 10 == 0;
    }
    
    private static byte[] GenerateKey()
    {
        var key = new byte[32];
        System.Security.Cryptography.RandomNumberGenerator.Fill(key);
        return key;
    }
}
```

---

## Step 658: Subscription Billing

```csharp
// ============================================
// Subscription / In-App Purchase
// ============================================

public interface IInAppPurchaseService
{
    Task<IReadOnlyList<IProduct>> GetProductsAsync(string[] productIds);
    Task<PurchaseResult> PurchaseAsync(string productId);
    Task<bool> RestorePurchasesAsync();
    bool IsSubscribed(string productId);
}

public class MauiInAppPurchaseService : IInAppPurchaseService
{
    private readonly IInAppPurchase _iap;
    private readonly SQLiteConnection _db;
    
    public MauiInAppPurchaseService(IInAppPurchase iap, SQLiteConnection db)
    {
        _iap = iap;
        _db = db;
        _db.CreateTable<PurchaseRecord>();
    }
    
    public async Task<IReadOnlyList<IProduct>> GetProductsAsync(string[] productIds)
        => await _iap.GetProductInfoAsync(productIds);
    
    public async Task<PurchaseResult> PurchaseAsync(string productId)
    {
        try
        {
            var result = await _iap.PurchaseAsync(productId);
            
            if (result.State == PurchaseState.Purchased)
            {
                _db.Insert(new PurchaseRecord
                {
                    ProductId = productId,
                    TransactionId = result.Id,
                    PurchasedAt = DateTime.UtcNow,
                    ExpiresAt = GetExpiry(productId)
                });
                
                return new PurchaseResult(true, productId, null);
            }
            
            return new PurchaseResult(false, productId, "Purchase not completed");
        }
        catch (Exception ex)
        {
            return new PurchaseResult(false, productId, ex.Message);
        }
    }
    
    public async Task<bool> RestorePurchasesAsync()
    {
        var purchases = await _iap.GetPurchasesAsync();
        
        foreach (var p in purchases)
        {
            var existing = _db.Table<PurchaseRecord>()
                .FirstOrDefault(r => r.TransactionId == p.Id);
            
            if (existing == null)
            {
                _db.Insert(new PurchaseRecord
                {
                    ProductId = p.ProductId,
                    TransactionId = p.Id,
                    PurchasedAt = p.TransactionDateUtc,
                    ExpiresAt = GetExpiry(p.ProductId)
                });
            }
        }
        
        return purchases.Any();
    }
    
    public bool IsSubscribed(string productId)
    {
        var record = _db.Table<PurchaseRecord>()
            .Where(r => r.ProductId == productId)
            .OrderByDescending(r => r.PurchasedAt)
            .FirstOrDefault();
        
        if (record == null) return false;
        if (record.ExpiresAt == null) return true; // lifetime
        return record.ExpiresAt > DateTime.UtcNow;
    }
    
    private DateTime? GetExpiry(string productId)
    {
        return productId switch
        {
            "premium_monthly" => DateTime.UtcNow.AddMonths(1),
            "premium_yearly" => DateTime.UtcNow.AddYears(1),
            "premium_lifetime" => null,
            _ => DateTime.UtcNow.AddMonths(1)
        };
    }
}

public class PurchaseRecord
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string ProductId { get; set; } = string.Empty;
    public string TransactionId { get; set; } = string.Empty;
    public DateTime PurchasedAt { get; set; }
    public DateTime? ExpiresAt { get; set; }
}

public record PurchaseResult(bool Success, string ProductId, string? Error);
```

---

## Step 659: Receipt & Invoice

```csharp
// ============================================
// PDF Receipt Generation
// ============================================

public class ReceiptGenerator
{
    public async Task<string> GenerateReceiptAsync(OrderReceiptData data)
    {
        var filePath = Path.Combine(FileSystem.CacheDirectory,
            $"receipt_{data.OrderId}.txt");
        
        var sb = new StringBuilder();
        sb.AppendLine("=================================");
        sb.AppendLine($"  {data.RestaurantName}");
        sb.AppendLine("=================================");
        sb.AppendLine($"ออร์เดอร์ #{data.OrderId}");
        sb.AppendLine($"วันที่: {data.OrderDate:dd/MM/yyyy HH:mm}");
        sb.AppendLine("---------------------------------");
        
        foreach (var item in data.Items)
        {
            sb.AppendLine($"{item.Name,-20} x{item.Qty}");
            sb.AppendLine($"{"",20} ฿{item.Price:F2}");
        }
        
        sb.AppendLine("---------------------------------");
        sb.AppendLine($"{"ราคาอาหาร",-25} ฿{data.Subtotal:F2}");
        sb.AppendLine($"{"ค่าจัดส่ง",-25} ฿{data.DeliveryFee:F2}");
        
        if (data.Discount > 0)
            sb.AppendLine($"{"ส่วนลด",-25} -฿{data.Discount:F2}");
        
        sb.AppendLine("---------------------------------");
        sb.AppendLine($"{"รวมทั้งสิ้น",-25} ฿{data.Total:F2}");
        sb.AppendLine($"{"ชำระด้วย",-25} {data.PaymentMethod}");
        sb.AppendLine("=================================");
        sb.AppendLine("ขอบคุณที่ใช้บริการ");
        
        await File.WriteAllTextAsync(filePath, sb.ToString());
        return filePath;
    }
    
    public async Task ShareReceiptAsync(string filePath)
    {
        await Share.Default.RequestAsync(new ShareFileRequest
        {
            Title = "ใบเสร็จ",
            File = new ShareFile(filePath)
        });
    }
}

public record OrderReceiptData(
    int OrderId, string RestaurantName, DateTime OrderDate,
    List<ReceiptItem> Items, decimal Subtotal, decimal DeliveryFee,
    decimal Discount, decimal Total, string PaymentMethod);

public record ReceiptItem(string Name, int Qty, decimal Price);
```

---

## Step 660: Refund Flow

```csharp
// ============================================
// Refund Management
// ============================================

public class RefundService
{
    private readonly IPaymentService _payments;
    private readonly WalletService _wallet;
    private readonly IOrderRepository _orders;
    private readonly IStructuredLogger _logger;
    
    public RefundService(
        IPaymentService payments, WalletService wallet,
        IOrderRepository orders, IStructuredLogger logger)
    {
        _payments = payments;
        _wallet = wallet;
        _orders = orders;
        _logger = logger;
    }
    
    public async Task<RefundResult> ProcessRefundAsync(RefundRequest request)
    {
        var order = await _orders.GetByIdAsync(request.OrderId)
            ?? throw new DomainException("Order not found");
        
        if (!CanRefund(order))
            return new RefundResult(false, "ไม่สามารถคืนเงินได้ในขณะนี้");
        
        var refundAmount = CalculateRefundAmount(order, request);
        
        try
        {
            // Refund based on original payment method
            RefundResult result = order.PaymentMethod switch
            {
                PaymentMethod.Wallet =>
                    await RefundToWalletAsync(order.CustomerId, refundAmount, order.Id),
                
                PaymentMethod.CreditCard =>
                    await RefundToCardAsync(order.PaymentTransactionId!, refundAmount),
                
                PaymentMethod.PromptPay =>
                    await RefundToWalletAsync(order.CustomerId, refundAmount, order.Id),
                
                _ => new RefundResult(false, "ไม่รองรับการคืนเงินสำหรับวิธีชำระนี้")
            };
            
            if (result.Success)
            {
                _logger.Info("Refund processed", new()
                {
                    ["order_id"] = order.Id,
                    ["amount"] = (object)refundAmount,
                    ["reason"] = request.Reason
                });
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.Error("Refund failed", ex, new() { ["order_id"] = order.Id });
            return new RefundResult(false, "เกิดข้อผิดพลาด กรุณาติดต่อเรา");
        }
    }
    
    private bool CanRefund(FoodOrder order)
        => order.Status == OrderStatus.Placed || order.Status == OrderStatus.Confirmed
            || order.Status == OrderStatus.Accepted;
    
    private decimal CalculateRefundAmount(FoodOrder order, RefundRequest request)
        => request.IsFullRefund ? order.Total.Amount : request.Amount;
    
    private Task<RefundResult> RefundToWalletAsync(int userId, decimal amount, int orderId)
    {
        _wallet.Refund(userId, amount, orderId.ToString());
        return Task.FromResult(new RefundResult(true, null, amount));
    }
    
    private Task<RefundResult> RefundToCardAsync(string transactionId, decimal amount)
        => _payments.RefundAsync(transactionId, amount);
}

public record RefundRequest(int OrderId, string Reason, bool IsFullRefund, decimal Amount = 0);
public record RefundResult(bool Success, string? Error, decimal? RefundedAmount = null);
```

---

## สรุป Part 66

ใน Part 66 เราได้เรียนรู้:

1. **Payment Domain Model** - Payment aggregate, domain events
2. **PromptPay QR** - EMVCo format, CRC16, payload generation
3. **Stripe Integration** - Payment Intent, confirm, tokenize
4. **In-App Wallet** - Balance, top-up, debit, refund
5. **Payment Page XAML** - Method selection, wallet toggle, coupon
6. **Payment ViewModel** - Process flow, routing to sub-pages
7. **Payment Security** - AES encryption, Luhn validation
8. **In-App Purchase** - MAUI IInAppPurchase, subscription management
9. **Receipt Generation** - Text receipt, share
10. **Refund Flow** - Method-aware refund, wallet/card routing

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 66 | Steps 651-660*
