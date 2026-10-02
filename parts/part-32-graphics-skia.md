# Part 32: Graphics with SkiaSharp
## Steps 311-320: Custom Drawing & Charts

---

## Step 311: SkiaSharp Basics

```csharp
// ============================================
// SkiaSharp ใน .NET MAUI
// ============================================

// Install NuGet:
// SkiaSharp.Views.Maui.Controls
// SkiaSharp.Views.Maui.Core

// XAML:
/*
<ContentPage xmlns:skia="clr-namespace:SkiaSharp.Views.Maui.Controls;assembly=SkiaSharp.Views.Maui.Controls">
    <skia:SKCanvasView PaintSurface="OnPaintSurface" 
                       EnableTouchEvents="True" />
</ContentPage>
*/

using SkiaSharp;
using SkiaSharp.Views.Maui;
using SkiaSharp.Views.Maui.Controls;

public partial class DrawingPage : ContentPage
{
    private void OnPaintSurface(object sender, SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColors.White);
        
        // Basic shapes
        using var paint = new SKPaint
        {
            Color = SKColors.Blue,
            StrokeWidth = 3,
            IsAntialias = true,
            Style = SKPaintStyle.Stroke
        };
        
        // Draw rectangle
        canvas.DrawRect(50, 50, 200, 100, paint);
        
        // Draw circle
        paint.Color = SKColors.Red;
        paint.Style = SKPaintStyle.Fill;
        canvas.DrawCircle(info.Width / 2f, info.Height / 2f, 80, paint);
        
        // Draw line
        paint.Color = SKColors.Green;
        paint.Style = SKPaintStyle.Stroke;
        canvas.DrawLine(0, 0, info.Width, info.Height, paint);
        
        // Draw text
        using var textPaint = new SKPaint
        {
            Color = SKColors.Black,
            TextSize = 24,
            IsAntialias = true,
            Typeface = SKTypeface.Default
        };
        canvas.DrawText("Hello SkiaSharp!", 50, info.Height - 50, textPaint);
        
        // Draw path
        using var path = new SKPath();
        path.MoveTo(100, 200);
        path.LineTo(200, 100);
        path.LineTo(300, 200);
        path.Close();
        
        paint.Color = new SKColor(255, 165, 0, 180); // Orange semi-transparent
        paint.Style = SKPaintStyle.Fill;
        canvas.DrawPath(path, paint);
    }
}
```

---

## Step 312: Gradients & Effects

```csharp
// ============================================
// Gradients and Visual Effects
// ============================================

private void DrawGradients(SKCanvas canvas, SKImageInfo info)
{
    // Linear gradient
    using var linearGradient = SKShader.CreateLinearGradient(
        new SKPoint(0, 0),
        new SKPoint(info.Width, info.Height),
        new[] { SKColors.Blue, SKColors.Purple, SKColors.Red },
        new[] { 0f, 0.5f, 1f },
        SKShaderTileMode.Clamp);
    
    using var gradientPaint = new SKPaint
    {
        Shader = linearGradient,
        IsAntialias = true
    };
    
    canvas.DrawRect(0, 0, info.Width, 200, gradientPaint);
    
    // Radial gradient
    using var radialGradient = SKShader.CreateRadialGradient(
        new SKPoint(info.Width / 2f, 300),
        150,
        new[] { SKColors.Yellow, SKColors.Orange, SKColor.Empty },
        new[] { 0f, 0.5f, 1f },
        SKShaderTileMode.Clamp);
    
    gradientPaint.Shader = radialGradient;
    canvas.DrawCircle(info.Width / 2f, 300, 150, gradientPaint);
    
    // Blur effect
    using var blurPaint = new SKPaint
    {
        IsAntialias = true,
        MaskFilter = SKMaskFilter.CreateBlur(SKBlurStyle.Normal, 5)
    };
    blurPaint.Color = SKColors.Black.WithAlpha(100);
    canvas.DrawRoundRect(50, 400, 300, 100, 20, 20, blurPaint); // Shadow
    
    // Gradient text
    using var textShader = SKShader.CreateLinearGradient(
        new SKPoint(0, 0),
        new SKPoint(400, 0),
        new[] { SKColors.Red, SKColors.Blue },
        null,
        SKShaderTileMode.Clamp);
    
    using var textPaint = new SKPaint
    {
        TextSize = 40,
        Shader = textShader,
        IsAntialias = true
    };
    canvas.DrawText("Gradient Text", 50, 500, textPaint);
}
```

---

## Step 313: Bar Chart

```csharp
// ============================================
// Bar Chart Custom View
// ============================================

public class BarChartView : SKCanvasView
{
    public static readonly BindableProperty DataProperty =
        BindableProperty.Create(nameof(Data), typeof(List<ChartData>), typeof(BarChartView),
            propertyChanged: (b, _, _) => ((BarChartView)b).InvalidateSurface());
    
    public static readonly BindableProperty BarColorProperty =
        BindableProperty.Create(nameof(BarColor), typeof(Color), typeof(BarChartView), Colors.Blue,
            propertyChanged: (b, _, _) => ((BarChartView)b).InvalidateSurface());
    
    public List<ChartData>? Data
    {
        get => (List<ChartData>?)GetValue(DataProperty);
        set => SetValue(DataProperty, value);
    }
    
    public Color BarColor
    {
        get => (Color)GetValue(BarColorProperty);
        set => SetValue(BarColorProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColors.White);
        
        if (Data == null || Data.Count == 0) return;
        
        var padding = 40f;
        var chartWidth = info.Width - padding * 2;
        var chartHeight = info.Height - padding * 2 - 30; // Bottom for labels
        
        var maxValue = Data.Max(d => d.Value);
        var barWidth = chartWidth / Data.Count;
        var barSpacing = barWidth * 0.2f;
        
        // Draw axes
        using var axisPaint = new SKPaint
        {
            Color = SKColors.Gray,
            StrokeWidth = 1,
            IsAntialias = true
        };
        
        canvas.DrawLine(padding, padding, padding, padding + chartHeight, axisPaint);
        canvas.DrawLine(padding, padding + chartHeight, padding + chartWidth, padding + chartHeight, axisPaint);
        
        // Draw bars
        var barColor = new SKColor(
            (byte)(BarColor.Red * 255),
            (byte)(BarColor.Green * 255),
            (byte)(BarColor.Blue * 255));
        
        using var barPaint = new SKPaint
        {
            Color = barColor,
            IsAntialias = true,
            Style = SKPaintStyle.Fill
        };
        
        using var labelPaint = new SKPaint
        {
            Color = SKColors.DarkGray,
            TextSize = 12,
            IsAntialias = true,
            TextAlign = SKTextAlign.Center
        };
        
        for (var i = 0; i < Data.Count; i++)
        {
            var item = Data[i];
            var barHeight = (float)(item.Value / maxValue * chartHeight);
            var x = padding + i * barWidth + barSpacing / 2;
            var y = padding + chartHeight - barHeight;
            var w = barWidth - barSpacing;
            
            // Bar with rounded top
            using var barPath = new SKPath();
            barPath.AddRoundRect(
                new SKRoundRect(new SKRect(x, y, x + w, y + barHeight), 4, 4));
            
            // Gradient bar
            using var barGradient = SKShader.CreateLinearGradient(
                new SKPoint(x, y), new SKPoint(x, y + barHeight),
                new[] { barColor, barColor.WithAlpha(180) },
                null, SKShaderTileMode.Clamp);
            
            barPaint.Shader = barGradient;
            canvas.DrawPath(barPath, barPaint);
            
            // Label
            canvas.DrawText(item.Label, x + w / 2, padding + chartHeight + 20, labelPaint);
            
            // Value on top
            labelPaint.Color = SKColors.DarkSlateGray;
            canvas.DrawText(item.Value.ToString("N0"), x + w / 2, y - 5, labelPaint);
        }
    }
}

public record ChartData(string Label, double Value);
```

---

## Step 314: Line Chart

```csharp
// ============================================
// Line Chart
// ============================================

public class LineChartView : SKCanvasView
{
    public static readonly BindableProperty PointsProperty =
        BindableProperty.Create(nameof(Points), typeof(List<(float X, float Y)>), 
            typeof(LineChartView),
            propertyChanged: (b, _, _) => ((LineChartView)b).InvalidateSurface());
    
    public List<(float X, float Y)>? Points
    {
        get => (List<(float X, float Y)>?)GetValue(PointsProperty);
        set => SetValue(PointsProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColors.White);
        
        if (Points == null || Points.Count < 2) return;
        
        var padding = 40f;
        var chartWidth = info.Width - padding * 2;
        var chartHeight = info.Height - padding * 2;
        
        var minX = Points.Min(p => p.X);
        var maxX = Points.Max(p => p.X);
        var minY = Points.Min(p => p.Y);
        var maxY = Points.Max(p => p.Y);
        
        SKPoint Map(float x, float y) => new(
            padding + (x - minX) / (maxX - minX) * chartWidth,
            padding + chartHeight - (y - minY) / (maxY - minY) * chartHeight
        );
        
        // Draw grid
        using var gridPaint = new SKPaint
        {
            Color = SKColors.LightGray,
            StrokeWidth = 0.5f
        };
        for (var i = 0; i <= 5; i++)
        {
            var y = padding + chartHeight / 5 * i;
            canvas.DrawLine(padding, y, padding + chartWidth, y, gridPaint);
        }
        
        // Draw area fill
        using var areaPath = new SKPath();
        var firstPoint = Map(Points[0].X, Points[0].Y);
        areaPath.MoveTo(firstPoint.X, padding + chartHeight);
        areaPath.LineTo(firstPoint.X, firstPoint.Y);
        
        for (var i = 1; i < Points.Count; i++)
        {
            var prev = Map(Points[i - 1].X, Points[i - 1].Y);
            var curr = Map(Points[i].X, Points[i].Y);
            var cp1 = new SKPoint((prev.X + curr.X) / 2, prev.Y);
            var cp2 = new SKPoint((prev.X + curr.X) / 2, curr.Y);
            areaPath.CubicTo(cp1, cp2, curr);
        }
        
        var lastPoint = Map(Points[^1].X, Points[^1].Y);
        areaPath.LineTo(lastPoint.X, padding + chartHeight);
        areaPath.Close();
        
        using var areaGradient = SKShader.CreateLinearGradient(
            new SKPoint(0, padding), new SKPoint(0, padding + chartHeight),
            new[] { new SKColor(33, 150, 243, 100), new SKColor(33, 150, 243, 10) },
            null, SKShaderTileMode.Clamp);
        
        using var areaPaint = new SKPaint { Shader = areaGradient };
        canvas.DrawPath(areaPath, areaPaint);
        
        // Draw line
        using var linePath = new SKPath();
        firstPoint = Map(Points[0].X, Points[0].Y);
        linePath.MoveTo(firstPoint.X, firstPoint.Y);
        
        for (var i = 1; i < Points.Count; i++)
        {
            var prev = Map(Points[i - 1].X, Points[i - 1].Y);
            var curr = Map(Points[i].X, Points[i].Y);
            var cp1 = new SKPoint((prev.X + curr.X) / 2, prev.Y);
            var cp2 = new SKPoint((prev.X + curr.X) / 2, curr.Y);
            linePath.CubicTo(cp1, cp2, curr);
        }
        
        using var linePaint = new SKPaint
        {
            Color = new SKColor(33, 150, 243),
            StrokeWidth = 2.5f,
            Style = SKPaintStyle.Stroke,
            IsAntialias = true,
            StrokeCap = SKStrokeCap.Round,
            StrokeJoin = SKStrokeJoin.Round
        };
        canvas.DrawPath(linePath, linePaint);
        
        // Draw dots
        using var dotPaint = new SKPaint { Color = SKColors.White, IsAntialias = true };
        using var dotBorderPaint = new SKPaint { Color = new SKColor(33, 150, 243), IsAntialias = true };
        
        foreach (var p in Points)
        {
            var mapped = Map(p.X, p.Y);
            canvas.DrawCircle(mapped.X, mapped.Y, 5, dotBorderPaint);
            canvas.DrawCircle(mapped.X, mapped.Y, 3, dotPaint);
        }
    }
}
```

---

## Step 315: Pie Chart

```csharp
// ============================================
// Pie/Donut Chart
// ============================================

public class PieChartView : SKCanvasView
{
    private static readonly SKColor[] DefaultColors =
    {
        new(33, 150, 243), new(244, 67, 54), new(76, 175, 80),
        new(255, 152, 0), new(156, 39, 176), new(0, 188, 212)
    };
    
    public static readonly BindableProperty SlicesProperty =
        BindableProperty.Create(nameof(Slices), typeof(List<PieSlice>), typeof(PieChartView),
            propertyChanged: (b, _, _) => ((PieChartView)b).InvalidateSurface());
    
    public static readonly BindableProperty IsDonutProperty =
        BindableProperty.Create(nameof(IsDonut), typeof(bool), typeof(PieChartView), false,
            propertyChanged: (b, _, _) => ((PieChartView)b).InvalidateSurface());
    
    public List<PieSlice>? Slices
    {
        get => (List<PieSlice>?)GetValue(SlicesProperty);
        set => SetValue(SlicesProperty, value);
    }
    
    public bool IsDonut
    {
        get => (bool)GetValue(IsDonutProperty);
        set => SetValue(IsDonutProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColors.White);
        
        if (Slices == null || Slices.Count == 0) return;
        
        var cx = info.Width / 2f;
        var cy = info.Height / 2f;
        var radius = Math.Min(cx, cy) * 0.8f;
        var innerRadius = IsDonut ? radius * 0.5f : 0;
        
        var total = Slices.Sum(s => s.Value);
        var startAngle = -90f; // Start from top
        
        using var paint = new SKPaint { IsAntialias = true, Style = SKPaintStyle.Fill };
        using var textPaint = new SKPaint
        {
            Color = SKColors.White,
            TextSize = 14,
            IsAntialias = true,
            TextAlign = SKTextAlign.Center
        };
        
        for (var i = 0; i < Slices.Count; i++)
        {
            var slice = Slices[i];
            var sweepAngle = (float)(slice.Value / total * 360);
            var midAngle = startAngle + sweepAngle / 2;
            var colorIndex = i % DefaultColors.Length;
            
            paint.Color = DefaultColors[colorIndex];
            
            using var path = new SKPath();
            path.MoveTo(cx, cy);
            path.ArcTo(new SKRect(cx - radius, cy - radius, cx + radius, cy + radius),
                startAngle, sweepAngle, false);
            path.Close();
            
            canvas.DrawPath(path, paint);
            
            // Cut inner circle for donut
            if (IsDonut)
            {
                paint.Color = SKColors.White;
                canvas.DrawCircle(cx, cy, innerRadius, paint);
            }
            
            // Draw label
            var labelRadius = (radius + innerRadius) / 2;
            var lx = cx + labelRadius * (float)Math.Cos(midAngle * Math.PI / 180);
            var ly = cy + labelRadius * (float)Math.Sin(midAngle * Math.PI / 180);
            
            if (sweepAngle > 20)
            {
                canvas.DrawText($"{slice.Value / total:P0}", lx, ly + 5, textPaint);
            }
            
            startAngle += sweepAngle;
        }
        
        // Center text for donut
        if (IsDonut && !string.IsNullOrEmpty(Slices.FirstOrDefault()?.Label))
        {
            textPaint.Color = SKColors.DarkGray;
            textPaint.TextSize = 20;
            canvas.DrawText("รวม", cx, cy - 10, textPaint);
            textPaint.TextSize = 28;
            canvas.DrawText(total.ToString("N0"), cx, cy + 25, textPaint);
        }
        
        // Legend
        DrawLegend(canvas, info, total);
    }
    
    private void DrawLegend(SKCanvas canvas, SKImageInfo info, double total)
    {
        if (Slices == null) return;
        
        using var legendPaint = new SKPaint { IsAntialias = true };
        using var textPaint = new SKPaint
        {
            Color = SKColors.DarkGray,
            TextSize = 12,
            IsAntialias = true
        };
        
        var legendY = info.Height - (Slices.Count * 20 + 10);
        
        for (var i = 0; i < Slices.Count; i++)
        {
            var colorIndex = i % DefaultColors.Length;
            legendPaint.Color = DefaultColors[colorIndex];
            canvas.DrawRect(10, legendY + i * 20, 12, 12, legendPaint);
            canvas.DrawText($"{Slices[i].Label} ({Slices[i].Value:N0})", 
                28, legendY + i * 20 + 10, textPaint);
        }
    }
}

public record PieSlice(string Label, double Value);
```

---

## Step 316: Interactive Drawing

```csharp
// ============================================
// Touch Drawing Canvas
// ============================================

public class DrawingCanvasView : SKCanvasView
{
    private readonly List<DrawingPath> _paths = new();
    private DrawingPath? _currentPath;
    
    public static readonly BindableProperty StrokeColorProperty =
        BindableProperty.Create(nameof(StrokeColor), typeof(Color), typeof(DrawingCanvasView), Colors.Black);
    
    public static readonly BindableProperty StrokeWidthProperty =
        BindableProperty.Create(nameof(StrokeWidth), typeof(float), typeof(DrawingCanvasView), 3f);
    
    public Color StrokeColor
    {
        get => (Color)GetValue(StrokeColorProperty);
        set => SetValue(StrokeColorProperty, value);
    }
    
    public float StrokeWidth
    {
        get => (float)GetValue(StrokeWidthProperty);
        set => SetValue(StrokeWidthProperty, value);
    }
    
    public DrawingCanvasView()
    {
        EnableTouchEvents = true;
        Touch += OnTouch;
    }
    
    private void OnTouch(object? sender, SKTouchEventArgs e)
    {
        switch (e.ActionType)
        {
            case SKTouchAction.Pressed:
                _currentPath = new DrawingPath(
                    new SKPaint
                    {
                        Color = StrokeColor.ToSKColor(),
                        StrokeWidth = StrokeWidth,
                        Style = SKPaintStyle.Stroke,
                        IsAntialias = true,
                        StrokeCap = SKStrokeCap.Round,
                        StrokeJoin = SKStrokeJoin.Round
                    });
                _currentPath.Path.MoveTo(e.Location);
                _paths.Add(_currentPath);
                break;
            
            case SKTouchAction.Moved:
                _currentPath?.Path.LineTo(e.Location);
                InvalidateSurface();
                break;
            
            case SKTouchAction.Released:
                _currentPath = null;
                break;
        }
        
        e.Handled = true;
    }
    
    public void Undo()
    {
        if (_paths.Count > 0)
        {
            _paths.RemoveAt(_paths.Count - 1);
            InvalidateSurface();
        }
    }
    
    public void Clear()
    {
        _paths.Clear();
        InvalidateSurface();
    }
    
    public async Task<byte[]> ExportAsync()
    {
        // Create bitmap and draw all paths
        using var bitmap = new SKBitmap((int)Width, (int)Height);
        using var canvas = new SKCanvas(bitmap);
        canvas.Clear(SKColors.White);
        
        foreach (var drawing in _paths)
            canvas.DrawPath(drawing.Path, drawing.Paint);
        
        using var data = bitmap.Encode(SKEncodedImageFormat.Png, 100);
        return data.ToArray();
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.White);
        
        foreach (var drawing in _paths)
            canvas.DrawPath(drawing.Path, drawing.Paint);
    }
}

public class DrawingPath
{
    public SKPath Path { get; } = new();
    public SKPaint Paint { get; }
    
    public DrawingPath(SKPaint paint) => Paint = paint;
}
```

---

## Step 317: Sparkline / Mini Chart

```csharp
// ============================================
// Sparkline for Lists
// ============================================

public class SparklineView : SKCanvasView
{
    public static readonly BindableProperty ValuesProperty =
        BindableProperty.Create(nameof(Values), typeof(IList<double>), typeof(SparklineView),
            propertyChanged: (b, _, _) => ((SparklineView)b).InvalidateSurface());
    
    public static readonly BindableProperty LineColorProperty =
        BindableProperty.Create(nameof(LineColor), typeof(Color), typeof(SparklineView), Colors.Blue,
            propertyChanged: (b, _, _) => ((SparklineView)b).InvalidateSurface());
    
    public IList<double>? Values
    {
        get => (IList<double>?)GetValue(ValuesProperty);
        set => SetValue(ValuesProperty, value);
    }
    
    public Color LineColor
    {
        get => (Color)GetValue(LineColorProperty);
        set => SetValue(LineColorProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColor.Empty);
        
        if (Values == null || Values.Count < 2) return;
        
        var max = Values.Max();
        var min = Values.Min();
        var range = max - min;
        if (range == 0) range = 1;
        
        var w = info.Width;
        var h = info.Height;
        var xStep = (float)w / (Values.Count - 1);
        
        var points = Values.Select((v, i) => new SKPoint(
            i * xStep,
            (float)(h - (v - min) / range * h * 0.9 - h * 0.05)
        )).ToArray();
        
        using var path = new SKPath();
        path.MoveTo(points[0]);
        for (var i = 1; i < points.Length; i++)
            path.LineTo(points[i]);
        
        var skColor = LineColor.ToSKColor();
        
        using var linePaint = new SKPaint
        {
            Color = skColor,
            StrokeWidth = 1.5f,
            Style = SKPaintStyle.Stroke,
            IsAntialias = true
        };
        canvas.DrawPath(path, linePaint);
        
        // Last point dot
        using var dotPaint = new SKPaint { Color = skColor, IsAntialias = true };
        canvas.DrawCircle(points[^1].X, points[^1].Y, 3, dotPaint);
    }
}
```

---

## Step 318: Progress Ring

```csharp
// ============================================
// Animated Progress Ring
// ============================================

public class ProgressRingView : SKCanvasView
{
    private float _animatedProgress;
    private Animation? _animation;
    
    public static readonly BindableProperty ProgressProperty =
        BindableProperty.Create(nameof(Progress), typeof(double), typeof(ProgressRingView), 0.0,
            propertyChanged: OnProgressChanged);
    
    public static readonly BindableProperty TrackColorProperty =
        BindableProperty.Create(nameof(TrackColor), typeof(Color), typeof(ProgressRingView), 
            Color.FromRgb(220, 220, 220));
    
    public static readonly BindableProperty ProgressColorProperty =
        BindableProperty.Create(nameof(ProgressColor), typeof(Color), typeof(ProgressRingView), Colors.Blue);
    
    public double Progress
    {
        get => (double)GetValue(ProgressProperty);
        set => SetValue(ProgressProperty, value);
    }
    
    public Color TrackColor
    {
        get => (Color)GetValue(TrackColorProperty);
        set => SetValue(TrackColorProperty, value);
    }
    
    public Color ProgressColor
    {
        get => (Color)GetValue(ProgressColorProperty);
        set => SetValue(ProgressColorProperty, value);
    }
    
    private static void OnProgressChanged(BindableObject b, object oldValue, object newValue)
    {
        var view = (ProgressRingView)b;
        view.AnimateTo((double)newValue);
    }
    
    private void AnimateTo(double target)
    {
        _animation?.Dispose();
        var start = _animatedProgress;
        
        _animation = new Animation(v =>
        {
            _animatedProgress = (float)v;
            InvalidateSurface();
        }, start, (float)target);
        
        _animation.Commit(this, "progress", length: 600, easing: Easing.CubicOut);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColor.Empty);
        
        var cx = info.Width / 2f;
        var cy = info.Height / 2f;
        var strokeWidth = Math.Min(cx, cy) * 0.15f;
        var radius = Math.Min(cx, cy) - strokeWidth / 2 - 5;
        
        var rect = new SKRect(cx - radius, cy - radius, cx + radius, cy + radius);
        
        // Track
        using var trackPaint = new SKPaint
        {
            Color = TrackColor.ToSKColor(),
            StrokeWidth = strokeWidth,
            Style = SKPaintStyle.Stroke,
            IsAntialias = true,
            StrokeCap = SKStrokeCap.Round
        };
        canvas.DrawCircle(cx, cy, radius, trackPaint);
        
        // Progress arc
        if (_animatedProgress > 0)
        {
            using var progressPaint = new SKPaint
            {
                Color = ProgressColor.ToSKColor(),
                StrokeWidth = strokeWidth,
                Style = SKPaintStyle.Stroke,
                IsAntialias = true,
                StrokeCap = SKStrokeCap.Round
            };
            
            canvas.DrawArc(rect, -90, _animatedProgress * 360f, false, progressPaint);
        }
        
        // Text
        using var textPaint = new SKPaint
        {
            Color = SKColors.DarkGray,
            TextSize = radius * 0.5f,
            IsAntialias = true,
            TextAlign = SKTextAlign.Center
        };
        canvas.DrawText($"{_animatedProgress:P0}", cx, cy + textPaint.TextSize / 3, textPaint);
    }
}
```

---

## Step 319: Heatmap

```csharp
// ============================================
// Heatmap Calendar
// ============================================

public class CalendarHeatmapView : SKCanvasView
{
    public static readonly BindableProperty DataProperty =
        BindableProperty.Create(nameof(Data), typeof(Dictionary<DateTime, int>), 
            typeof(CalendarHeatmapView),
            propertyChanged: (b, _, _) => ((CalendarHeatmapView)b).InvalidateSurface());
    
    public Dictionary<DateTime, int>? Data
    {
        get => (Dictionary<DateTime, int>?)GetValue(DataProperty);
        set => SetValue(DataProperty, value);
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        var info = e.Info;
        
        canvas.Clear(SKColors.White);
        
        if (Data == null) return;
        
        var cellSize = info.Width / 53f; // 52 weeks + 1
        var today = DateTime.Today;
        var startDate = today.AddDays(-(int)today.DayOfWeek - 7 * 52);
        
        var maxValue = Data.Values.DefaultIfEmpty(0).Max();
        
        using var cellPaint = new SKPaint { IsAntialias = true };
        using var emptyPaint = new SKPaint
        {
            Color = new SKColor(235, 237, 240),
            IsAntialias = true
        };
        
        for (var week = 0; week < 53; week++)
        {
            for (var day = 0; day < 7; day++)
            {
                var date = startDate.AddDays(week * 7 + day);
                if (date > today) continue;
                
                var x = week * cellSize;
                var y = day * cellSize;
                var rect = new SKRect(x + 1, y + 1, x + cellSize - 1, y + cellSize - 1);
                
                if (Data.TryGetValue(date.Date, out var value) && value > 0)
                {
                    var intensity = (float)value / maxValue;
                    cellPaint.Color = GetHeatColor(intensity);
                    canvas.DrawRoundRect(rect, 2, 2, cellPaint);
                }
                else
                {
                    canvas.DrawRoundRect(rect, 2, 2, emptyPaint);
                }
            }
        }
    }
    
    private static SKColor GetHeatColor(float intensity)
    {
        // GitHub-style green scale
        return intensity switch
        {
            < 0.25f => new SKColor(155, 233, 168),
            < 0.5f => new SKColor(64, 196, 99),
            < 0.75f => new SKColor(48, 161, 78),
            _ => new SKColor(33, 110, 57)
        };
    }
}
```

---

## Step 320: Performance Optimization for Graphics

```csharp
// ============================================
// SkiaSharp Performance Tips
// ============================================

public class OptimizedChartView : SKCanvasView
{
    // Cache paint objects - don't create in OnPaintSurface
    private readonly SKPaint _linePaint = new()
    {
        Color = SKColors.Blue,
        StrokeWidth = 2,
        Style = SKPaintStyle.Stroke,
        IsAntialias = true
    };
    
    private readonly SKPaint _fillPaint = new()
    {
        IsAntialias = true,
        Style = SKPaintStyle.Fill
    };
    
    // Cache path - only recreate when data changes
    private SKPath? _cachedPath;
    private List<ChartData>? _lastData;
    
    private SKPath GetOrCreatePath(List<ChartData> data, float width, float height)
    {
        if (_cachedPath != null && ReferenceEquals(_lastData, data))
            return _cachedPath;
        
        _cachedPath?.Dispose();
        _cachedPath = BuildPath(data, width, height);
        _lastData = data;
        return _cachedPath;
    }
    
    private static SKPath BuildPath(List<ChartData> data, float width, float height)
    {
        var path = new SKPath();
        // Build path...
        return path;
    }
    
    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        // Use Span for performance
        var info = e.Info;
        var canvas = e.Surface.Canvas;
        
        canvas.Clear(SKColors.White);
        
        // Use hardware acceleration hint
        canvas.Save();
        
        // Clip to bounds for performance
        canvas.ClipRect(new SKRect(0, 0, info.Width, info.Height));
        
        // Draw cached path
        // var path = GetOrCreatePath(...);
        // canvas.DrawPath(path, _linePaint);
        
        canvas.Restore();
    }
    
    protected override void OnHandlerChanging(HandlerChangingEventArgs args)
    {
        base.OnHandlerChanging(args);
        
        if (args.NewHandler == null)
        {
            // Cleanup - dispose SKPaint objects
            _linePaint.Dispose();
            _fillPaint.Dispose();
            _cachedPath?.Dispose();
        }
    }
}
```

---

## สรุป Part 32

ใน Part 32 เราได้เรียนรู้:

1. **SkiaSharp Basics** - Drawing shapes, lines, text
2. **Gradients & Effects** - Linear, radial gradients, blur
3. **Bar Chart** - Custom bar chart view
4. **Line Chart** - Smooth curves with area fill
5. **Pie/Donut Chart** - With legends
6. **Interactive Drawing** - Touch drawing canvas
7. **Sparklines** - Mini charts for lists
8. **Progress Ring** - Animated circular progress
9. **Heatmap Calendar** - GitHub-style contribution chart
10. **Performance** - Cached paints, paths

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 32 | Steps 311-320*
