Objective:
Develop a real-time weather monitoring system that retrieves weather data from the OpenWeatherMap API at regular intervals, processes it, and provides insights via rollups and aggregates. This system should support alerting thresholds and visualizations.

Key Features:
Data Retrieval: Regularly fetches weather data for metro cities (Delhi, Mumbai, Chennai, Bangalore, Kolkata, Hyderabad).
Data Processing: Converts temperatures from Kelvin to Celsius and calculates daily aggregates.
Rollups and Aggregates: Daily summaries for average, max, min temperatures, and dominant weather conditions.
Alerting: User-configurable alerts based on temperature or weather conditions.
Visualizations: Graphical representation of daily summaries, historical trends, and triggered alerts.
Dependencies in Code Format:
Here is the list of dependencies needed for building this project:
# 1. Install Python dependencies
pip install requests pandas matplotlib sqlite3 python-dotenv

# 2. Database setup
# Use SQLite for data persistence
# SQLite comes pre-installed with Python

# 3. (Optional) For alerts via email, use smtplib (comes pre-installed with Python)
# You may also use Flask for a web-based interface (if needed)
pip install flask

# 4. API Key Setup
# Create a .env file to securely store your OpenWeatherMap API key and configuration values
echo "OPENWEATHER_API_KEY=your_api_key_here" > .env
Code Structure:
Below is the code structure for implementing the weather monitoring system in Jupyter Notebook:

Setup and Configuration:
import requests
import pandas as pd
import sqlite3
import matplotlib.pyplot as plt
from datetime import datetime
import os
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Configuration
API_KEY = os.getenv("OPENWEATHER_API_KEY")
API_URL = "http://api.openweathermap.org/data/2.5/weather"
LOCATIONS = ["Delhi", "Mumbai", "Chennai", "Bangalore", "Kolkata", "Hyderabad"]
INTERVAL = 300  # 5 minutes (can be configured)
Data Retrieval and Processing:
def fetch_weather_data(city):
    params = {
        'q': city,
        'appid': API_KEY
    }
    response = requests.get(API_URL, params=params)
    if response.status_code == 200:
        data = response.json()
        return {
            'city': city,
            'main': data['weather'][0]['main'],
            'temp': data['main']['temp'] - 273.15,  # Kelvin to Celsius
            'feels_like': data['main']['feels_like'] - 273.15,
            'dt': datetime.fromtimestamp(data['dt'])
        }
    return None
Storing Data in SQLite:
# Connect to SQLite database
conn = sqlite3.connect('weather_data.db')
cursor = conn.cursor()

# Create table to store weather data
cursor.execute('''
    CREATE TABLE IF NOT EXISTS weather (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        city TEXT,
        main TEXT,
        temp REAL,
        feels_like REAL,
        dt TEXT
    )
''')
conn.commit()

# Store fetched data
def store_weather_data(data):
    cursor.execute('''
        INSERT INTO weather (city, main, temp, feels_like, dt)
        VALUES (?, ?, ?, ?, ?)
    ''', (data['city'], data['main'], data['temp'], data['feels_like'], data['dt']))
    conn.commit()
Data Rollups and Aggregates:
def calculate_daily_summary():
    df = pd.read_sql_query("SELECT * FROM weather", conn)
    df['dt'] = pd.to_datetime(df['dt'])
    df['date'] = df['dt'].dt.date

    # Calculate daily aggregates
    daily_summary = df.groupby(['date', 'city']).agg(
        avg_temp=('temp', 'mean'),
        max_temp=('temp', 'max'),
        min_temp=('temp', 'min'),
        dominant_condition=('main', lambda x: x.value_counts().idxmax())
    ).reset_index()
    return daily_summary
Alerting Mechanism:
def check_alerts(threshold_temp=35):
    df = pd.read_sql_query("SELECT * FROM weather", conn)
    df['dt'] = pd.to_datetime(df['dt'])

    # Check if temperature exceeds the threshold for two consecutive entries
    for city in LOCATIONS:
        city_df = df[df['city'] == city].sort_values(by='dt')
        city_df['exceeded'] = city_df['temp'] > threshold_temp
        if city_df['exceeded'].rolling(2).sum().any():
            print(f"Alert! {city}: Temperature exceeded {threshold_temp}°C for two consecutive updates.")
Visualizations:
def plot_daily_summaries():
    summary = calculate_daily_summary()
    plt.figure(figsize=(12, 6))
    
    for city in LOCATIONS:
        city_summary = summary[summary['city'] == city]
        plt.plot(city_summary['date'], city_summary['avg_temp'], label=f"{city} Avg Temp")
    
    plt.title("Daily Average Temperatures")
    plt.xlabel("Date")
    plt.ylabel("Temperature (°C)")
    plt.legend()
    plt.show()
System Workflow:
import time

def run_weather_monitoring_system():
    while True:
        for city in LOCATIONS:
            data = fetch_weather_data(city)
            if data:
                store_weather_data(data)
        check_alerts(threshold_temp=35)
        plot_daily_summaries()
        time.sleep(INTERVAL)
This structured code will help you set up the real-time weather monitoring system with configurable alerting and visualizations.
