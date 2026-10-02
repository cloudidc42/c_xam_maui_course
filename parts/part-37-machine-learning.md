# Part 37: Machine Learning ใน .NET MAUI
## Steps 361-370: AI/ML Integration

---

## Step 361: ML.NET Overview

```csharp
// ============================================
// ML.NET ใน .NET MAUI
// ============================================

/*
 * ML.NET คืออะไร?
 * - Machine Learning framework สำหรับ .NET
 * - Train model บน PC, run บน mobile
 * - ใช้ได้ใน .NET MAUI โดยตรง
 *
 * Use Cases ใน Mobile Apps:
 * 1. Image Classification - จำแนกประเภทรูปภาพ
 * 2. Sentiment Analysis - วิเคราะห์ความรู้สึก
 * 3. Recommendation - แนะนำสินค้า
 * 4. Anomaly Detection - ตรวจจับความผิดปกติ
 * 5. Forecasting - พยากรณ์ราคา, ยอดขาย
 *
 * NuGet Packages:
 * Microsoft.ML
 * Microsoft.ML.Vision (image)
 * Microsoft.ML.OnnxRuntime (ONNX models)
 */
```

---

## Step 362: Sentiment Analysis

```csharp
// ============================================
// Sentiment Analysis (Text Classification)
// ============================================

using Microsoft.ML;
using Microsoft.ML.Data;

public class SentimentData
{
    [LoadColumn(0)] public string Text { get; set; } = string.Empty;
    [LoadColumn(1), ColumnName("Label")] public bool Sentiment { get; set; }
}

public class SentimentPrediction : SentimentData
{
    [ColumnName("PredictedLabel")] public bool PredictedSentiment { get; set; }
    public float Probability { get; set; }
    public float Score { get; set; }
}

public class SentimentAnalysisService
{
    private readonly MLContext _mlContext;
    private ITransformer? _model;
    private PredictionEngine<SentimentData, SentimentPrediction>? _predEngine;
    
    public SentimentAnalysisService()
    {
        _mlContext = new MLContext(seed: 0);
    }
    
    public void TrainModel(IEnumerable<SentimentData> trainingData)
    {
        var dataView = _mlContext.Data.LoadFromEnumerable(trainingData);
        
        var pipeline = _mlContext.Transforms.Text
            .FeaturizeText("Features", nameof(SentimentData.Text))
            .Append(_mlContext.BinaryClassification.Trainers
                .SdcaLogisticRegression(
                    labelColumnName: "Label",
                    featureColumnName: "Features"));
        
        _model = pipeline.Fit(dataView);
        _predEngine = _mlContext.Model.CreatePredictionEngine<SentimentData, SentimentPrediction>(_model);
    }
    
    public void SaveModel(string path)
    {
        if (_model == null) return;
        _mlContext.Model.Save(_model, null, path);
    }
    
    public void LoadModel(string path)
    {
        _model = _mlContext.Model.Load(path, out _);
        _predEngine = _mlContext.Model.CreatePredictionEngine<SentimentData, SentimentPrediction>(_model);
    }
    
    public SentimentResult Predict(string text)
    {
        if (_predEngine == null)
            throw new InvalidOperationException("Model not loaded");
        
        var prediction = _predEngine.Predict(new SentimentData { Text = text });
        
        return new SentimentResult(
            IsPositive: prediction.PredictedSentiment,
            Confidence: prediction.Probability,
            Label: prediction.PredictedSentiment ? "เชิงบวก" : "เชิงลบ");
    }
    
    // Batch predict
    public List<SentimentResult> PredictBatch(IEnumerable<string> texts)
        => texts.Select(t => Predict(t)).ToList();
}

public record SentimentResult(bool IsPositive, float Confidence, string Label);

// Training data (Thai)
public static class ThaiSentimentData
{
    public static IEnumerable<SentimentData> GetTrainingData() =>
    [
        new() { Text = "สินค้าดีมาก คุณภาพเยี่ยม", Sentiment = true },
        new() { Text = "ส่งเร็วมาก บรรจุภัณฑ์ดี", Sentiment = true },
        new() { Text = "ประทับใจมาก แนะนำเลย", Sentiment = true },
        new() { Text = "คุ้มค่ามาก จะซื้ออีก", Sentiment = true },
        new() { Text = "สินค้าไม่ตรงปก น่าผิดหวัง", Sentiment = false },
        new() { Text = "ส่งช้ามาก ไม่ควรซื้อ", Sentiment = false },
        new() { Text = "คุณภาพแย่มาก สิ้นเปลืองเงิน", Sentiment = false },
        new() { Text = "ผิดหวังมาก ไม่คุ้มค่า", Sentiment = false },
    ];
}
```

---

## Step 363: Product Recommendation

```csharp
// ============================================
// Collaborative Filtering Recommendation
// ============================================

public class ProductRatingData
{
    public float UserId { get; set; }
    public float ProductId { get; set; }
    public float Rating { get; set; }
}

public class ProductRatingPrediction
{
    public float Score { get; set; }
}

public class RecommendationService
{
    private readonly MLContext _ml;
    private ITransformer? _model;
    
    public RecommendationService() => _ml = new MLContext(seed: 0);
    
    public void Train(IEnumerable<ProductRatingData> ratings)
    {
        var dataView = _ml.Data.LoadFromEnumerable(ratings);
        
        var pipeline = _ml.Recommendation().Trainers.MatrixFactorization(
            labelColumnName: nameof(ProductRatingData.Rating),
            matrixColumnIndexColumnName: nameof(ProductRatingData.UserId),
            matrixRowIndexColumnName: nameof(ProductRatingData.ProductId),
            approximationRank: 10,
            learningRate: 0.01f,
            numberOfIterations: 100);
        
        _model = pipeline.Fit(dataView);
    }
    
    public List<ProductRecommendation> GetRecommendations(
        int userId, IEnumerable<int> allProductIds, int count = 5)
    {
        if (_model == null) return [];
        
        var engine = _ml.Model.CreatePredictionEngine<ProductRatingData, ProductRatingPrediction>(_model);
        
        var recommendations = allProductIds
            .Select(productId => new
            {
                ProductId = productId,
                Score = engine.Predict(new ProductRatingData
                {
                    UserId = userId,
                    ProductId = productId
                }).Score
            })
            .OrderByDescending(r => r.Score)
            .Take(count)
            .Select(r => new ProductRecommendation(r.ProductId, r.Score))
            .ToList();
        
        return recommendations;
    }
}

public record ProductRecommendation(int ProductId, float Score);

// ViewModel
public partial class RecommendationsViewModel : ObservableObject
{
    private readonly RecommendationService _recommendations;
    private readonly IProductRepository _products;
    
    [ObservableProperty] private ObservableCollection<Product> _recommendedProducts = new();
    
    public RecommendationsViewModel(
        RecommendationService recommendations, 
        IProductRepository products)
    {
        _recommendations = recommendations;
        _products = products;
    }
    
    [RelayCommand]
    private async Task LoadAsync(int userId)
    {
        var allIds = (await _products.GetAllAsync()).Select(p => p.Id);
        var recs = _recommendations.GetRecommendations(userId, allIds);
        
        var products = new List<Product>();
        foreach (var rec in recs)
        {
            var product = await _products.GetByIdAsync(rec.ProductId);
            if (product != null) products.Add(product);
        }
        
        RecommendedProducts = new ObservableCollection<Product>(products);
    }
}
```

---

## Step 364: Image Classification

```csharp
// ============================================
// Image Classification with ONNX
// ============================================

// Install: Microsoft.ML.OnnxRuntime
// Model: MobileNet, ResNet, EfficientNet

public class ImageClassificationService
{
    private InferenceSession? _session;
    private readonly string[] _labels;
    
    public ImageClassificationService(string modelPath, string[] labels)
    {
        _labels = labels;
        LoadModel(modelPath);
    }
    
    private void LoadModel(string path)
    {
        _session = new InferenceSession(path);
    }
    
    public async Task<ClassificationResult> ClassifyImageAsync(Stream imageStream)
    {
        if (_session == null) throw new InvalidOperationException("Model not loaded");
        
        // Preprocess image
        var tensor = await PreprocessImageAsync(imageStream, 224, 224);
        
        // Run inference
        var inputs = new List<NamedOnnxValue>
        {
            NamedOnnxValue.CreateFromTensor("input", tensor)
        };
        
        using var results = _session.Run(inputs);
        var output = results.First().AsTensor<float>().ToArray();
        
        // Get top prediction
        var maxIndex = Array.IndexOf(output, output.Max());
        var confidence = output[maxIndex];
        
        return new ClassificationResult(
            Label: maxIndex < _labels.Length ? _labels[maxIndex] : "Unknown",
            Confidence: confidence,
            AllScores: output.Select((score, i) => 
                new LabelScore(_labels.ElementAtOrDefault(i) ?? i.ToString(), score))
                .OrderByDescending(s => s.Score)
                .Take(5)
                .ToList());
    }
    
    private static async Task<Microsoft.ML.OnnxRuntime.Tensors.DenseTensor<float>> PreprocessImageAsync(
        Stream imageStream, int width, int height)
    {
        // Load and resize image using SkiaSharp
        using var bitmap = SKBitmap.Decode(imageStream);
        using var resized = bitmap.Resize(new SKImageInfo(width, height), SKFilterQuality.High);
        
        // Normalize to [-1, 1] or [0, 1] based on model
        var tensor = new Microsoft.ML.OnnxRuntime.Tensors.DenseTensor<float>(
            new[] { 1, 3, height, width });
        
        for (var y = 0; y < height; y++)
        for (var x = 0; x < width; x++)
        {
            var pixel = resized.GetPixel(x, y);
            tensor[0, 0, y, x] = (pixel.Red / 255f - 0.485f) / 0.229f;   // R
            tensor[0, 1, y, x] = (pixel.Green / 255f - 0.456f) / 0.224f; // G
            tensor[0, 2, y, x] = (pixel.Blue / 255f - 0.406f) / 0.225f;  // B
        }
        
        return tensor;
    }
}

public record ClassificationResult(
    string Label,
    float Confidence,
    List<LabelScore> AllScores
);

public record LabelScore(string Label, float Score);
```

---

## Step 365: Anomaly Detection

```csharp
// ============================================
// Time Series Anomaly Detection
// ============================================

public class SalesData
{
    [LoadColumn(0)] public float Value { get; set; }
}

public class SalesPrediction
{
    [VectorType(3)]
    public double[] Prediction { get; set; } = Array.Empty<double>();
}

public class AnomalyDetectionService
{
    private readonly MLContext _ml;
    
    public AnomalyDetectionService() => _ml = new MLContext(seed: 0);
    
    public List<AnomalyResult> DetectAnomalies(
        IEnumerable<double> timeSeries, double sensitivity = 95)
    {
        var data = timeSeries.Select(v => new SalesData { Value = (float)v }).ToList();
        var dataView = _ml.Data.LoadFromEnumerable(data);
        
        // Spike detection
        var spikePipeline = _ml.Transforms.DetectIidSpike(
            outputColumnName: "Prediction",
            inputColumnName: nameof(SalesData.Value),
            confidence: sensitivity,
            pvalueHistoryLength: data.Count / 4);
        
        var transformedSpike = spikePipeline.Fit(dataView).Transform(dataView);
        var spikePredictions = _ml.Data.CreateEnumerable<SalesPrediction>(
            transformedSpike, reuseRowObject: false).ToList();
        
        // Change point detection
        var changePipeline = _ml.Transforms.DetectIidChangePoint(
            outputColumnName: "Prediction",
            inputColumnName: nameof(SalesData.Value),
            confidence: sensitivity,
            changeHistoryLength: data.Count / 4);
        
        var transformedChange = changePipeline.Fit(dataView).Transform(dataView);
        var changePredictions = _ml.Data.CreateEnumerable<SalesPrediction>(
            transformedChange, reuseRowObject: false).ToList();
        
        // Combine results
        return data.Select((item, i) =>
        {
            var isSpike = spikePredictions[i].Prediction[0] == 1;
            var isChange = changePredictions[i].Prediction[0] == 1;
            
            return new AnomalyResult(
                Index: i,
                Value: item.Value,
                IsAnomaly: isSpike || isChange,
                IsSpike: isSpike,
                IsChangePoint: isChange,
                SpikeScore: spikePredictions[i].Prediction[1],
                ChangeScore: changePredictions[i].Prediction[1]);
        }).ToList();
    }
}

public record AnomalyResult(
    int Index,
    double Value,
    bool IsAnomaly,
    bool IsSpike,
    bool IsChangePoint,
    double SpikeScore,
    double ChangeScore
);
```

---

## Step 366: Demand Forecasting

```csharp
// ============================================
// Sales Forecasting
// ============================================

public class SalesForecastData
{
    public float Month { get; set; }
    public float Sales { get; set; }
}

public class SalesForecastPrediction
{
    public float[] ForecastedSales { get; set; } = Array.Empty<float>();
    public float[] LowerBoundSales { get; set; } = Array.Empty<float>();
    public float[] UpperBoundSales { get; set; } = Array.Empty<float>();
}

public class SalesForecastingService
{
    private readonly MLContext _ml;
    private ITransformer? _model;
    
    public SalesForecastingService() => _ml = new MLContext();
    
    public void Train(IEnumerable<SalesForecastData> historicalData)
    {
        var dataView = _ml.Data.LoadFromEnumerable(historicalData);
        
        var pipeline = _ml.Forecasting.ForecastBySsa(
            outputColumnName: "ForecastedSales",
            inputColumnName: nameof(SalesForecastData.Sales),
            windowSize: 3,
            seriesLength: 12,
            trainSize: historicalData.Count(),
            horizon: 3,
            confidenceLevel: 0.95f,
            confidenceLowerBoundColumn: "LowerBoundSales",
            confidenceUpperBoundColumn: "UpperBoundSales");
        
        _model = pipeline.Fit(dataView);
    }
    
    public ForecastResult Forecast(int months = 3)
    {
        if (_model == null) throw new InvalidOperationException("Model not trained");
        
        var engine = _model.CreateTimeSeriesEngine<SalesForecastData, SalesForecastPrediction>(_ml);
        var prediction = engine.Predict();
        
        return new ForecastResult(
            Forecasts: prediction.ForecastedSales
                .Select((v, i) => new MonthForecast(i + 1, v,
                    prediction.LowerBoundSales[i],
                    prediction.UpperBoundSales[i]))
                .ToList());
    }
}

public record ForecastResult(List<MonthForecast> Forecasts);
public record MonthForecast(int Month, float Value, float Lower, float Upper);
```

---

## Step 367: Object Detection

```csharp
// ============================================
// Object Detection (YOLO via ONNX)
// ============================================

public class ObjectDetectionService
{
    private readonly InferenceSession? _session;
    private readonly string[] _labels;
    private const float ConfidenceThreshold = 0.5f;
    private const float NmsThreshold = 0.45f;
    
    public ObjectDetectionService(byte[] modelBytes, string[] labels)
    {
        _session = new InferenceSession(modelBytes);
        _labels = labels;
    }
    
    public async Task<List<DetectedObject>> DetectAsync(Stream imageStream)
    {
        if (_session == null) return [];
        
        // Load and preprocess
        using var bitmap = SKBitmap.Decode(imageStream);
        var originalWidth = bitmap.Width;
        var originalHeight = bitmap.Height;
        
        using var resized = bitmap.Resize(new SKImageInfo(416, 416), SKFilterQuality.High);
        var input = CreateInputTensor(resized);
        
        // Run inference
        var inputs = new[] { NamedOnnxValue.CreateFromTensor("input", input) };
        using var outputs = _session.Run(inputs);
        
        // Parse YOLO output
        var output = outputs.First().AsTensor<float>();
        var detections = ParseYoloOutput(output, originalWidth, originalHeight);
        
        // Apply NMS
        return ApplyNms(detections, NmsThreshold);
    }
    
    private Microsoft.ML.OnnxRuntime.Tensors.DenseTensor<float> CreateInputTensor(SKBitmap bitmap)
    {
        var tensor = new Microsoft.ML.OnnxRuntime.Tensors.DenseTensor<float>(
            new[] { 1, 3, 416, 416 });
        
        for (var y = 0; y < 416; y++)
        for (var x = 0; x < 416; x++)
        {
            var pixel = bitmap.GetPixel(x, y);
            tensor[0, 0, y, x] = pixel.Red / 255f;
            tensor[0, 1, y, x] = pixel.Green / 255f;
            tensor[0, 2, y, x] = pixel.Blue / 255f;
        }
        
        return tensor;
    }
    
    private List<DetectedObject> ParseYoloOutput(
        Microsoft.ML.OnnxRuntime.Tensors.Tensor<float> output,
        int origWidth, int origHeight)
    {
        // YOLO output parsing (simplified)
        var detections = new List<DetectedObject>();
        // Implementation varies by YOLO version
        return detections;
    }
    
    private static List<DetectedObject> ApplyNms(
        List<DetectedObject> detections, float threshold)
    {
        // Non-Maximum Suppression
        var result = new List<DetectedObject>();
        var sorted = detections.OrderByDescending(d => d.Confidence).ToList();
        
        while (sorted.Count > 0)
        {
            var best = sorted[0];
            result.Add(best);
            sorted.RemoveAt(0);
            
            sorted.RemoveAll(d => 
                d.Label == best.Label && 
                CalculateIou(best.BoundingBox, d.BoundingBox) > threshold);
        }
        
        return result;
    }
    
    private static float CalculateIou(Rect a, Rect b)
    {
        var intersectionLeft = Math.Max(a.Left, b.Left);
        var intersectionTop = Math.Max(a.Top, b.Top);
        var intersectionRight = Math.Min(a.Right, b.Right);
        var intersectionBottom = Math.Min(a.Bottom, b.Bottom);
        
        if (intersectionLeft >= intersectionRight || intersectionTop >= intersectionBottom)
            return 0;
        
        var intersectionArea = (intersectionRight - intersectionLeft) *
                               (intersectionBottom - intersectionTop);
        var unionArea = a.Width * a.Height + b.Width * b.Height - intersectionArea;
        
        return intersectionArea / unionArea;
    }
}

public record DetectedObject(
    string Label,
    float Confidence,
    Rect BoundingBox
);
```

---

## Step 368: On-Device AI

```csharp
// ============================================
// On-Device AI (No Internet Required)
// ============================================

public class LocalAIService
{
    // Text summarization using lightweight model
    public string Summarize(string text, int maxSentences = 3)
    {
        // Simple extractive summarization
        var sentences = SplitIntoSentences(text);
        
        if (sentences.Count <= maxSentences) return text;
        
        // Score sentences by importance
        var wordFreq = CalculateWordFrequency(text);
        var scored = sentences.Select(s => new
        {
            Sentence = s,
            Score = s.Split(' ').Sum(w => wordFreq.GetValueOrDefault(w.ToLower(), 0))
        });
        
        return string.Join(" ", scored
            .OrderByDescending(s => s.Score)
            .Take(maxSentences)
            .Select(s => s.Sentence));
    }
    
    private static List<string> SplitIntoSentences(string text)
    {
        return text.Split(new[] { '.', '!', '?' }, StringSplitOptions.RemoveEmptyEntries)
            .Select(s => s.Trim())
            .Where(s => s.Length > 10)
            .ToList();
    }
    
    private static Dictionary<string, int> CalculateWordFrequency(string text)
    {
        var stopWords = new HashSet<string> { "ที่", "และ", "ใน", "ของ", "กับ", "เป็น", "มี", "จาก" };
        
        return text.ToLower()
            .Split(' ', StringSplitOptions.RemoveEmptyEntries)
            .Where(w => !stopWords.Contains(w))
            .GroupBy(w => w)
            .ToDictionary(g => g.Key, g => g.Count());
    }
    
    // Fuzzy string matching (for search)
    public static double CalculateSimilarity(string s1, string s2)
    {
        if (string.IsNullOrEmpty(s1) || string.IsNullOrEmpty(s2)) return 0;
        
        var distance = LevenshteinDistance(s1.ToLower(), s2.ToLower());
        var maxLen = Math.Max(s1.Length, s2.Length);
        
        return 1.0 - (double)distance / maxLen;
    }
    
    private static int LevenshteinDistance(string s, string t)
    {
        var m = s.Length;
        var n = t.Length;
        var d = new int[m + 1, n + 1];
        
        for (var i = 0; i <= m; i++) d[i, 0] = i;
        for (var j = 0; j <= n; j++) d[0, j] = j;
        
        for (var i = 1; i <= m; i++)
        for (var j = 1; j <= n; j++)
        {
            var cost = s[i - 1] == t[j - 1] ? 0 : 1;
            d[i, j] = Math.Min(
                Math.Min(d[i - 1, j] + 1, d[i, j - 1] + 1),
                d[i - 1, j - 1] + cost);
        }
        
        return d[m, n];
    }
}
```

---

## Step 369: AI Chat Integration

```csharp
// ============================================
// OpenAI/Claude API Integration
// ============================================

public class AIChatService
{
    private readonly HttpClient _http;
    private const string ApiUrl = "https://api.anthropic.com/v1/messages";
    
    public AIChatService(HttpClient http) => _http = http;
    
    public async IAsyncEnumerable<string> StreamChatAsync(
        string userMessage,
        string systemPrompt = "คุณเป็นผู้ช่วย AI ที่เป็นมิตร ตอบเป็นภาษาไทย",
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
    {
        var request = new
        {
            model = "claude-haiku-4-5-20251001",
            max_tokens = 1024,
            stream = true,
            system = systemPrompt,
            messages = new[] { new { role = "user", content = userMessage } }
        };
        
        using var content = new StringContent(
            System.Text.Json.JsonSerializer.Serialize(request),
            System.Text.Encoding.UTF8,
            "application/json");
        
        using var response = await _http.PostAsync(ApiUrl, content, ct);
        response.EnsureSuccessStatusCode();
        
        using var stream = await response.Content.ReadAsStreamAsync(ct);
        using var reader = new System.IO.StreamReader(stream);
        
        while (!reader.EndOfStream && !ct.IsCancellationRequested)
        {
            var line = await reader.ReadLineAsync(ct);
            if (string.IsNullOrEmpty(line) || !line.StartsWith("data: ")) continue;
            
            var json = line["data: ".Length..];
            if (json == "[DONE]") break;
            
            // Parse SSE delta
            var doc = System.Text.Json.JsonDocument.Parse(json);
            if (doc.RootElement.TryGetProperty("delta", out var delta) &&
                delta.TryGetProperty("text", out var text))
            {
                yield return text.GetString() ?? string.Empty;
            }
        }
    }
}

public partial class AIChatViewModel : ObservableObject
{
    private readonly AIChatService _ai;
    
    [ObservableProperty] private ObservableCollection<ChatMessage> _messages = new();
    [ObservableProperty] private string _inputText = string.Empty;
    [ObservableProperty] private bool _isGenerating;
    
    public AIChatViewModel(AIChatService ai) => _ai = ai;
    
    [RelayCommand]
    private async Task SendAsync()
    {
        if (string.IsNullOrWhiteSpace(InputText) || IsGenerating) return;
        
        var userMessage = InputText;
        InputText = string.Empty;
        
        Messages.Add(new ChatMessage("user", "ฉัน", userMessage, DateTimeOffset.Now));
        
        var aiMessage = new ChatMessage("assistant", "AI", string.Empty, DateTimeOffset.Now);
        Messages.Add(aiMessage);
        IsGenerating = true;
        
        try
        {
            var sb = new System.Text.StringBuilder();
            
            await foreach (var chunk in _ai.StreamChatAsync(userMessage))
            {
                sb.Append(chunk);
                // Update message content
                var updatedMessage = aiMessage with { Content = sb.ToString() };
                var idx = Messages.IndexOf(aiMessage);
                if (idx >= 0) Messages[idx] = updatedMessage;
            }
        }
        finally
        {
            IsGenerating = false;
        }
    }
}
```

---

## Step 370: Model Management

```csharp
// ============================================
// ML Model Management
// ============================================

public class ModelManager
{
    private readonly string _modelDir;
    private readonly HttpClient _http;
    
    public ModelManager(HttpClient http)
    {
        _http = http;
        _modelDir = Path.Combine(FileSystem.AppDataDirectory, "models");
        Directory.CreateDirectory(_modelDir);
    }
    
    public async Task<string> EnsureModelAsync(
        string modelName, string downloadUrl,
        IProgress<double>? progress = null)
    {
        var modelPath = Path.Combine(_modelDir, modelName);
        
        if (File.Exists(modelPath))
            return modelPath;
        
        // Download model
        await DownloadModelAsync(downloadUrl, modelPath, progress);
        return modelPath;
    }
    
    private async Task DownloadModelAsync(
        string url, string savePath, IProgress<double>? progress)
    {
        using var response = await _http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead);
        response.EnsureSuccessStatusCode();
        
        var total = response.Content.Headers.ContentLength ?? -1;
        
        using var source = await response.Content.ReadAsStreamAsync();
        using var dest = File.Create(savePath);
        
        var buffer = new byte[8192];
        long downloaded = 0;
        int read;
        
        while ((read = await source.ReadAsync(buffer)) > 0)
        {
            await dest.WriteAsync(buffer.AsMemory(0, read));
            downloaded += read;
            
            if (total > 0)
                progress?.Report((double)downloaded / total);
        }
    }
    
    public bool IsModelAvailable(string modelName)
        => File.Exists(Path.Combine(_modelDir, modelName));
    
    public void DeleteModel(string modelName)
    {
        var path = Path.Combine(_modelDir, modelName);
        if (File.Exists(path)) File.Delete(path);
    }
    
    public long GetModelSize(string modelName)
    {
        var path = Path.Combine(_modelDir, modelName);
        return File.Exists(path) ? new FileInfo(path).Length : 0;
    }
    
    public IEnumerable<(string Name, long Size)> ListModels()
    {
        return Directory.GetFiles(_modelDir)
            .Select(f => (Name: Path.GetFileName(f), Size: new FileInfo(f).Length));
    }
}
```

---

## สรุป Part 37

ใน Part 37 เราได้เรียนรู้:

1. **ML.NET Overview** - เมื่อไหร่ควรใช้
2. **Sentiment Analysis** - Text classification
3. **Recommendation** - Collaborative filtering
4. **Image Classification** - ONNX model
5. **Anomaly Detection** - Spike/change point
6. **Demand Forecasting** - Time series SSA
7. **Object Detection** - YOLO via ONNX
8. **On-Device AI** - Text summarization, fuzzy search
9. **AI Chat** - Claude/OpenAI streaming API
10. **Model Management** - Download, cache models

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 37 | Steps 361-370*
