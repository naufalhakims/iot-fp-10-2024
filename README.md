# iot-fp-10-2024

#include <Wire.h>
#include <WiFi.h>
#include <NTPClient.h>
#include <WiFiUdp.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <ESPAsyncWebServer.h>
#include <ESP32Servo.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>

// ------------------ Configuration ------------------

// Replace with your WiFi credentials
const char* ssid = "a54-cuy";
const char* password = "akusayangibu";

// Define NTP Client to get time
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", 7 * 3600, 60000); // WIB offset in seconds, update interval in ms

// Initialize the OLED display with dimensions 128x64
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// Define buzzer pin
const int buzzer = 4;
unsigned long buzzerLastMillis = 0;
bool buzzerState = false;
const unsigned long buzzerInterval = 500;

// Define servo motor pin
Servo myServo;
const int servoPin = 5;

// Define motor control pin
const int motorPin = 18;
bool motorState = false;

// Alarm time variables
int alarmHour = -1;
int alarmMinute = -1;
bool alarmSet = false;

// ------------------ Telegram Bot Configuration ------------------

// Telegram BOT Token
#define BOTtoken "8194004479:AAHOIlMpmITS07T3LYLMOOO2yE2mgsAvaoo"  // Your Bot Token (Get from Botfather)

// WiFi Client and Telegram Bot
WiFiClientSecure client;
UniversalTelegramBot bot(BOTtoken, client);
int botRequestDelay = 1000;
unsigned long lastTimeBotRan;

// ------------------ Setup ------------------

void setup() {
  Serial.begin(115200);

  // Initialize OLED
  if (!display.begin(SSD1306_PAGEADDR, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  // Initialize buzzer
  pinMode(buzzer, OUTPUT);
  digitalWrite(buzzer, LOW);

  // Initialize servo motor
  myServo.attach(servoPin);
  myServo.write(90);

  // Initialize motor control
  pinMode(motorPin, OUTPUT);
  digitalWrite(motorPin, LOW);

  // Connect to WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");

  // Initialize NTP Client
  timeClient.begin();
  timeClient.update();

  // Initialize Telegram Bot
  client.setCACert(TELEGRAM_CERTIFICATE_ROOT);
}

// ------------------ Telegram Bot Handle ------------------

void handleNewMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);  // Get chat ID
    String text = bot.messages[i].text;               // Get the message text
    String from_name = bot.messages[i].from_name;     // Get sender's name

    // Display chat ID in Serial Monitor
    Serial.println("New message received!");
    Serial.println("Chat ID: " + chat_id);
    Serial.println("From: " + from_name);
    Serial.println("Message: " + text);

    // Respond to /start command
    if (text == "/start") {
      String welcome = "Welcome, " + from_name + ".\n";
      welcome += "Your Chat ID is: " + chat_id + "\n";
      welcome += "Use this ID to allow access to bot commands.\n\n";
      welcome += "Available commands:\n";
      welcome += "/setalarm hour minute - Set alarm time\n";
      welcome += "/clearalarm - Clear the alarm\n";
      welcome += "/servo angle - Set servo angle\n";
      welcome += "/motor on/off - Control the motor\n";
      bot.sendMessage(chat_id, welcome, "");
    }

    // Add more command handling below
    if (text.startsWith("/setalarm")) {
      // /setalarm hour minute
      int hour = text.substring(10, 12).toInt();
      int minute = text.substring(13, 15).toInt();
      if (hour >= 0 && hour < 24 && minute >= 0 && minute < 60) {
        alarmHour = hour;
        alarmMinute = minute;
        alarmSet = true;
        bot.sendMessage(chat_id, "Alarm set to " + String(hour) + ":" + String(minute), "");
      } else {
        bot.sendMessage(chat_id, "Invalid time format", "");
      }
    }

    if (text == "/clearalarm") {
      alarmSet = false;
      alarmHour = -1;
      alarmMinute = -1;
      bot.sendMessage(chat_id, "Alarm cleared", "");
    }

    if (text.startsWith("/servo")) {
      // /servo angle
      int angle = text.substring(7).toInt();
      if (angle >= 0 && angle <= 180) {
        myServo.write(angle);
        bot.sendMessage(chat_id, "Servo angle set to " + String(angle), "");
      } else {
        bot.sendMessage(chat_id, "Invalid angle. Must be between 0 and 180.", "");
      }
    }

    if (text.startsWith("/motor")) {
      // /motor on/off
      String state = text.substring(7);
      if (state == "on") {
        motorState = true;
        digitalWrite(motorPin, HIGH);
        bot.sendMessage(chat_id, "Motor turned ON", "");
      } else if (state == "off") {
        motorState = false;
        digitalWrite(motorPin, LOW);
        bot.sendMessage(chat_id, "Motor turned OFF", "");
      } else {
        bot.sendMessage(chat_id, "Invalid motor state. Use 'on' or 'off'.", "");
      }
    }
  }
}

void loop() {
  if (millis() > lastTimeBotRan + botRequestDelay) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }
    lastTimeBotRan = millis();
  }

  // Update the time
  timeClient.update();

  // Display current time on OLED
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("Current Time:");
  display.setTextSize(2);
  display.setCursor(0, 10);
  display.printf("%02d:%02d", timeClient.getHours(), timeClient.getMinutes());

  // Display alarm status on OLED
  display.setTextSize(1);
  display.setCursor(0, 40);
  if (alarmSet) {
    display.print("Alarm: ");
    display.printf("%02d:%02d", alarmHour, alarmMinute);
  } else {
    display.println("Alarm: Off");
  }
  display.display();

  // Check alarm
  if (alarmSet && timeClient.getHours() == alarmHour && timeClient.getMinutes() == alarmMinute) {
    unsigned long currentMillis = millis();
    if (currentMillis - buzzerLastMillis >= buzzerInterval) {
      buzzerLastMillis = currentMillis;
      buzzerState = !buzzerState;
      digitalWrite(buzzer, buzzerState ? HIGH : LOW);
    }
    digitalWrite(motorPin, HIGH);
    myServo.write(0);
  } else {
    digitalWrite(buzzer, LOW);
    digitalWrite(motorPin, motorState ? HIGH : LOW);
  }

  delay(1000);
}
