---
title: "Stats"
date: 2026-09-21
---

Total page views on this site over time.

<canvas id="viewsChart"></canvas>
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js"></script>
<script>
fetch('/data/views.txt')
  .then(r => r.text())
  .then(text => {
    const lines = text.trim().split('\n').filter(Boolean);

    const dataMap = {};
    lines.forEach(line => {
      const [ymd, count] = line.split(' ');
      const dateStr = `20${ymd.slice(0,2)}-${ymd.slice(2,4)}-${ymd.slice(4,6)}`;
      dataMap[dateStr] = parseInt(count, 10);
    });

    const sortedDates = Object.keys(dataMap).sort();
    if (sortedDates.length === 0) {
      document.getElementById('viewsChart').outerHTML = '<p>No view data yet — check back soon.</p>';
      return;
    }

    const firstDate = new Date(sortedDates[0]);
    const lastDate = new Date(sortedDates[sortedDates.length - 1]);

    const labels = [];
    const counts = [];
    for (let d = new Date(firstDate); d <= lastDate; d.setDate(d.getDate() + 1)) {
      const dateStr = d.toISOString().slice(0, 10);
      labels.push(dateStr);
      counts.push(dataMap[dateStr] || 0);
    }

    const styles = getComputedStyle(document.body);
    const textMuted = styles.getPropertyValue('--text-muted').trim();
    const textColor = styles.getPropertyValue('--text').trim();
    const linkColor = styles.getPropertyValue('--link').trim();
    const gridColor = styles.getPropertyValue('--border').trim() + '33';
    const surfaceColor = styles.getPropertyValue('--surface').trim();
    const borderColor = styles.getPropertyValue('--border').trim();

    new Chart(document.getElementById('viewsChart'), {
      type: 'line',
      data: {
        labels: labels,
        datasets: [{
          label: 'Views',
          data: counts,
          borderColor: linkColor,
          tension: 0.2,
          fill: false,
          pointRadius: 2,
          pointHoverRadius: 5,
          pointHitRadius: 15
        }]
      },
      options: {
        interaction: {
          mode: 'index',
          intersect: false
        },
        scales: {
          x: {
            ticks: {
              color: textMuted,
              autoSkip: false,
              maxRotation: 0,
              callback: function(value, index) {
                const label = labels[index];
                return label && label.endsWith('-01') ? label.slice(0, 7) : '';
              }
            },
            grid: { color: gridColor },
            border: { color: borderColor }
          },
          y: {
            ticks: { color: textMuted },
            grid: { color: gridColor },
            border: { color: borderColor },
            beginAtZero: true
          }
        },
        plugins: {
          legend: { display: false },
          tooltip: {
            backgroundColor: surfaceColor,
            titleColor: textColor,
            bodyColor: textColor,
            borderColor: borderColor,
            borderWidth: 1
          }
        }
      }
    });
  })
  .catch(() => {
    document.getElementById('viewsChart').outerHTML = '<p>Couldn\'t load view data.</p>';
  });
</script>
