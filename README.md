#include <WiFi.h>
#include <DHT.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <HTTPClient.h>

// Wi-Fi credentials
const char* ssid = "OPPO A5 2020";       // your Wi-Fi name
const char* password = "06220622"; // your Wi-Fi password

// ThingSpeak API Key (Write Key)
const String apiKey = "K9RVZ5G1XW7YTBKZ"; // Your ThingSpeak Write API Key

// DHT11 sensor setup
#define DHTPIN 27
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// DS18B20 sensor setup
#define ONE_WIRE_BUS 2  // GPIO2
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// Optional: You can skip fixed address if only one DS18B20 is connected
DeviceAddress sensor1 = { 0x28, 0xBE, 0x74, 0x46, 0xD4, 0xC9, 0x52, 0x77 };

void sendSensorData() {
  // DHT11 readings
  float humidity = dht.readHumidity();
  float temperatureDHT = dht.readTemperature();

  if (isnan(humidity) || isnan(temperatureDHT)) {
    Serial.println("Failed to read from DHT sensor!");
    temperatureDHT = -999;
    humidity = -999;
  }

  // DS18B20 readings
  sensors.requestTemperatures();
  float temperatureDS18B20 = sensors.getTempC(sensor1);

  if (temperatureDS18B20 == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: Could not read DS18B20 temperature.");
    temperatureDS18B20 = -999;
  }

  // Print readings
  Serial.print("DHT11 Temp: ");
  Serial.print(temperatureDHT);
  Serial.print(" °C, Humidity: ");
  Serial.println(humidity);

  Serial.print("DS18B20 Temp: ");
  Serial.print(temperatureDS18B20);
  Serial.println(" °C");

  // Prepare ThingSpeak URL
  String url = String("http://api.thingspeak.com/update?api_key=") + apiKey +
               "&field1=" + String(temperatureDHT) +
               "&field2=" + String(humidity) +
               "&field3=" + String(temperatureDS18B20);

  // Send to ThingSpeak
  HTTPClient http;
  http.begin(url);
  int httpCode = http.GET();

  if (httpCode > 0) {
    Serial.print("HTTP Response code: ");
    Serial.println(httpCode);
  } else {
    Serial.print("Error on HTTP request: ");
    Serial.println(httpCode);
  }

  http.end();
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println("Connecting to Wi-Fi...");
  WiFi.begin(ssid, password);

  unsigned long startAttemptTime = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - startAttemptTime < 10000) {
    delay(500);
    Serial.print(".");
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWi-Fi connected");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("\nWi-Fi connection failed.");
  }

  dht.begin();
  sensors.begin();
}

void loop() {
  sendSensorData();
  delay(15000); // ThingSpeak minimum update interval = 15 sec
}
