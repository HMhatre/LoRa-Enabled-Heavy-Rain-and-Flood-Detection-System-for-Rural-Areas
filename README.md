/*
========================================================
 PROJECT: LoRa-Based Heavy Rain and Flood Detection
 AUTHOR: Harsh Mahesh Mhatre
 CONTROLLER: Arduino Nano
 MODULE: SX1278 LoRa 433 MHz

 COMPONENTS:
 1. Arduino Nano - Main controller
 2. SX1278 - Wireless data transmission
 3. DHT11 - Temperature and humidity measurement
 4. Rain Sensor - Rainfall detection
 5. Water Level Sensor - Water-level monitoring
 6. HC-SR04 - Ultrasonic distance measurement
 7. Float Switch - High-water detection
 8. SSD1306 OLED - Sensor information display

 LIBRARIES:
 - LoRa by Sandeep Mistry
 - DHT Sensor Library
 - Adafruit Unified Sensor
 - Adafruit GFX
 - Adafruit SSD1306
========================================================
*/

#include <SPI.h>
#include <LoRa.h>
#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ---------------- DHT11 SENSOR ----------------
#define DHT_PIN 4
#define DHT_TYPE DHT11
DHT dht(DHT_PIN, DHT_TYPE);

// ---------------- SENSOR CONNECTIONS ----------------
#define RAIN_PIN A0
#define WATER_LEVEL_PIN A1
#define FLOAT_SWITCH_PIN 3

// ---------------- ULTRASONIC SENSOR ----------------
#define TRIG_PIN 5
#define ECHO_PIN 6

// ---------------- LORA MODULE ----------------
#define LORA_SS 10
#define LORA_RST 9
#define LORA_DIO0 2
#define LORA_FREQUENCY 433E6

// ---------------- OLED DISPLAY ----------------
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ---------------- THRESHOLDS ----------------
#define RAIN_THRESHOLD 500
#define WATER_THRESHOLD 500
#define HIGH_WATER_DISTANCE 20.0

// ---------------- TIMING ----------------
unsigned long previousMillis = 0;
const unsigned long interval = 2000;

// =================================================
// ULTRASONIC DISTANCE FUNCTION
// =================================================
float readDistance()
{
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  unsigned long duration = pulseIn(ECHO_PIN, HIGH, 30000UL);

  if (duration == 0)
    return -1;

  return duration * 0.0343 / 2;
}

// =================================================
// SETUP
// =================================================
void setup()
{
  Serial.begin(9600);
  dht.begin();

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(FLOAT_SWITCH_PIN, INPUT_PULLUP);

  // Initialize OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C))
  {
    Serial.println("OLED initialization failed");
  }

  // Initialize LoRa module
  LoRa.setPins(LORA_SS, LORA_RST, LORA_DIO0);

  if (!LoRa.begin(LORA_FREQUENCY))
  {
    Serial.println("LoRa initialization failed");
    while (1);
  }

  Serial.println("FLOOD TRANSMITTER INITIALIZED");
}

// =================================================
// MAIN LOOP
// =================================================
void loop()
{
  if (millis() - previousMillis < interval)
    return;

  previousMillis = millis();

  // Read rain sensor
  int rainValue = analogRead(RAIN_PIN);

  // Read water-level sensor
  int waterValue = analogRead(WATER_LEVEL_PIN);

  // Read temperature and humidity
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();

  if (isnan(temperature))
    temperature = -99;

  if (isnan(humidity))
    humidity = -1;

  // Measure water distance
  float distance = readDistance();

  // Read float switch
  int floatStatus = (digitalRead(FLOAT_SWITCH_PIN) == LOW) ? 1 : 0;

  // Determine environmental conditions
  bool rainDetected = (rainValue <= RAIN_THRESHOLD);
  bool waterHigh = (waterValue >= WATER_THRESHOLD);
  bool ultrasonicHigh = (distance > 0 && distance <= HIGH_WATER_DISTANCE);

  bool floodWarning = rainDetected && (waterHigh || ultrasonicHigh || floatStatus);

  // Create LoRa data packet
  String packet = "FLOOD1,";
  packet += String(rainValue) + ",";
  packet += String(waterValue) + ",";
  packet += String(temperature, 1) + ",";
  packet += String(humidity, 1) + ",";
  packet += String(distance, 1) + ",";
  packet += String(floatStatus);

  // Transmit sensor data
  LoRa.beginPacket();
  LoRa.print(packet);
  LoRa.endPacket();

  Serial.println("TRANSMITTED DATA:");
  Serial.println(packet);
  Serial.print("Flood Warning: ");
  Serial.println(floodWarning ? "YES" : "NO");

  // Update OLED display
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);

  display.println("FLOOD TRANSMITTER");
  display.print("Rain: "); display.println(rainValue);
  display.print("Water: "); display.println(waterValue);
  display.print("Temp: "); display.println(temperature);
  display.print("Humidity: "); display.println(humidity);
  display.print("Distance: "); display.println(distance);
  display.print("Float: "); display.println(floatStatus);

  display.display();
}
