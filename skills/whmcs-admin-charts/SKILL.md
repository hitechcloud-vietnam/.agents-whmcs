# WHMCS Admin Charts

## Overview
Guide for implementing charts and data visualization in WHMCS admin area. Covers chart types, data preparation, and integration.

## Chart Implementation

### Chart Assets

```php
<?php
// /includes/hooks/admin_charts.php

add_hook("AdminChartAssets", 1, function(array $params) {
    return [
        "js" => [
            "/assets/js/Chart.min.js"
        ],
        "css" => []
    ];
});
```

### Chart Data Preparation

```php
function prepareChartData(string $chartType, array $config): array
{
    switch ($chartType) {
        case "line":
            return prepareLineChartData($config);
        case "bar":
            return prepareBarChartData($config);
        case "pie":
            return preparePieChartData($config);
        case "doughnut":
            return prepareDoughnutChartData($config);
        default:
            return [];
    }
}

function prepareLineChartData(array $config): array
{
    $labels = [];
    $datasets = [];
    
    $data = getChartData($config["query"], $config["date_range"]);
    
    foreach ($config["datasets"] as $index => $dataset) {
        $datasets[] = [
            "label" => $dataset["label"],
            "data" => array_column($data, $dataset["field"]),
            "borderColor" => $dataset["color"] ?? generateColor($index),
            "backgroundColor" => $dataset["fill_color"] ?? "transparent",
            "fill" => $dataset["fill"] ?? false,
            "tension" => $dataset["tension"] ?? 0.4
        ];
    }
    
    $labels = array_column($data, $config["label_field"]);
    
    return [
        "type" => "line",
        "data" => [
            "labels" => $labels,
            "datasets" => $datasets
        ],
        "options" => getChartOptions($config)
    ];
}

function getChartData(string $query, array $dateRange): array
{
    $data = Capsule::select($query, [
        $dateRange["start"],
        $dateRange["end"]
    ]);
    
    return $data;
}

function getChartOptions(array $config): array
{
    return [
        "responsive" => true,
        "maintainAspectRatio" => false,
        "plugins" => [
            "legend" => [
                "display" => $config["show_legend"] ?? true,
                "position" => $config["legend_position"] ?? "top"
            ],
            "tooltip" => [
                "enabled" => true,
                "mode" => "index",
                "intersect" => false
            ]
        ],
        "scales" => [
            "y" => [
                "beginAtZero" => true,
                "grid" => [
                    "display" => true
                ]
            ],
            "x" => [
                "grid" => [
                    "display" => false
                ]
            ]
        ]
    ];
}
```

### Revenue Chart

```php
add_hook("AdminDashboardCharts", 1, function(array $params) {
    $months = 12;
    $labels = [];
    $revenueData = [];
    
    for ($i = $months - 1; $i >= 0; $i--) {
        $month = date("Y-m", strtotime("-{$i} months"));
        $labels[] = date("M Y", strtotime("-{$i} months"));
        
        $revenue = Capsule::table("tblaccounts")
            ->where("date", "like", $month . "%")
            ->sum("amountin") ?? 0;
        
        $revenueData[] = round($revenue, 2);
    }
    
    return [
        "revenue_chart" => [
            "type" => "line",
            "title" => "Monthly Revenue",
            "data" => [
                "labels" => $labels,
                "datasets" => [[
                    "label" => "Revenue",
                    "data" => $revenueData,
                    "borderColor" => "#4CAF50",
                    "backgroundColor" => "rgba(76, 175, 80, 0.1)",
                    "fill" => true,
                    "tension" => 0.4
                ]]
            ]
        ]
    ];
});
```

### Chart Template

```smarty
<!-- /admin/templates/chart_container.tpl -->
<div class="chart-container" id="{$chart.id}">
    <div class="chart-header">
        <h4>{$chart.title}</h4>
        {if $chart.period_selector}
            <div class="period-selector">
                <select name="period" class="form-control input-sm">
                    <option value="7d">Last 7 days</option>
                    <option value="30d">Last 30 days</option>
                    <option value="90d" selected>Last 90 days</option>
                    <option value="1y">Last year</option>
                </select>
            </div>
        {/if}
    </div>
    <div class="chart-body">
        <canvas id="chart-canvas-{$chart.id}"></canvas>
    </div>
</div>

<script>
(function() {
    var ctx = document.getElementById('chart-canvas-{$chart.id}').getContext('2d');
    var chartData = {$chart.data|json_encode};
    
    // Merge with default options
    chartData.options = Object.assign({
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
            legend: {
                display: true
            }
        }
    }, chartData.options);
    
    var myChart = new Chart(ctx, chartData);
    
    // Period selector
    $('.period-selector select').on('change', function() {
        var period = $(this).val();
        $.post('ajax.php?action=update_chart', {
            chart_id: '{$chart.id}',
            period: period,
            token: '{$token}'
        }, function(response) {
            myChart.data = response.data;
            myChart.update();
        });
    });
})();
</script>

<style>
.chart-container {
    background: #fff;
    border-radius: 4px;
    padding: 15px;
    margin-bottom: 20px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}
.chart-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
}
.chart-header h4 {
    margin: 0;
}
.chart-body {
    height: 300px;
    position: relative;
}
</style>
```

### Multiple Chart Types

```php
function preparePieChartData(array $config): array
{
    $data = Capsule::table($config["table"])
        ->select($config["group_by"], Capsule::raw("COUNT(*) as count"))
        ->groupBy($config["group_by"])
        ->get();
    
    $labels = [];
    $values = [];
    $colors = [];
    
    foreach ($data as $row) {
        $labels[] = $row->{$config["group_by"]};
        $values[] = $row->count;
        $colors[] = generateColor(count($labels) - 1);
    }
    
    return [
        "type" => "pie",
        "data" => [
            "labels" => $labels,
            "datasets" => [[
                "data" => $values,
                "backgroundColor" => $colors,
                "borderWidth" => 2,
                "borderColor" => "#fff"
            ]]
        ],
        "options" => [
            "responsive" => true,
            "maintainAspectRatio" => false,
            "plugins" => [
                "legend" => [
                    "position" => "right"
                ]
            ]
        ]
    ];
}
```

## Best Practices

1. **Responsive**: Use responsive chart containers
2. **Performance**: Limit data points for large datasets
3. **Colors**: Use consistent, accessible colors
4. **Labels**: Clear, descriptive axis labels
5. **Legend**: Show/hide based on chart complexity
6. **Tooltips**: Enable informative tooltips
7. **Loading**: Show loading state during data fetch
8. **Export**: Support chart export to image
