# Part 74: Machine Learning & Smart Recommendations
## Steps 731-740: On-Device ML, Recommendation Engine, NLP Search, Image Recognition

---

## Step 731: ML.NET Integration

```csharp
// ============================================
// ML.NET On-Device Recommendation Engine
// ============================================

// Install: Microsoft.ML, Microsoft.ML.Recommender

public class FoodRecommendationModel
{
    private readonly MLContext _mlContext;
    private ITransformer? _model;
    private PredictionEngine<OrderRating, OrderPrediction>? _predictor;
    
    public FoodRecommendationModel()
    {
        _mlContext = new MLContext(seed: 42);
    }
    
    public async Task TrainAsync(IEnumerable<OrderRating> trainingData)
    {
        var data = _mlContext.Data.LoadFromEnumerable(trainingData);
        
        var pipeline = _mlContext.Transforms.Conversion.MapValueToKey("userId")
            .Append(_mlContext.Transforms.Conversion.MapValueToKey("itemId"))
            .Append(_mlContext.Recommendation().Trainers.MatrixFactorization(
                labelColumnName: "label",
                matrixColumnIndexColumnName: "userId",
                matrixRowIndexColumnName: "itemId",
                numberOfIterations: 20,
                approximationRank: 100));
        
        _model = await Task.Run(() => pipeline.Fit(data));
        _predictor = _mlContext.Model.CreatePredictionEngine<OrderRating, OrderPrediction>(_model);
    }
    
    public float PredictScore(string userId, string menuItemId)
    {
        if (_predictor == null) return 0;
        
        var prediction = _predictor.Predict(new OrderRating
        {
            UserId = userId,
            ItemId = menuItemId
        });
        
        return Math.Clamp(prediction.Score, 1, 5);
    }
    
    public async Task SaveModelAsync(string path)
    {
        if (_model == null) return;
        await Task.Run(() => _mlContext.Model.Save(_model, null, path));
    }
    
    public async Task LoadModelAsync(string path)
    {
        if (!File.Exists(path)) return;
        await Task.Run(() =>
        {
            _model = _mlContext.Model.Load(path, out _);
            _predictor = _mlContext.Model.CreatePredictionEngine<OrderRating, OrderPrediction>(_model);
        });
    }
}

public class OrderRating
{
    public string UserId { get; set; } = "";
    public string ItemId { get; set; } = "";
    public float Label { get; set; } // 1-5 stars
}

public class OrderPrediction
{
    public float Score { get; set; }
}
```

---

## Step 732: Recommendation Service

```csharp
// ============================================
// Smart Recommendation Service
// ============================================

public class RecommendationService
{
    private readonly FoodRecommendationModel _mlModel;
    private readonly IMenuRepository _menuRepo;
    private readonly IOrderRepository _orderRepo;
    
    public RecommendationService(FoodRecommendationModel mlModel,
        IMenuRepository menuRepo, IOrderRepository orderRepo)
    {
        _mlModel = mlModel;
        _menuRepo = menuRepo;
        _orderRepo = orderRepo;
    }
    
    public async Task<IReadOnlyList<RecommendedItem>> GetPersonalizedAsync(
        string userId, int count = 10)
    {
        var allItems = await _menuRepo.GetAllMenuItemsAsync();
        var orderedIds = (await _orderRepo.GetOrderedItemIdsAsync(userId)).ToHashSet();
        
        // Score each item not yet ordered
        var scored = allItems
            .Where(item => !orderedIds.Contains(item.Id))
            .Select(item => new RecommendedItem(
                item,
                Score: _mlModel.PredictScore(userId, item.Id),
                Reason: "แนะนำสำหรับคุณ"))
            .OrderByDescending(x => x.Score)
            .Take(count)
            .ToList();
        
        return scored.AsReadOnly();
    }
    
    public async Task<IReadOnlyList<RecommendedItem>> GetTrendingAsync(
        string restaurantId, int hours = 24, int count = 5)
    {
        var cutoff = DateTime.UtcNow.AddHours(-hours);
        var trending = await _orderRepo.GetTrendingItemsAsync(restaurantId, cutoff, count);
        
        return trending.Select((item, i) => new RecommendedItem(
            item.MenuItem,
            Score: (float)(count - i) / count,
            Reason: $"ยอดนิยม {item.OrderCount} ออเดอร์ใน {hours} ชั่วโมง"))
            .ToList().AsReadOnly();
    }
    
    public async Task<IReadOnlyList<RecommendedItem>> GetSimilarItemsAsync(
        string menuItemId, int count = 5)
    {
        var item = await _menuRepo.GetByIdAsync(menuItemId);
        if (item == null) return Array.Empty<RecommendedItem>();
        
        // Content-based: same category + similar price range
        var similar = await _menuRepo.GetSimilarAsync(item.CategoryId, item.Price, count + 1);
        
        return similar
            .Where(x => x.Id != menuItemId)
            .Take(count)
            .Select(x => new RecommendedItem(x, Score: 0.8f, Reason: "เมนูที่คล้ายกัน"))
            .ToList().AsReadOnly();
    }
    
    public async Task<IReadOnlyList<RecommendedItem>> GetTimeBasedAsync(
        string userId, int count = 5)
    {
        var hour = DateTime.Now.Hour;
        var meal = hour switch
        {
            >= 6 and < 10 => "breakfast",
            >= 10 and < 14 => "lunch",
            >= 14 and < 18 => "snack",
            _ => "dinner"
        };
        
        var items = await _menuRepo.GetByMealTypeAsync(meal, count);
        return items.Select(x => new RecommendedItem(x, Score: 0.7f,
            Reason: hour < 12 ? "เหมาะสำหรับมื้อเช้า" :
                    hour < 18 ? "เหมาะสำหรับมื้อกลางวัน" : "เหมาะสำหรับมื้อเย็น"))
            .ToList().AsReadOnly();
    }
}

public record RecommendedItem(MenuItem MenuItem, float Score, string Reason);
```

---

## Step 733: Semantic Search with TF-IDF

```csharp
// ============================================
// Thai Food Search with TF-IDF Scoring
// ============================================

public class ThaiTextSearchEngine
{
    private readonly Dictionary<string, Dictionary<string, double>> _tfidf = new();
    private readonly Dictionary<string, MenuItem> _index = new();
    
    public void IndexMenuItems(IEnumerable<MenuItem> items)
    {
        var documents = items.ToDictionary(x => x.Id, x =>
            $"{x.Name} {x.Description} {string.Join(" ", x.Tags)}");
        
        var termFrequency = ComputeTF(documents);
        var docFrequency = ComputeDF(documents);
        var totalDocs = documents.Count;
        
        foreach (var (id, tf) in termFrequency)
        {
            _tfidf[id] = tf.ToDictionary(
                kv => kv.Key,
                kv => kv.Value * Math.Log((double)totalDocs / (docFrequency[kv.Key] + 1)));
        }
        
        foreach (var item in items)
            _index[item.Id] = item;
    }
    
    public IReadOnlyList<SearchResult> Search(string query, int topK = 10)
    {
        var queryTerms = Tokenize(query);
        
        var scores = _tfidf.Select(doc =>
        {
            var score = queryTerms
                .Where(term => doc.Value.ContainsKey(term))
                .Sum(term => doc.Value[term]);
            return (Id: doc.Key, Score: score);
        })
        .Where(x => x.Score > 0)
        .OrderByDescending(x => x.Score)
        .Take(topK);
        
        return scores
            .Where(x => _index.ContainsKey(x.Id))
            .Select(x => new SearchResult(_index[x.Id], x.Score))
            .ToList().AsReadOnly();
    }
    
    private static Dictionary<string, Dictionary<string, double>> ComputeTF(
        Dictionary<string, string> docs)
    {
        return docs.ToDictionary(
            kv => kv.Key,
            kv =>
            {
                var terms = Tokenize(kv.Value);
                var count = terms.Count;
                return terms.GroupBy(t => t)
                    .ToDictionary(g => g.Key, g => (double)g.Count() / count);
            });
    }
    
    private static Dictionary<string, int> ComputeDF(Dictionary<string, string> docs)
    {
        var df = new Dictionary<string, int>();
        foreach (var doc in docs.Values)
        {
            foreach (var term in Tokenize(doc).Distinct())
            {
                df[term] = df.GetValueOrDefault(term, 0) + 1;
            }
        }
        return df;
    }
    
    private static List<string> Tokenize(string text)
    {
        // Simple tokenization for Thai (syllable-based)
        // In production: use Jieba.NET or custom Thai word segmenter
        return text.ToLower()
            .Split(new[] { ' ', ',', '.', '/', '-' }, StringSplitOptions.RemoveEmptyEntries)
            .Where(t => t.Length > 1)
            .ToList();
    }
}

public record SearchResult(MenuItem Item, double Score);
```

---

## Step 734: Smart Autocomplete

```csharp
// ============================================
// Thai Search Autocomplete with Trie
// ============================================

public class TrieNode
{
    public Dictionary<char, TrieNode> Children { get; } = new();
    public bool IsEnd { get; set; }
    public string? Word { get; set; }
    public int Frequency { get; set; }
}

public class SearchTrie
{
    private readonly TrieNode _root = new();
    
    public void Insert(string word, int frequency = 1)
    {
        var node = _root;
        foreach (var ch in word.ToLower())
        {
            if (!node.Children.ContainsKey(ch))
                node.Children[ch] = new TrieNode();
            node = node.Children[ch];
        }
        node.IsEnd = true;
        node.Word = word;
        node.Frequency += frequency;
    }
    
    public List<AutoCompleteResult> Search(string prefix, int maxResults = 5)
    {
        var node = _root;
        foreach (var ch in prefix.ToLower())
        {
            if (!node.Children.TryGetValue(ch, out var next)) return new();
            node = next;
        }
        
        var results = new List<AutoCompleteResult>();
        DFS(node, results);
        return results.OrderByDescending(x => x.Frequency).Take(maxResults).ToList();
    }
    
    private void DFS(TrieNode node, List<AutoCompleteResult> results)
    {
        if (node.IsEnd && node.Word != null)
            results.Add(new AutoCompleteResult(node.Word, node.Frequency));
        
        foreach (var child in node.Children.Values)
            DFS(child, results);
    }
}

public record AutoCompleteResult(string Text, int Frequency);

// SearchViewModel with debounced autocomplete
public partial class SearchViewModel : ObservableObject
{
    private readonly SearchTrie _trie;
    private readonly ThaiTextSearchEngine _engine;
    private readonly IDebouncer _debouncer;
    
    [ObservableProperty] private string _query = "";
    [ObservableProperty] private ObservableCollection<AutoCompleteResult> _suggestions = new();
    [ObservableProperty] private ObservableCollection<SearchResult> _results = new();
    
    partial void OnQueryChanged(string value)
    {
        _debouncer.Debounce(TimeSpan.FromMilliseconds(300), () =>
        {
            MainThread.BeginInvokeOnMainThread(() =>
            {
                if (value.Length >= 1)
                {
                    var suggestions = _trie.Search(value, 5);
                    Suggestions = new ObservableCollection<AutoCompleteResult>(suggestions);
                }
                
                if (value.Length >= 2)
                {
                    var results = _engine.Search(value, 20);
                    Results = new ObservableCollection<SearchResult>(results);
                }
            });
        });
    }
    
    public SearchViewModel(SearchTrie trie, ThaiTextSearchEngine engine, IDebouncer debouncer)
    {
        _trie = trie;
        _engine = engine;
        _debouncer = debouncer;
    }
}
```

---

## Step 735: Image Recognition for Food

```csharp
// ============================================
// On-Device Food Image Classification
// ============================================

// Uses ONNX model for on-device inference
// Install: Microsoft.ML.OnnxRuntime

public class FoodClassifier
{
    private readonly InferenceSession? _session;
    private readonly string[] _labels;
    
    public FoodClassifier()
    {
        // Model bundled as MauiAsset
        _labels = new[] {
            "ข้าวผัดกุ้ง", "ผัดไทย", "ต้มยำกุ้ง", "ส้มตำ", "ข้าวมันไก่",
            "หมูกะทะ", "ข้าวผัดปู", "แกงเขียวหวาน", "มะม่วงน้ำปลาหวาน", "ข้าวหน้าเป็ด"
        };
        
        try
        {
            using var stream = FileSystem.OpenAppPackageFileAsync("food_model.onnx").Result;
            var bytes = new byte[stream.Length];
            stream.Read(bytes, 0, bytes.Length);
            _session = new InferenceSession(bytes);
        }
        catch
        {
            // Model not available
        }
    }
    
    public async Task<FoodClassification> ClassifyAsync(byte[] imageBytes)
    {
        if (_session == null)
            return new FoodClassification("ไม่สามารถวิเคราะห์ได้", 0);
        
        var tensor = await PreprocessImageAsync(imageBytes);
        
        var inputs = new List<NamedOnnxValue>
        {
            NamedOnnxValue.CreateFromTensor("input", tensor)
        };
        
        using var results = _session.Run(inputs);
        var output = results.First().AsEnumerable<float>().ToArray();
        
        var maxIdx = Array.IndexOf(output, output.Max());
        var confidence = output[maxIdx];
        
        return new FoodClassification(
            _labels[Math.Min(maxIdx, _labels.Length - 1)],
            confidence);
    }
    
    private async Task<DenseTensor<float>> PreprocessImageAsync(byte[] imageBytes)
    {
        // Resize to 224x224, normalize to [-1, 1]
        await Task.CompletedTask;
        return new DenseTensor<float>(new[] { 1, 3, 224, 224 });
    }
}

public record FoodClassification(string Label, float Confidence);
```

---

## Step 736: Personalization Engine

```csharp
// ============================================
// User Preference Learning
// ============================================

public class PersonalizationEngine
{
    private readonly SQLiteAsyncConnection _db;
    
    public PersonalizationEngine(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<UserPreference>().Wait();
        _db.CreateTableAsync<UserInteraction>().Wait();
    }
    
    // Record every interaction
    public async Task RecordInteractionAsync(string userId, string itemId,
        InteractionType type, float? rating = null)
    {
        await _db.InsertAsync(new UserInteraction
        {
            UserId = userId,
            ItemId = itemId,
            Type = type,
            Rating = rating,
            Timestamp = DateTime.UtcNow
        });
        
        await UpdatePreferencesAsync(userId, itemId, type, rating);
    }
    
    private async Task UpdatePreferencesAsync(string userId, string itemId,
        InteractionType type, float? rating)
    {
        var existing = await _db.Table<UserPreference>()
            .Where(p => p.UserId == userId && p.ItemId == itemId)
            .FirstOrDefaultAsync();
        
        var score = type switch
        {
            InteractionType.View => 0.1f,
            InteractionType.AddToCart => 0.3f,
            InteractionType.Order => 0.7f,
            InteractionType.Rate => rating ?? 0.5f,
            InteractionType.Skip => -0.1f,
            _ => 0f
        };
        
        if (existing == null)
        {
            await _db.InsertAsync(new UserPreference
            {
                UserId = userId,
                ItemId = itemId,
                Score = score,
                UpdatedAt = DateTime.UtcNow
            });
        }
        else
        {
            // Exponential moving average
            existing.Score = existing.Score * 0.7f + score * 0.3f;
            existing.UpdatedAt = DateTime.UtcNow;
            await _db.UpdateAsync(existing);
        }
    }
    
    public async Task<Dictionary<string, float>> GetUserPreferencesAsync(string userId)
    {
        var prefs = await _db.Table<UserPreference>()
            .Where(p => p.UserId == userId)
            .ToListAsync();
        return prefs.ToDictionary(p => p.ItemId, p => p.Score);
    }
    
    // Collaborative filtering: "users like you also ordered"
    public async Task<IReadOnlyList<string>> GetCollaborativeRecsAsync(
        string userId, int count = 5)
    {
        var myPrefs = await GetUserPreferencesAsync(userId);
        if (myPrefs.Count == 0) return Array.Empty<string>();
        
        // Find similar users
        var allUsers = await _db.Table<UserPreference>().ToListAsync();
        
        var similarity = allUsers
            .Where(p => p.UserId != userId)
            .GroupBy(p => p.UserId)
            .Select(g =>
            {
                var otherPrefs = g.ToDictionary(p => p.ItemId, p => p.Score);
                var sim = CosineSimilarity(myPrefs, otherPrefs);
                return (UserId: g.Key, Similarity: sim, Prefs: otherPrefs);
            })
            .OrderByDescending(x => x.Similarity)
            .Take(5);
        
        var myItems = myPrefs.Keys.ToHashSet();
        
        return similarity
            .SelectMany(u => u.Prefs.Keys)
            .Where(itemId => !myItems.Contains(itemId))
            .GroupBy(x => x)
            .OrderByDescending(g => g.Count())
            .Take(count)
            .Select(g => g.Key)
            .ToList().AsReadOnly();
    }
    
    private float CosineSimilarity(
        Dictionary<string, float> a, Dictionary<string, float> b)
    {
        var common = a.Keys.Intersect(b.Keys);
        var dot = common.Sum(k => a[k] * b[k]);
        var magA = Math.Sqrt(a.Values.Sum(v => v * v));
        var magB = Math.Sqrt(b.Values.Sum(v => v * v));
        return magA == 0 || magB == 0 ? 0 : (float)(dot / (magA * magB));
    }
}

public enum InteractionType { View, AddToCart, Order, Rate, Skip }

[Table("user_preferences")]
public class UserPreference
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    [Indexed] public string UserId { get; set; } = "";
    [Indexed] public string ItemId { get; set; } = "";
    public float Score { get; set; }
    public DateTime UpdatedAt { get; set; }
}

[Table("user_interactions")]
public class UserInteraction
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string UserId { get; set; } = "";
    public string ItemId { get; set; } = "";
    public InteractionType Type { get; set; }
    public float? Rating { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## Step 737: Smart Notifications

```csharp
// ============================================
// ML-Powered Push Notification Timing
// ============================================

public class SmartNotificationScheduler
{
    private readonly PersonalizationEngine _engine;
    private readonly RecommendationService _recs;
    
    public SmartNotificationScheduler(
        PersonalizationEngine engine, RecommendationService recs)
    {
        _engine = engine;
        _recs = recs;
    }
    
    public async Task<NotificationPlan> PlanAsync(string userId)
    {
        var prefs = await _engine.GetUserPreferencesAsync(userId);
        var recs = await _recs.GetPersonalizedAsync(userId, 3);
        
        // Predict best time based on past order history
        var bestHour = await PredictBestNotificationHourAsync(userId);
        
        var plan = new NotificationPlan
        {
            UserId = userId,
            ScheduledHour = bestHour,
            TopRecommendations = recs.Take(3).ToList()
        };
        
        return plan;
    }
    
    private async Task<int> PredictBestNotificationHourAsync(string userId)
    {
        // Find hour with most past orders
        var interactions = await Task.FromResult(new List<UserInteraction>());
        
        if (!interactions.Any()) return 12; // Default: noon
        
        return interactions
            .GroupBy(i => i.Timestamp.Hour)
            .OrderByDescending(g => g.Count())
            .First().Key;
    }
}

public class NotificationPlan
{
    public string UserId { get; set; } = "";
    public int ScheduledHour { get; set; }
    public List<RecommendedItem> TopRecommendations { get; set; } = new();
    
    public string BuildNotificationTitle()
    {
        var hour = ScheduledHour;
        return hour < 10 ? "มื้อเช้าวันนี้ รับส่วนลด 20%" :
               hour < 14 ? "หิวมื้อกลางวันไหม? มีเมนูแนะนำ" :
               hour < 18 ? "ของว่างยามบ่าย รับฟรีเครื่องดื่ม" :
               "มื้อเย็นอร่อยๆ รอคุณอยู่";
    }
    
    public string BuildNotificationBody()
        => TopRecommendations.Count > 0
            ? $"เมนูแนะนำ: {string.Join(", ", TopRecommendations.Take(2).Select(r => r.MenuItem.Name))}"
            : "มีโปรโมชั่นพิเศษรอคุณอยู่";
}
```

---

## Step 738: A/B Testing for Recommendations

```csharp
// ============================================
// A/B Test for Recommendation Algorithm
// ============================================

public class RecommendationExperiment
{
    private readonly CanaryReleaseService _canary;
    private readonly RecommendationService _service;
    private readonly PersonalizationEngine _ml;
    
    public RecommendationExperiment(CanaryReleaseService canary,
        RecommendationService service, PersonalizationEngine ml)
    {
        _canary = canary;
        _service = service;
        _ml = ml;
    }
    
    public async Task<IReadOnlyList<RecommendedItem>> GetRecommendationsAsync(
        string userId, int count = 10)
    {
        // 50% see ML personalized, 50% see trending
        var isInMlGroup = await _canary.IsInCanaryGroupAsync("ml-recommendations-v2", userId, 50);
        
        if (isInMlGroup)
        {
            TrackExperiment(userId, "ml-group");
            return await _service.GetPersonalizedAsync(userId, count);
        }
        else
        {
            TrackExperiment(userId, "trending-group");
            return await _service.GetTrendingAsync("all", count: count);
        }
    }
    
    private void TrackExperiment(string userId, string variant)
        => WeakReferenceMessenger.Default.Send(
            new ExperimentEvent("recommendation-algo", userId, variant));
}

public record ExperimentEvent(string ExperimentId, string UserId, string Variant);
```

---

## Step 739: Natural Language Order Processing

```csharp
// ============================================
// Natural Language Order Parser
// ============================================

public class NaturalLanguageOrderParser
{
    private static readonly Dictionary<string, decimal> _quantityWords = new()
    {
        { "หนึ่ง", 1 }, { "สอง", 2 }, { "สาม", 3 },
        { "สี่", 4 }, { "ห้า", 5 }, { "หก", 6 },
        { "1", 1 }, { "2", 2 }, { "3", 3 },
        { "4", 4 }, { "5", 5 }, { "6", 6 },
    };
    
    private static readonly string[] _orderVerbs = { "สั่ง", "เอา", "ขอ", "อยากได้" };
    
    public ParsedOrder Parse(string input, IReadOnlyList<MenuItem> menu)
    {
        var result = new ParsedOrder();
        var lower = input.ToLower();
        
        // Find items in the input
        foreach (var item in menu)
        {
            if (!lower.Contains(item.Name.ToLower())) continue;
            
            var qty = ExtractQuantity(lower, item.Name.ToLower());
            result.Items.Add(new ParsedOrderItem(item, qty));
        }
        
        // Extract modifiers (ไม่เผ็ด, เผ็ดน้อย, ไม่ใส่ผัก)
        result.Modifiers = ExtractModifiers(lower);
        
        // Delivery notes
        result.Notes = ExtractNotes(input);
        
        return result;
    }
    
    private int ExtractQuantity(string text, string itemName)
    {
        var idx = text.IndexOf(itemName);
        if (idx < 0) return 1;
        
        // Look for quantity before item name
        var beforeItem = text[..idx];
        
        foreach (var (word, qty) in _quantityWords)
        {
            if (beforeItem.Contains(word))
                return (int)qty;
        }
        
        return 1;
    }
    
    private List<string> ExtractModifiers(string text)
    {
        var mods = new List<string>();
        if (text.Contains("ไม่เผ็ด")) mods.Add("ไม่เผ็ด");
        if (text.Contains("เผ็ดน้อย")) mods.Add("เผ็ดน้อย");
        if (text.Contains("เผ็ดมาก")) mods.Add("เผ็ดมาก");
        if (text.Contains("ไม่ใส่ผัก")) mods.Add("ไม่ใส่ผัก");
        if (text.Contains("ไม่ใส่ผักชี")) mods.Add("ไม่ใส่ผักชี");
        if (text.Contains("ไม่หวาน")) mods.Add("ไม่หวาน");
        return mods;
    }
    
    private string? ExtractNotes(string text)
    {
        // Extract content after "หมายเหตุ:" or "note:"
        var noteIdx = text.IndexOf("หมายเหตุ:", StringComparison.OrdinalIgnoreCase);
        if (noteIdx < 0) return null;
        return text[(noteIdx + 9)..].Trim();
    }
}

public class ParsedOrder
{
    public List<ParsedOrderItem> Items { get; set; } = new();
    public List<string> Modifiers { get; set; } = new();
    public string? Notes { get; set; }
    public bool IsValid => Items.Count > 0;
}

public record ParsedOrderItem(MenuItem MenuItem, int Quantity);
```

---

## Step 740: Recommendation Analytics

```csharp
// ============================================
// Track Recommendation Performance
// ============================================

public class RecommendationAnalytics
{
    private readonly SQLiteAsyncConnection _db;
    
    public RecommendationAnalytics(SQLiteAsyncConnection db)
    {
        _db = db;
        _db.CreateTableAsync<RecoImpression>().Wait();
        _db.CreateTableAsync<RecoClick>().Wait();
        _db.CreateTableAsync<RecoConversion>().Wait();
    }
    
    public async Task TrackImpressionAsync(string userId, string itemId, string source)
        => await _db.InsertAsync(new RecoImpression
        {
            UserId = userId, ItemId = itemId, Source = source,
            Timestamp = DateTime.UtcNow
        });
    
    public async Task TrackClickAsync(string userId, string itemId)
        => await _db.InsertAsync(new RecoClick
        {
            UserId = userId, ItemId = itemId, Timestamp = DateTime.UtcNow
        });
    
    public async Task TrackConversionAsync(string userId, string itemId, decimal revenue)
        => await _db.InsertAsync(new RecoConversion
        {
            UserId = userId, ItemId = itemId, Revenue = revenue,
            Timestamp = DateTime.UtcNow
        });
    
    public async Task<RecoMetrics> ComputeMetricsAsync(string source, DateTime since)
    {
        var impressions = await _db.Table<RecoImpression>()
            .Where(x => x.Source == source && x.Timestamp >= since)
            .CountAsync();
        
        var clicks = await _db.Table<RecoClick>()
            .Where(x => x.Timestamp >= since).CountAsync();
        
        var conversions = await _db.Table<RecoConversion>()
            .Where(x => x.Timestamp >= since).ToListAsync();
        
        return new RecoMetrics
        {
            Impressions = impressions,
            Clicks = clicks,
            Conversions = conversions.Count,
            TotalRevenue = conversions.Sum(c => c.Revenue),
            CTR = impressions == 0 ? 0 : (double)clicks / impressions,
            ConversionRate = clicks == 0 ? 0 : (double)conversions.Count / clicks
        };
    }
}

public class RecoMetrics
{
    public int Impressions { get; set; }
    public int Clicks { get; set; }
    public int Conversions { get; set; }
    public decimal TotalRevenue { get; set; }
    public double CTR { get; set; }
    public double ConversionRate { get; set; }
    
    public override string ToString()
        => $"Impressions: {Impressions}, CTR: {CTR:P1}, CVR: {ConversionRate:P1}, Revenue: ฿{TotalRevenue:F0}";
}

[Table("reco_impressions")]
public class RecoImpression
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string UserId { get; set; } = "";
    public string ItemId { get; set; } = "";
    public string Source { get; set; } = "";
    public DateTime Timestamp { get; set; }
}

[Table("reco_clicks")]
public class RecoClick
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string UserId { get; set; } = "";
    public string ItemId { get; set; } = "";
    public DateTime Timestamp { get; set; }
}

[Table("reco_conversions")]
public class RecoConversion
{
    [PrimaryKey, AutoIncrement] public int Id { get; set; }
    public string UserId { get; set; } = "";
    public string ItemId { get; set; } = "";
    public decimal Revenue { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## สรุป Part 74

ใน Part 74 เราได้เรียนรู้:

1. **ML.NET Recommendation** - Matrix factorization, train/predict/save/load
2. **Recommendation Service** - Personalized, trending, similar items, time-based
3. **TF-IDF Search** - Tokenization, term frequency, inverse document frequency
4. **Smart Autocomplete** - Trie data structure, frequency-ranked suggestions
5. **Food Image Classification** - ONNX runtime, on-device inference
6. **Personalization Engine** - Interaction tracking, EMA score update, collaborative filtering
7. **Smart Notifications** - ML-based timing, personalized content
8. **A/B Testing** - Canary bucket assignment for algorithm comparison
9. **NL Order Parsing** - Thai quantity words, modifiers, notes extraction
10. **Recommendation Analytics** - CTR, CVR, revenue tracking

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 74 | Steps 731-740*
