html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, 
initial-scale=1.0">
    <title>Shelter KC Financial Tracking</title>

https://projects.propublica.org/nonprofits/organizations/431287029
    <!-- 
Load Chart.js from a public network -->
    <script src="https://jsdelivr.net"></script>
    <style>
        body { font-family: -apple-system, sans-serif; 
margin: 40px; background: #f9f9f9; }
        .container { max-width: 800px; margin: 0 auto; background: 
white; padding: 20px; border-radius: 
8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        h2 { color: #333; text-align: center; }
        p { color: #666; font-size: 14px; line-height: 1.5; }
    </style>
</head>
<body>

<div class="container">
    <h2>Shelter KC: 5-Year Financial Tracking</h2>
    <p>Data compiled directly from official 
IRS Form 990 filings via ProPublica. 
This chart maps total incoming revenue against 
total operating expenses from 2021 to 2025.</p>
    
    <canvas id="financialChart"></canvas>
</div>

<script>
const ctx = document.getElementById('financialChart')
getContext('2d');
const financialChart = new Chart(ctx, {
    type: 'bar',
    data: {
        labels: ['2021', '2022', '2023', '2024', '2025'],
        datasets: [
            {
                label: 'Total Revenue ($)',
                data:,
                backgroundColor: 'rgba(54, 162, 235, 0.7)',
                borderColor: 'rgba(54, 162, 235, 1)',
                borderWidth: 1
            },
            {
                label: 'Total Expenses ($)',
                data:,
                backgroundColor: 'rgba(255, 99, 132, 0.7)',
                borderColor: 'rgba(255, 99, 132, 1)',
                borderWidth: 1
            }
        ]
    },
    options: {
        responsive: true,
        scales: {
            y: {
                beginAtZero: true,
                title: { display: true, text: 'Amount in USD ($)' }
            }
        }
    }
});
</script>

</body>
</html>
