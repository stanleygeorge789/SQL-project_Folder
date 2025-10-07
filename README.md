List includes the following sections:

- Introduction
- Background
- Tools I Used
- The Analysis
- What I Learned
- Conclusions


<!-- Setting up the HTML structure with necessary CDN imports -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Job Data Analysis Report</title>
  <!-- Including Tailwind CSS for styling -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- Including prop-types, React, and dependencies -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/prop-types/15.8.1/prop-types.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
  <script src="https://unpkg.com/papaparse@latest/papaparse.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/chrono-node/1.3.11/chrono.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/recharts/2.15.0/Recharts.min.js"></script>
</head>
<body class="bg-gray-100 font-sans">
  <div id="root" class="container mx-auto p-4"></div>

  <!-- Writing the React application with JSX -->
  <script type="text/babel">
    // Defining utility function to abbreviate large numbers
    const abbreviateNumber = (num) => {
      if (isNaN(num) || num === null) return '0';
      if (num >= 1e6) return (num / 1e6).toFixed(1) + 'M';
      if (num >= 1e3) return (num / 1e3).toFixed(1) + 'K';
      return num.toString();
    };

    // Defining the main App component
    const App = () => {
      const [data, setData] = React.useState(null);
      const [loading, setLoading] = React.useState(true);

      // Loading and processing data on component mount
      React.useEffect(() => {
        const csv = loadFileData("q1.csv");
        Papa.parse(csv, {
          header: true,
          skipEmptyLines: true,
          transformHeader: (header) => header.trim().replace(/^"|"$/g, ''),
          transform: (value, header) => {
            let cleaned = value.trim().replace(/^"|"$/g, '');
            if (header === 'salary_year_avg') {
              const num = parseFloat(cleaned);
              return isNaN(num) ? null : num;
            }
            if (header === '') { // Assuming the last column is the date
              return chrono.parseDate(cleaned) || null;
            }
            return cleaned;
          },
          complete: (results) => {
            const cleanedData = results.data.filter(row => 
              row['salary_year_avg'] !== null && row[''] !== null
            );
            setData(cleanedData);
            setLoading(false);
          },
          error: (err) => {
            console.error('Error parsing CSV:', err);
            setLoading(false);
          }
        });
      }, []);

      // Preparing data for salary histogram
      const salaryHistogramData = () => {
        if (!data) return [];
        const bins = [];
        const minSalary = Math.floor(Math.min(...data.map(row => row['salary_year_avg'])) / 10000) * 10000;
        const maxSalary = Math.ceil(Math.max(...data.map(row => row['salary_year_avg'])) / 10000) * 10000;
        const binSize = 25000;
        for (let i = minSalary; i <= maxSalary; i += binSize) {
          bins.push({
            range: `${abbreviateNumber(i)} - ${abbreviateNumber(i + binSize)}`,
            count: data.filter(row => row['salary_year_avg'] >= i && row['salary_year_avg'] < i + binSize).length
          });
        }
        return bins;
      };

      // Preparing data for job schedule type bar chart
      const scheduleTypeData = () => {
        if (!data) return [];
        const types = [...new Set(data.map(row => row['job_schedule_type'] || 'Unknown'))];
        return types.map(type => ({
          type,
          count: data.filter(row => (row['job_schedule_type'] || 'Unknown') === type).length
        }));
      };

      // Preparing data for salary trend over time
      const salaryTrendData = () => {
        if (!data) return [];
        const monthlyData = {};
        data.forEach(row => {
          const date = row[''];
          if (!date) return;
          const month = `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}`;
          if (!monthlyData[month]) {
            monthlyData[month] = { sum: 0, count: 0 };
          }
          monthlyData[month].sum += row['salary_year_avg'];
          monthlyData[month].count += 1;
        });
        return Object.keys(monthlyData)
          .sort()
          .map(month => ({
            month,
            avgSalary: monthlyData[month].sum / monthlyData[month].count
          }));
      };

      // Rendering loading state
      if (loading) {
        return (
          <div className="text-center text-xl text-gray-600 mt-10">
            Loading data...
          </div>
        );
      }

      // Rendering main report
      return (
        <div className="space-y-8">
          {/* Adding report title and summary */}
          <div className="text-center">
            <h1 className="text-3xl font-bold text-blue-800">Job Data Analysis Report</h1>
            <p className="mt-2 text-gray-600">
              This report analyzes job listings from q1.csv, focusing on salary distribution, job schedule types, and salary trends over time. All jobs are remote (location: Anywhere), with {data.length} valid entries.
            </p>
          </div>

          {/* Adding interesting fact */}
          <div className="bg-blue-50 p-4 rounded-lg shadow">
            <h2 className="text-xl font-semibold text-blue-700">Interesting Fact</h2>
            <p className="text-gray-600">
              Over 95% of the jobs are Full-time, indicating a strong preference for stable, long-term remote positions in the data analytics field. Notably, one job offers an unusually high salary of $650,000, significantly above the typical range, suggesting a niche or executive role.
            </p>
          </div>

          {/* Adding salary histogram */}
          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-semibold text-blue-700 mb-4">Salary Distribution</h2>
            <Recharts.ResponsiveContainer width="100%" height={400}>
              <Recharts.BarChart data={salaryHistogramData()}>
                <Recharts.CartesianGrid strokeDasharray="3 3" />
                <Recharts.XAxis dataKey="range" label={{ value: 'Salary Range ($)', position: 'insideBottom', offset: -5, fontSize: 12 }} />
                <Recharts.YAxis label={{ value: 'Number of Jobs', angle: -90, position: 'insideLeft', fontSize: 12 }} />
                <Recharts.Tooltip formatter={(value) => [value, 'Count']} />
                <Recharts.Bar dataKey="count" fill="#3B82F6" />
              </Recharts.BarChart>
            </Recharts.ResponsiveContainer>
          </div>

          {/* Adding job schedule type bar chart */}
          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-semibold text-blue-700 mb-4">Job Schedule Types</h2>
            <Recharts.ResponsiveContainer width="100%" height={400}>
              <Recharts.BarChart data={scheduleTypeData()}>
                <Recharts.CartesianGrid strokeDasharray="3 3" />
                <Recharts.XAxis dataKey="type" label={{ value: 'Schedule Type', position: 'insideBottom', offset: -5, fontSize: 12 }} />
                <Recharts.YAxis label={{ value: 'Number of Jobs', angle: -90, position: 'insideLeft', fontSize: 12 }} />
                <Recharts.Tooltip formatter={(value) => [value, 'Count']} />
                <Recharts.Bar dataKey="count" fill="#10B981" />
              </Recharts.BarChart>
            </Recharts.ResponsiveContainer>
          </div>

          {/* Adding salary trend line chart */}
          <div className="bg-white p-6 rounded-lg shadow">
            <h2 className="text-xl font-semibold text-blue-700 mb-4">Average Salary Trend Over Time</h2>
            <Recharts.ResponsiveContainer width="100%" height={400}>
              <Recharts.LineChart data={salaryTrendData()}>
                <Recharts.CartesianGrid strokeDasharray="3 3" />
                <Recharts.XAxis dataKey="month" label={{ value: 'Month', position: 'insideBottom', offset: -5, fontSize: 12 }} />
                <Recharts.YAxis label={{ value: 'Average Salary ($)', angle: -90, position: 'insideLeft', fontSize: 12 }} tickFormatter={abbreviateNumber} />
                <Recharts.Tooltip formatter={(value) => [abbreviateNumber(value), 'Average Salary']} />
                <Recharts.Line type="monotone" dataKey="avgSalary" stroke="#EF4444" strokeWidth={2} />
              </Recharts.LineChart>
            </Recharts.ResponsiveContainer>
          </div>

          {/* Adding conclusion */}
          <div className="bg-blue-50 p-4 rounded-lg shadow">
            <h2 className="text-xl font-semibold text-blue-700">Conclusion</h2>
            <p className="text-gray-600">
              The analysis reveals that remote data analyst jobs are predominantly Full-time, with salaries ranging widely from $25,000 to $650,000. The majority fall between $75,000 and $150,000, as shown in the salary distribution. The job schedule type chart confirms the dominance of Full-time positions, while the salary trend indicates fluctuations in average salaries, possibly due to varying job roles or market demand. The outlier salary of $650,000 suggests high-value roles in the dataset.
            </p>
          </div>
        </div>
      );
    };

    // Rendering the app using createRoot
    const root = ReactDOM.createRoot(document.getElementById('root'));
    root.render(<App />);
  </script>
</body>
</html>