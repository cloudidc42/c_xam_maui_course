# Part 87: Payments & Monetization
## Steps 861-870: Payment Gateway, In-App Purchases, Subscriptions, Loyalty Points

---

## Step 861: Payment Gateway Integration

```csharp
// ============================================
// Payment Gateway Abstraction
// ============================================

public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(string transactionId, decimal amount, string reason);
    Task<PaymentMethodResult> SaveCardAsync(CardDetails card);
    Task<List<SavedPaymentMethod>> GetSavedMethodsAsync(string customerId);
}

public record PaymentRequest(
    string CustomerId, decimal Amount, string Currency,
    string PaymentMethodId, string Description, string OrderId);

public record PaymentResult(
    bool Success, string TransactionId, string Status,
    string? ErrorCode, string? ErrorMessage);

public record CardDetails(
    string Number, string ExpiryMonth, string ExpiryYear,
    string Cvv, string HolderName);

// Omise (popular Thai payment gateway) adapter
public class OmisePaymentGateway : IPaymentGateway
{
    private readonly HttpClient _http;
    private readonly string _secretKey;
    
    public OmisePaymentGateway(HttpClient http, string secretKey)
    {
        _http = http;
        _secretKey = secretKey;
        
        _http.BaseAddress = new Uri("https://api.omise.co/");
        _http.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue(
                "Basic",
                Convert.ToBase64String(
                    System.Text.Encoding.ASCII.GetBytes($"{_secretKey}:")));
    }
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest request)
    {
        var payload = new FormUrlEncodedContent(new[]
        {
            new KeyValuePair<string, string>("amount", ((int)(request.Amount * 100)).ToString()),
            new KeyValuePair<string, string>("currency", request.Currency),
            new KeyValuePair<string, string>("card", request.PaymentMethodId),
            new KeyValuePair<string, string>("description", request.Description)
        });
        
        var response = await _http.PostAsync("/charges", payload);
        var json = await response.Content.ReadFromJsonAsync<OmiseChargeResponse>();
        
        return new PaymentResult(
            Success: json?.Status == "successful",
            TransactionId: json?.Id ?? "",
            Status: json?.Status ?? "failed",
            ErrorCode: json?.FailureCode,
            ErrorMessage: json?.FailureMessage);
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount, string reason)
    {
        var payload = new FormUrlEncodedContent(new[]
        {
            new KeyValuePair<string, string>("amount", ((int)(amount * 100)).ToString()),
            new KeyValuePair<string, string>("void", "false")
        });
        
        var response = await _http.PostAsync($"/charges/{transactionId}/refunds", payload);
        var json = await response.Content.ReadFromJsonAsync<OmiseRefundResponse>();
        
        return new RefundResult(json?.Status == "pending", json?.Id ?? "");
    }
    
    public async Task<PaymentMethodResult> SaveCardAsync(CardDetails card)
    {
        // In production: use Omise.js on the client to tokenize
        // Then send the token to your backend
        var response = await _http.PostAsJsonAsync("/tokens", card);
        var token = await response.Content.ReadFromJsonAsync<OmiseToken>();
        return new PaymentMethodResult(token?.Id ?? "", "card");
    }
    
    public async Task<List<SavedPaymentMethod>> GetSavedMethodsAsync(string customerId)
    {
        var response = await _http.GetFromJsonAsync<OmiseCustomerResponse>(
            $"/customers/{customerId}");
        return response?.Cards?.Data?.Select(c => new SavedPaymentMethod(
            c.Id, $"**** {c.LastDigits}", c.Brand, c.ExpirationMonth, c.ExpirationYear)).ToList()
            ?? new List<SavedPaymentMethod>();
    }
}

public record OmiseChargeResponse(string Id, string Status, string? FailureCode, string? FailureMessage);
public record OmiseRefundResponse(string Id, string Status);
public record OmiseToken(string Id);
public record OmiseCustomerResponse(OmiseCards? Cards);
public record OmiseCards(List<OmiseCard>? Data);
public record OmiseCard(string Id, string LastDigits, string Brand, int ExpirationMonth, int ExpirationYear);
public record RefundResult(bool Success, string RefundId);
public record PaymentMethodResult(string Id, string Type);
public record SavedPaymentMethod(string Id, string MaskedNumber, string Brand, int ExpiryMonth, int ExpiryYear);
```

---

## Step 862: Payment ViewModel

```csharp
// ============================================
// Checkout Payment ViewModel
// ============================================

public partial class PaymentViewModel : BaseViewModel
{
    private readonly IPaymentGateway _gateway;
    private readonly IEventBus _events;
    
    [ObservableProperty] private List<SavedPaymentMethod> _savedMethods = new();
    [ObservableProperty] private SavedPaymentMethod? _selectedMethod;
    [ObservableProperty] private bool _isProcessing;
    [ObservableProperty] private string _errorMessage = "";
    
    public decimal OrderTotal { get; set; }
    
    public PaymentViewModel(IPaymentGateway gateway, IEventBus events)
    {
        _gateway = gateway;
        _events = events;
    }
    
    [RelayCommand]
    public async Task LoadMethodsAsync()
    {
        var customerId = Preferences.Get("omise_customer_id", "");
        if (!string.IsNullOrEmpty(customerId))
            SavedMethods = await _gateway.GetSavedMethodsAsync(customerId);
    }
    
    [RelayCommand]
    public async Task PayAsync(string orderId)
    {
        if (SelectedMethod == null) { ErrorMessage = "กรุณาเลือกวิธีชำระเงิน"; return; }
        
        IsProcessing = true;
        ErrorMessage = "";
        
        try
        {
            var result = await _gateway.ChargeAsync(new PaymentRequest(
                CustomerId: Preferences.Get("omise_customer_id", ""),
                Amount: OrderTotal,
                Currency: "THB",
                PaymentMethodId: SelectedMethod.Id,
                Description: $"ออเดอร์ #{orderId}",
                OrderId: orderId));
            
            if (result.Success)
            {
                _events.Publish(new PaymentSucceededEvent(orderId, result.TransactionId));
                await Shell.Current.GoToAsync($"//order-tracking?orderId={orderId}");
            }
            else
            {
                ErrorMessage = GetThaiErrorMessage(result.ErrorCode);
            }
        }
        finally
        {
            IsProcessing = false;
        }
    }
    
    private static string GetThaiErrorMessage(string? code) => code switch
    {
        "insufficient_fund" => "ยอดเงินในบัตรไม่เพียงพอ",
        "stolen_or_lost_card" => "บัตรถูกระงับการใช้งาน",
        "failed_fraud_check" => "ธุรกรรมไม่ผ่านการตรวจสอบ",
        "payment_cancelled" => "ยกเลิกการชำระเงิน",
        _ => "การชำระเงินล้มเหลว กรุณาลองอีกครั้ง"
    };
}
```

---

## Step 863: PromPay QR Code

```csharp
// ============================================
// PromPay QR Code Generation
// ============================================

public class PromPayService
{
    // PromPay QR follows EMVCo specification
    // Format: "00020101021129370016A00000067701011201130066XXXXXXXXXX5303764540X.XX5802TH63XXXX"
    
    public string GenerateQrData(string phoneOrNationalId, decimal amount)
    {
        var sb = new StringBuilder();
        
        // Payload Format Indicator
        sb.Append("000201");
        // Point of Initiation Method: 12 = dynamic
        sb.Append("010212");
        // Merchant Account Information (PromPay)
        var accountInfo = BuildAccountInfo(phoneOrNationalId);
        sb.Append($"29{accountInfo.Length:D2}{accountInfo}");
        // Transaction Currency: 764 = THB
        sb.Append("5303764");
        // Transaction Amount
        var amtStr = amount.ToString("F2");
        sb.Append($"54{amtStr.Length:D2}{amtStr}");
        // Country Code
        sb.Append("5802TH");
        // CRC
        var data = sb.ToString() + "6304";
        var crc = ComputeCrc16(data);
        sb.Append($"6304{crc:X4}");
        
        return sb.ToString();
    }
    
    private string BuildAccountInfo(string account)
    {
        // AID for PromPay
        const string aid = "A000000677010112";
        // Mobile: 00 + E.164, National ID: 02 + ID
        var isPhone = account.StartsWith("0") && account.Length == 10;
        var tag = isPhone ? "01" : "02";
        var value = isPhone ? $"0066{account[1..]}" : account;
        return $"0016{aid}{tag}{value.Length:D2}{value}";
    }
    
    private static ushort ComputeCrc16(string data)
    {
        ushort crc = 0xFFFF;
        foreach (var c in data)
        {
            crc ^= (ushort)(c << 8);
            for (int i = 0; i < 8; i++)
                crc = (crc & 0x8000) != 0
                    ? (ushort)((crc << 1) ^ 0x1021)
                    : (ushort)(crc << 1);
        }
        return crc;
    }
}
```

---

## Step 864: In-App Purchases

```csharp
// ============================================
// In-App Purchase (MAUI / Plugin.InAppBilling)
// ============================================

public interface IInAppPurchaseService
{
    Task<List<InAppProduct>> GetProductsAsync(IEnumerable<string> productIds);
    Task<PurchaseResult> PurchaseAsync(string productId);
    Task<bool> RestorePurchasesAsync();
    Task<bool> IsOwnedAsync(string productId);
}

public record InAppProduct(
    string Id, string Title, string Description,
    string FormattedPrice, decimal Price, string CurrencyCode);

public record PurchaseResult(
    bool Success, string? TransactionId, string? ErrorMessage);

// Simplified implementation (real uses Plugin.InAppBilling or MAUI Essentials future API)
public class InAppPurchaseService : IInAppPurchaseService
{
    private readonly HashSet<string> _ownedProducts;
    
    public InAppPurchaseService()
    {
        // Load from secure storage
        var owned = Preferences.Get("owned_iap", "");
        _ownedProducts = owned.Split(',', StringSplitOptions.RemoveEmptyEntries).ToHashSet();
    }
    
    public async Task<List<InAppProduct>> GetProductsAsync(IEnumerable<string> productIds)
    {
        // In production: query App Store / Play Store
        await Task.Delay(100);
        
        return productIds.Select(id => id switch
        {
            "premium_monthly" => new InAppProduct(id, "Premium รายเดือน", "ฟีเจอร์พรีเมียม", "฿99/เดือน", 99, "THB"),
            "premium_yearly" => new InAppProduct(id, "Premium รายปี", "ประหยัด 40%", "฿699/ปี", 699, "THB"),
            "coin_pack_100" => new InAppProduct(id, "100 คอยน์", "สำหรับส่วนลดพิเศษ", "฿39", 39, "THB"),
            _ => new InAppProduct(id, id, "", "฿0", 0, "THB")
        }).ToList();
    }
    
    public async Task<PurchaseResult> PurchaseAsync(string productId)
    {
        // Trigger native IAP dialog
        await Task.Delay(500);
        
        // Record as owned
        _ownedProducts.Add(productId);
        Preferences.Set("owned_iap", string.Join(",", _ownedProducts));
        
        return new PurchaseResult(true, Guid.NewGuid().ToString(), null);
    }
    
    public async Task<bool> RestorePurchasesAsync()
    {
        await Task.Delay(500);
        return true;
    }
    
    public Task<bool> IsOwnedAsync(string productId)
        => Task.FromResult(_ownedProducts.Contains(productId));
}
```

---

## Step 865: Subscription Management

```csharp
// ============================================
// Subscription Lifecycle Management
// ============================================

public enum SubscriptionTier { Free, Premium, Business }
public enum SubscriptionPeriod { Monthly, Yearly }

public record SubscriptionStatus(
    SubscriptionTier Tier, SubscriptionPeriod Period,
    DateTime? ExpiresAt, bool IsAutoRenew,
    List<string> Features);

public class SubscriptionService
{
    private readonly IInAppPurchaseService _iap;
    private readonly HttpClient _http;
    
    public SubscriptionService(IInAppPurchaseService iap, HttpClient http)
    {
        _iap = iap;
        _http = http;
    }
    
    public async Task<SubscriptionStatus> GetStatusAsync()
    {
        // Verify server-side (always verify receipts server-side)
        try
        {
            return await _http.GetFromJsonAsync<SubscriptionStatus>(
                "/api/subscription/status") ?? FreeStatus;
        }
        catch
        {
            return FreeStatus;
        }
    }
    
    public async Task<bool> UpgradeAsync(SubscriptionPeriod period)
    {
        var productId = period == SubscriptionPeriod.Monthly
            ? "premium_monthly" : "premium_yearly";
        
        var result = await _iap.PurchaseAsync(productId);
        
        if (!result.Success) return false;
        
        // Verify and activate server-side
        var activated = await _http.PostAsJsonAsync(
            "/api/subscription/activate",
            new { productId, transactionId = result.TransactionId });
        
        return activated.IsSuccessStatusCode;
    }
    
    public bool HasFeature(SubscriptionStatus status, string feature)
        => status.Features.Contains(feature);
    
    private static SubscriptionStatus FreeStatus =>
        new(SubscriptionTier.Free, SubscriptionPeriod.Monthly, null, false,
            new List<string> { "basic_search", "3_orders_history" });
}

// Gate premium features in UI
public class SubscriptionGate : ContentView
{
    public static readonly BindableProperty FeatureProperty =
        BindableProperty.Create(nameof(Feature), typeof(string), typeof(SubscriptionGate));
    
    public string Feature
    {
        get => (string)GetValue(FeatureProperty);
        set => SetValue(FeatureProperty, value);
    }
    
    public new View Content { get; set; } = new Label();
    
    public SubscriptionGate()
    {
        var lockedOverlay = new Frame
        {
            BackgroundColor = Color.FromArgb("#80000000"),
            Content = new VerticalStackLayout
            {
                HorizontalOptions = LayoutOptions.Center,
                VerticalOptions = LayoutOptions.Center,
                Children =
                {
                    new Label { Text = "🔒", FontSize = 32 },
                    new Label { Text = "Premium", TextColor = Colors.White, FontAttributes = FontAttributes.Bold },
                    new Button { Text = "อัปเกรด", Command = new Command(async () =>
                        await Shell.Current.GoToAsync("//subscription")) }
                }
            }
        };
        
        var grid = new Grid { Children = { Content, lockedOverlay } };
        base.Content = grid;
    }
}
```

---

## Step 866: Loyalty Points System

```csharp
// ============================================
// Loyalty Points Engine
// ============================================

public class LoyaltyService
{
    private readonly SQLiteAsyncConnection _db;
    private readonly HttpClient _http;
    
    public LoyaltyService(SQLiteAsyncConnection db, HttpClient http)
    {
        _db = db;
        _http = http;
        _db.CreateTableAsync<LoyaltyTransaction>().Wait();
    }
    
    // Earn: 1 point per 10 baht spent
    public int CalculateEarnedPoints(decimal orderAmount)
        => (int)(orderAmount / 10);
    
    // Redeem: 100 points = 10 baht discount
    public decimal CalculateRedemptionValue(int points)
        => points / 100m * 10;
    
    public async Task<int> GetBalanceAsync(string userId)
    {
        var earned = await _db.Table<LoyaltyTransaction>()
            .Where(t => t.UserId == userId && t.Type == "earn")
            .SumAsync(t => t.Points);
        
        var redeemed = await _db.Table<LoyaltyTransaction>()
            .Where(t => t.UserId == userId && t.Type == "redeem")
            .SumAsync(t => t.Points);
        
        return earned - redeemed;
    }
    
    public async Task EarnAsync(string userId, string orderId, decimal orderAmount)
    {
        var points = CalculateEarnedPoints(orderAmount);
        if (points <= 0) return;
        
        await _db.InsertAsync(new LoyaltyTransaction
        {
            Id = Guid.NewGuid().ToString(),
            UserId = userId,
            Type = "earn",
            Points = points,
            OrderId = orderId,
            Note = $"ซื้อสินค้า ฿{orderAmount:N0}",
            CreatedAt = DateTime.UtcNow
        });
    }
    
    public async Task<bool> RedeemAsync(string userId, int points, string orderId)
    {
        var balance = await GetBalanceAsync(userId);
        if (balance < points) return false;
        
        await _db.InsertAsync(new LoyaltyTransaction
        {
            Id = Guid.NewGuid().ToString(),
            UserId = userId,
            Type = "redeem",
            Points = points,
            OrderId = orderId,
            Note = $"แลกส่วนลด ฿{CalculateRedemptionValue(points):N0}",
            CreatedAt = DateTime.UtcNow
        });
        
        return true;
    }
    
    // Tier system
    public LoyaltyTier GetTier(int lifetimePoints) => lifetimePoints switch
    {
        < 500 => new LoyaltyTier("Bronze", "🥉", 0, 499, 1.0m),
        < 2000 => new LoyaltyTier("Silver", "🥈", 500, 1999, 1.5m),
        < 5000 => new LoyaltyTier("Gold", "🥇", 2000, 4999, 2.0m),
        _ => new LoyaltyTier("Platinum", "💎", 5000, int.MaxValue, 3.0m)
    };
}

public record LoyaltyTier(
    string Name, string Icon,
    int MinPoints, int MaxPoints, decimal PointMultiplier);

public class LoyaltyTransaction
{
    [PrimaryKey] public string Id { get; set; } = "";
    public string UserId { get; set; } = "";
    public string Type { get; set; } = ""; // earn / redeem / expire / adjust
    public int Points { get; set; }
    public string? OrderId { get; set; }
    public string Note { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public DateTime? ExpiresAt { get; set; }
}
```

---

## Step 867: Referral System

```csharp
// ============================================
// Referral & Promo Code System
// ============================================

public class ReferralService
{
    private readonly HttpClient _http;
    
    public ReferralService(HttpClient http) => _http = http;
    
    public string GetMyReferralCode()
    {
        var code = Preferences.Get("referral_code", "");
        if (!string.IsNullOrEmpty(code)) return code;
        
        // Generate from user ID
        var userId = Preferences.Get("user_id", Guid.NewGuid().ToString());
        code = GenerateCode(userId);
        Preferences.Set("referral_code", code);
        return code;
    }
    
    private string GenerateCode(string userId)
    {
        var hash = System.Security.Cryptography.SHA256.HashData(
            System.Text.Encoding.UTF8.GetBytes(userId));
        return Convert.ToBase64String(hash)[..8]
            .ToUpperInvariant()
            .Replace("/", "7")
            .Replace("+", "8");
    }
    
    public async Task<PromoValidation> ValidatePromoCodeAsync(string code, decimal orderAmount)
    {
        var result = await _http.GetFromJsonAsync<PromoValidation>(
            $"/api/promo?code={Uri.EscapeDataString(code)}&amount={orderAmount}");
        
        return result ?? new PromoValidation(false, 0, "ไม่พบโค้ดส่วนลด");
    }
    
    public async Task ApplyReferralAsync(string referralCode)
    {
        var myReferralCode = GetMyReferralCode();
        if (referralCode == myReferralCode) return; // can't refer yourself
        
        await _http.PostAsJsonAsync("/api/referral/apply",
            new { referralCode, myUserId = Preferences.Get("user_id", "") });
    }
}

public record PromoValidation(bool IsValid, decimal DiscountAmount, string? Message);
```

---

## Step 868: Wallet System

```csharp
// ============================================
// Digital Wallet (stored value)
// ============================================

public class WalletService
{
    private readonly HttpClient _http;
    
    public WalletService(HttpClient http) => _http = http;
    
    public async Task<WalletBalance> GetBalanceAsync()
    {
        return await _http.GetFromJsonAsync<WalletBalance>("/api/wallet/balance")
            ?? new WalletBalance(0, "THB", DateTime.UtcNow);
    }
    
    public async Task<TopUpResult> TopUpAsync(decimal amount, string paymentMethodId)
    {
        var result = await _http.PostAsJsonAsync(
            "/api/wallet/top-up",
            new { amount, paymentMethodId });
        
        return await result.Content.ReadFromJsonAsync<TopUpResult>()
            ?? new TopUpResult(false, 0, null);
    }
    
    public async Task<List<WalletTransaction>> GetHistoryAsync(int page = 1)
    {
        return await _http.GetFromJsonAsync<List<WalletTransaction>>(
            $"/api/wallet/transactions?page={page}") ?? new List<WalletTransaction>();
    }
    
    public async Task<PaymentResult> PayWithWalletAsync(decimal amount, string orderId)
    {
        var response = await _http.PostAsJsonAsync(
            "/api/wallet/pay",
            new { amount, orderId });
        
        return await response.Content.ReadFromJsonAsync<PaymentResult>()
            ?? new PaymentResult(false, "", "failed", null, "การชำระเงินล้มเหลว");
    }
}

public record WalletBalance(decimal Amount, string Currency, DateTime AsOf);
public record TopUpResult(bool Success, decimal NewBalance, string? TransactionId);
public record WalletTransaction(
    string Id, string Type, decimal Amount,
    decimal BalanceAfter, string Description, DateTime CreatedAt);
```

---

## Step 869: Subscription Analytics

```csharp
// ============================================
// Monetization Analytics
// ============================================

public class MonetizationAnalytics
{
    private readonly SQLiteAsyncConnection _db;
    
    public MonetizationAnalytics(SQLiteAsyncConnection db) => _db = db;
    
    public async Task<RevenueReport> GetReportAsync(DateTime from, DateTime to)
    {
        var orders = await _db.QueryAsync<OrderEntity>(
            "SELECT * FROM Orders WHERE CreatedAt BETWEEN ? AND ?", from, to);
        
        var totalRevenue = orders.Sum(o => o.Total);
        var orderCount = orders.Count;
        var avgOrderValue = orderCount > 0 ? totalRevenue / orderCount : 0;
        
        // ARPU: Average Revenue Per User
        var uniqueUsers = orders.Select(o => o.UserId).Distinct().Count();
        var arpu = uniqueUsers > 0 ? totalRevenue / uniqueUsers : 0;
        
        return new RevenueReport(
            TotalRevenue: totalRevenue,
            OrderCount: orderCount,
            AverageOrderValue: avgOrderValue,
            UniqueCustomers: uniqueUsers,
            Arpu: arpu,
            Period: $"{from:d} – {to:d}");
    }
    
    public async Task<List<TopItem>> GetTopEarningItemsAsync(int limit = 10)
    {
        return await _db.QueryAsync<TopItem>(
            @"SELECT mi.Name, SUM(oi.Subtotal) as Revenue, SUM(oi.Quantity) as UnitsSold
              FROM OrderItems oi
              JOIN MenuItems mi ON oi.MenuItemId = mi.Id
              GROUP BY oi.MenuItemId
              ORDER BY Revenue DESC
              LIMIT ?", limit);
    }
}

public record RevenueReport(
    decimal TotalRevenue, int OrderCount, decimal AverageOrderValue,
    int UniqueCustomers, decimal Arpu, string Period);

public record TopItem(string Name, decimal Revenue, int UnitsSold);
```

---

## Step 870: Payment Security & Testing

```csharp
// ============================================
// Payment Security Tests
// ============================================

[TestFixture]
[Category("Payment")]
public class PaymentTests
{
    [Test]
    public void LoyaltyService_EarnPoints_CorrectCalculation()
    {
        var svc = new LoyaltyService(null!, null!);
        Assert.That(svc.CalculateEarnedPoints(100), Is.EqualTo(10));
        Assert.That(svc.CalculateEarnedPoints(99), Is.EqualTo(9));
        Assert.That(svc.CalculateEarnedPoints(1000), Is.EqualTo(100));
    }
    
    [Test]
    public void LoyaltyService_RedemptionValue_Correct()
    {
        var svc = new LoyaltyService(null!, null!);
        Assert.That(svc.CalculateRedemptionValue(100), Is.EqualTo(10));
        Assert.That(svc.CalculateRedemptionValue(500), Is.EqualTo(50));
    }
    
    [Test]
    public void LoyaltyService_TierBoundaries_Correct()
    {
        var svc = new LoyaltyService(null!, null!);
        Assert.That(svc.GetTier(0).Name, Is.EqualTo("Bronze"));
        Assert.That(svc.GetTier(500).Name, Is.EqualTo("Silver"));
        Assert.That(svc.GetTier(2000).Name, Is.EqualTo("Gold"));
        Assert.That(svc.GetTier(5000).Name, Is.EqualTo("Platinum"));
    }
    
    [Test]
    public void PromPay_QrData_HasValidCrc()
    {
        var svc = new PromPayService();
        var qr = svc.GenerateQrData("0891234567", 100);
        Assert.That(qr, Contains.Substring("6304"));
        Assert.That(qr.Length, Is.GreaterThan(50));
    }
    
    [Test]
    public void PaymentError_ThaiMessages_NeverNull()
    {
        var vm = new PaymentViewModel(null!, null!);
        // All known error codes should have Thai messages
        foreach (var code in new[] { "insufficient_fund", "stolen_or_lost_card",
            "failed_fraud_check", "payment_cancelled", "unknown_code" })
        {
            // Accessing private method via reflection for test
            var method = typeof(PaymentViewModel).GetMethod(
                "GetThaiErrorMessage",
                System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Static);
            var result = (string?)method?.Invoke(null, new object?[] { code });
            Assert.That(result, Is.Not.Empty, $"No Thai message for error code: {code}");
        }
    }
}
```

---

## สรุป Part 87

ใน Part 87 เราได้เรียนรู้:

1. **Payment Gateway** - Omise adapter, charge/refund/save card
2. **Payment ViewModel** - Error handling, Thai error messages, navigation
3. **PromPay QR** - EMVCo spec, CRC-16/CCITT calculation
4. **In-App Purchases** - Product catalog, receipt verification pattern
5. **Subscription Management** - Tier gating, feature gates in UI
6. **Loyalty Points** - Earn rules, redemption, tier multipliers
7. **Referral System** - Code generation, validation, self-referral guard
8. **Digital Wallet** - Top-up, balance, transaction history
9. **Monetization Analytics** - Revenue report, ARPU, top items
10. **Payment Tests** - Points math, tier boundaries, PromPay CRC

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 87 | Steps 861-870*
