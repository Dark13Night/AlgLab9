# AlgLab9
№ Лабораторная работа №9
«Многопоточность, TPL, асинхронное программирование»

Цели работы:
Научиться работать со средствами многопоточности языка C#.
Научиться работать с паттерном асинхронного программирования платформы .Net языка C#.


№ Задание№1
Создайте многопоточное приложение для получения средних цен акций за год.
Используйте сайт https://finance.yahoo.com/ для получения дневных котировок списка акций из файла ticker.txt. Формат ссылки следующий:
https://query1.finance.yahoo.com/v7/finance/download/{Код_бумаги}?period1={Начальная_дата}&period2={Конечная_дата}&interval=1d&events=history&includeAdjustedClose=true
,где:
Код_бумаги – тикер из списка акций
Начальная_дата – метка времени начала запрашиваемого периода в UNIX формате (год назад).
Конечная_дата – метка времени конца запрашиваемого периода в UNIX формате (текущая дата).
Например, формат ссылки для AAPL:
https://query1.finance.yahoo.com/v7/finance/download/AAPL?period1=1629072000&period2=1660608000&interval=1d&events=history&includeAdjustedClose=true

По мере получения данных выполните запуск задачи(Task), которая будет считать среднюю цену акции за год (используйте среднее значение для каждого дня как (High+Low)/2. Сложите все полученные значения и поделите на число дней).
Результатом работы задачи будет являться среднее значение цены за год, которое необходимо вывести в файл в формате «Тикер:Цена». При этом обеспечьте потокобезопасный доступ к файлу между всеми задачами.
```
using System;
using System.Collections.Generic;
using System.IO;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        // Чтение списка акций из файла ticker.txt
        List<string> tickers = await ReadTickersFromFile("ticker.txt");

        // Запуск задач на получение и расчет средней цены для каждой акции
        List<Task> tasks = new List<Task>();
        foreach (string ticker in tickers)
        {
            tasks.Add(ProcessStockAsync(ticker));
        }

        // Ожидание завершения всех задач
        await Task.WhenAll(tasks);

        Console.WriteLine("Все задачи завершены.");
    }

    static async Task<List<string>> ReadTickersFromFile(string fileName)
    {
        List<string> tickers = new List<string>();

        using (StreamReader reader = new StreamReader(fileName))
        {
            string line;
            while ((line = await reader.ReadLineAsync()) != null)
            {
                tickers.Add(line.Trim());
            }
        }

        return tickers;
    }

    static async Task ProcessStockAsync(string ticker)
    {
        DateTime startDate = DateTime.Now.AddYears(-1);
        DateTime endDate = DateTime.Now;
        string url = $"https://query1.finance.yahoo.com/v7/finance/download/{ticker}?period1={ToUnixTimestamp(startDate)}&period2={ToUnixTimestamp(endDate)}&interval=1d&events=history&includeAdjustedClose=true";

        using (HttpClient client = new HttpClient())
        {
            try
            {
                string csvData = await client.GetStringAsync(url);

                double sum = 0;
                int count = 0;

                using (StringReader reader = new StringReader(csvData))
                {
                    // Пропустить заголовок CSV-файла
                    await reader.ReadLineAsync();

                    string line;
                    while ((line = await reader.ReadLineAsync()) != null)
                    {
                        string[] values = line.Split(',');

                        double high = double.Parse(values[2]);
                        double low = double.Parse(values[3]);

                        double average = (high + low) / 2;

                        sum += average;
                        count++;
                    }
                }

                double averagePrice = sum / count;
                string result = $"{ticker}:{averagePrice}";

                while (IsFileLocked("result.txt"))
                {
                    await Task.Delay(1000);
                }

                // Потокобезопасная запись результата в файл

                using (StreamWriter writer = File.AppendText("result.txt"))
                {
                    await writer.WriteLineAsync(result);
                }

                Console.WriteLine($"Задача для {ticker} выполнена.");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Ошибка при обработке акции {ticker}: {ex.Message}");
            }
        }
    }

    static bool IsFileLocked(string path)
    {
        try
        {
            using (FileStream stream = new FileStream(path, FileMode.Open, FileAccess.ReadWrite, FileShare.None))
            {
                return false;
            }
        }
        catch (IOException)
        {
            return true;
        }
    }

    static long ToUnixTimestamp(DateTime dateTime)
    {
        return (long)(dateTime.Subtract(new DateTime(1970, 1, 1))).TotalSeconds;
    }
}
```

№ Задание№2
На основе Лабораторной работы №6 создайте графическое приложение, получающее текущую погоду в разных городах мира.
Используйте WinForms или WPF.
Загрузите список городов с координатами из файла city.txt.
Добавьте в интерфейс приложения 2 элемента – один для отображения списка городов, второй элемент – кнопку, по нажатию на которую происходит загрузка текущей погоды в выбранном по API из Лабораторной работы №6.
Используйте ключевые слова async/await для загрузки данных и обновления интерфейса приложения таким образом, чтобы графический интерфейс не блокировался на время загрузки.
Если разработка ведется на *NIX или MACOS, допускается использовать консольное приложение, тем не менее следует использовать ключевые слова async/await при реализации методов.
```
using System.Text.Json;
using System.Text.RegularExpressions;
using System.Windows.Forms;

namespace WeatherApp
{
    public partial class AppWeather : Form
    {
        private const string apiKey = "78a42a81f9fcfc14fa64cee749e70b59";
        private const string apiUrl = "https://api.openweathermap.org/data/2.5/weather";

        public AppWeather()
        {
            InitializeComponent();
        }

        private async void Form1_Load(object sender, EventArgs e)
        {
            await LoadCityDataAsync();
        }

        private async Task LoadCityDataAsync()
        {
            try
            {
                var cities = await ReadCitiesFromFileAsync("city.txt");
                listBoxCities.Items.AddRange(cities.ToArray());
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error loading city data: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private async void buttonLoadWeather_Click(object sender, EventArgs e)
        {
            if (listBoxCities.SelectedItem == null)
            {
                MessageBox.Show("Please select a city", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }

            var selectedCity = listBoxCities.SelectedItem.ToString();
            var coordinates = selectedCity.Split(',');

            if (coordinates.Length != 3)
            {
                MessageBox.Show("Invalid city data", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }

            double latitude = Convert.ToDouble(coordinates[1], new System.Globalization.CultureInfo("en-US"));
            double longitude = Convert.ToDouble(coordinates[2], new System.Globalization.CultureInfo("en-US"));

            var weather = await GetWeatherDataAsync(latitude, longitude);
            if (weather != null)
            {
                DisplayWeatherInfo(weather);
            }
            else
            {
                labelWeatherInfo.Text = "Failed to fetch weather data";
            }

            Refresh();
        }

        private async Task<List<string>> ReadCitiesFromFileAsync(string filePath)
        {
            try
            {
                List<string> cities = new List<string>();

                using (StreamReader reader = new StreamReader(filePath))
                {
                    while (!reader.EndOfStream)
                    {
                        string line = await reader.ReadLineAsync();
                        string[] parts = line.Split('\t');

                        if (parts.Length == 2)
                        {
                            string cityName = parts[0].Trim();
                            string coordinates = parts[1].Trim();

                            if (IsValidCoordinates(coordinates))
                            {
                                cities.Add($"{cityName},{coordinates}");
                            }
                            else
                            {
                                MessageBox.Show($"Invalid coordinates format in line: {line}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                            }
                        }
                        else
                        {
                            MessageBox.Show($"Invalid line format: {line}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        }
                    }
                }

                return cities;
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error reading cities from file: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return new List<string>();
            }
        }

        private bool IsValidCoordinates(string coordinates)
        {
            if (Regex.IsMatch(coordinates, @"^\s*-?\d+(\.\d+)?,\s*-?\d+(\.\d+)?\s*$"))
            {
                return true;
            }
            return false;
        }

        private async Task<Weather> GetWeatherDataAsync(double latitude, double longitude)
        {
            using (HttpClient client = new HttpClient())
            {
                int maxAttempts = 10;
                int attempt = 0;

                while (attempt < maxAttempts)
                {
                    try
                    {
                        var response = await client.GetStringAsync($"{apiUrl}?lat={latitude}&lon={longitude}&appid={apiKey}&units=metric");
                        var weatherInfo = JsonSerializer.Deserialize<WeatherInfo>(response);

                        if (weatherInfo != null && !string.IsNullOrEmpty(weatherInfo.sys.country))
                        {
                            string country = weatherInfo.sys.country;
                            string name = weatherInfo.name;
                            double temp = weatherInfo.main.temp;
                            string description = weatherInfo.weather[0].description;

                            return new Weather
                            {
                                Country = country,
                                Name = name,
                                Temp = temp,
                                Description = description
                            };
                        }
                    }
                    catch (HttpRequestException ex)
                    {
                        MessageBox.Show($"HTTP request error: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        attempt++;
                    }
                    catch (JsonException ex)
                    {
                        MessageBox.Show($"JSON deserialization error: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        attempt++;
                    }
                    catch (Exception ex)
                    {
                        MessageBox.Show($"An error occurred: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        attempt++;
                    }
                }

                return null;
            }
        }

        private void DisplayWeatherInfo(Weather weather)
        {
            if (weather != null)
            {
                string weatherInfoText = $"Country: {weather.Country}\nCity: {weather.Name}\nTemperature: {weather.Temp}°C\nDescription: {weather.Description}";
                labelWeatherInfo.Text = weatherInfoText;
            }
            else
            {
                labelWeatherInfo.Text = "Weather data not available.";
            }
        }

    }

    public class Weather
    {
        public string Country { get; set; }
        public string Name { get; set; }
        public double Temp { get; set; }
        public string Description { get; set; }
    }

    public class WeatherInfo
    {
        public MainInfo main { get; set; }
        public WeatherDescription[] weather { get; set; }
        public string name { get; set; }
        public SysInfo sys { get; set; }
    }

    public class MainInfo
    {
        public double temp { get; set; }
    }

    public class WeatherDescription
    {
        public string description { get; set; }
    }

    public class SysInfo
    {
        public string country { get; set; }
    }

}
```
