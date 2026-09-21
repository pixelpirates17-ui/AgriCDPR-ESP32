# AgriCDPR-ESP32

/*
============================================================
                    AgriCDPR
       Smart Agriculture Monitoring System
============================================================

Sensors:
- BH1750 Light Sensor
- MQ Gas Sensor
- MQ-4 Methane Sensor
- HC-SR04 x 2
- OLED Display
- SIM800L

Communication:
- WiFi
- Firebase Realtime Database
- SIM800L SMS

Gemini:
- Will be connected through Firebase Cloud Functions
============================================================
*/


// ============================================================
// LIBRARIES
// ============================================================

#include <WiFi.h>
#include <HTTPClient.h>

#include <Wire.h>
#include <BH1750.h>

#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>


// ============================================================
// WIFI
// ============================================================

const char* WIFI_SSID = "YOUR_WIFI_NAME";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";


// ============================================================
// FIREBASE
// ============================================================

// Firebase Realtime Database URL
// Example:
// https://your-project-default-rtdb.firebaseio.com

const char* FIREBASE_URL =
  "https://YOUR_PROJECT_ID-default-rtdb.firebaseio.com";


// ============================================================
// OLED
// ============================================================

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define OLED_RESET -1
#define OLED_ADDRESS 0x3C

Adafruit_SSD1306 display(
  SCREEN_WIDTH,
  SCREEN_HEIGHT,
  &Wire,
  OLED_RESET
);


// ============================================================
// BH1750
// ============================================================

BH1750 lightMeter;


// ============================================================
// GAS SENSORS
// ============================================================

#define GAS_PIN 34
#define METHANE_PIN 35


// ============================================================
// ULTRASONIC SENSOR 1
// ============================================================

#define TRIG1 5
#define ECHO1 18


// ============================================================
// ULTRASONIC SENSOR 2
// ============================================================

#define TRIG2 25
#define ECHO2 19


// ============================================================
// SIM800L
// ============================================================

#define GSM_RX 16
#define GSM_TX 17

HardwareSerial gsm(2);


// ============================================================
// ALERT THRESHOLDS
// ============================================================

#define GAS_THRESHOLD 2500
#define METHANE_THRESHOLD 2500


// ============================================================
// PHONE NUMBER
// ============================================================

String phoneNumber = "+91XXXXXXXXXX";


// ============================================================
// ALERT FLAGS
// ============================================================

bool gasAlertSent = false;
bool methaneAlertSent = false;


// ============================================================
// SENSOR VARIABLES
// ============================================================

float lightValue = 0;

int gasValue = 0;

int methaneValue = 0;

float distance1 = 0;

float distance2 = 0;


// ============================================================
// TIMER
// ============================================================

unsigned long lastFirebaseUpload = 0;

const unsigned long firebaseInterval = 10000;


// ============================================================
// CONNECT WIFI
// ============================================================

void connectWiFi()
{
  Serial.println();
  Serial.println("Connecting to WiFi...");

  WiFi.begin(
    WIFI_SSID,
    WIFI_PASSWORD
  );

  while (
    WiFi.status() != WL_CONNECTED
  )
  {
    delay(500);

    Serial.print(".");
  }

  Serial.println();

  Serial.println(
    "WiFi connected!"
  );

  Serial.print(
    "IP Address: "
  );

  Serial.println(
    WiFi.localIP()
  );
}


// ============================================================
// ULTRASONIC FUNCTION
// ============================================================

float getDistance(
  int trigPin,
  int echoPin
)
{
  digitalWrite(
    trigPin,
    LOW
  );

  delayMicroseconds(2);

  digitalWrite(
    trigPin,
    HIGH
  );

  delayMicroseconds(10);

  digitalWrite(
    trigPin,
    LOW
  );


  long duration = pulseIn(
    echoPin,
    HIGH,
    30000
  );


  if (duration == 0)
  {
    return -1;
  }


  float distance =
    duration * 0.0343 / 2;


  return distance;
}


// ============================================================
// READ SENSORS
// ============================================================

void readSensors()
{
  // BH1750

  lightValue =
    lightMeter.readLightLevel();


  // Gas sensor

  gasValue =
    analogRead(GAS_PIN);


  // Methane

  methaneValue =
    analogRead(METHANE_PIN);


  // Ultrasonic 1

  distance1 =
    getDistance(
      TRIG1,
      ECHO1
    );


  delay(50);


  // Ultrasonic 2

  distance2 =
    getDistance(
      TRIG2,
      ECHO2
    );
}


// ============================================================
// OLED DISPLAY
// ============================================================

void updateOLED()
{
  display.clearDisplay();

  display.setTextColor(
    SSD1306_WHITE
  );

  display.setTextSize(1);


  // Light

  display.setCursor(
    0,
    0
  );

  display.print(
    "Light: "
  );

  display.print(
    lightValue,
    0
  );

  display.println(
    " lx"
  );


  // Gas

  display.setCursor(
    0,
    12
  );

  display.print(
    "Gas: "
  );

  display.println(
    gasValue
  );


  // Methane

  display.setCursor(
    0,
    24
  );

  display.print(
    "CH4: "
  );

  display.println(
    methaneValue
  );


  // Distance 1

  display.setCursor(
    0,
    36
  );

  display.print(
    "D1: "
  );


  if (distance1 < 0)
  {
    display.println(
      "ERROR"
    );
  }
  else
  {
    display.print(
      distance1,
      1
    );

    display.println(
      " cm"
    );
  }


  // Distance 2

  display.setCursor(
    0,
    48
  );

  display.print(
    "D2: "
  );


  if (distance2 < 0)
  {
    display.println(
      "ERROR"
    );
  }
  else
  {
    display.print(
      distance2,
      1
    );

    display.println(
      " cm"
    );
  }


  display.display();
}


// ============================================================
// SERIAL MONITOR
// ============================================================

void printSensorData()
{
  Serial.println();
  Serial.println(
    "========== AGRICDPR =========="
  );


  Serial.print(
    "Light: "
  );

  Serial.print(
    lightValue
  );

  Serial.println(
    " lux"
  );


  Serial.print(
    "Gas: "
  );

  Serial.println(
    gasValue
  );


  Serial.print(
    "Methane: "
  );

  Serial.println(
    methaneValue
  );


  Serial.print(
    "Distance 1: "
  );

  Serial.print(
    distance1
  );

  Serial.println(
    " cm"
  );


  Serial.print(
    "Distance 2: "
  );

  Serial.print(
    distance2
  );

  Serial.println(
    " cm"
  );


  Serial.print(
    "WiFi: "
  );

  if (
    WiFi.status() == WL_CONNECTED
  )
  {
    Serial.println(
      "Connected"
    );
  }
  else
  {
    Serial.println(
      "Disconnected"
    );
  }


  Serial.println(
    "=============================="
  );
}


// ============================================================
// SEND SMS
// ============================================================

void sendSMS(
  String message
)
{
  Serial.println(
    "Sending SMS..."
  );


  gsm.println(
    "AT+CMGF=1"
  );

  delay(1000);


  gsm.print(
    "AT+CMGS=\""
  );

  gsm.print(
    phoneNumber
  );

  gsm.println(
    "\""
  );

  delay(1000);


  gsm.print(
    message
  );

  delay(500);


  gsm.write(26);

  delay(5000);


  Serial.println(
    "SMS sent."
  );
}


// ============================================================
// GAS ALERT
// ============================================================

void checkGasAlert()
{
  if (
    gasValue >
    GAS_THRESHOLD
  )
  {
    if (!gasAlertSent)
    {
      sendSMS(
        "ALERT: High gas level detected by AgriCDPR."
      );

      gasAlertSent = true;
    }
  }
  else
  {
    gasAlertSent = false;
  }
}


// ============================================================
// METHANE ALERT
// ============================================================

void checkMethaneAlert()
{
  if (
    methaneValue >
    METHANE_THRESHOLD
  )
  {
    if (!methaneAlertSent)
    {
      sendSMS(
        "ALERT: High methane level detected by AgriCDPR."
      );

      methaneAlertSent = true;
    }
  }
  else
  {
    methaneAlertSent = false;
  }
}


// ============================================================
// SEND DATA TO FIREBASE
// ============================================================

void sendToFirebase()
{
  if (
    WiFi.status() != WL_CONNECTED
  )
  {
    Serial.println(
      "WiFi disconnected."
    );

    return;
  }


  HTTPClient http;


  // ----------------------------------------------------------
  // Firebase URL
  // ----------------------------------------------------------

  String url =
    String(FIREBASE_URL) +
    "/AgriCDPR/sensors.json";


  // ----------------------------------------------------------
  // JSON DATA
  // ----------------------------------------------------------

  String jsonData = "{";

  jsonData += "\"light\":";
  jsonData += String(
    lightValue,
    2
  );

  jsonData += ",";

  jsonData += "\"gas\":";
  jsonData += String(
    gasValue
  );

  jsonData += ",";

  jsonData += "\"methane\":";
  jsonData += String(
    methaneValue
  );

  jsonData += ",";

  jsonData += "\"distance1\":";
  jsonData += String(
    distance1,
    2
  );

  jsonData += ",";

  jsonData += "\"distance2\":";
  jsonData += String(
    distance2,
    2
  );

  jsonData += "}";


  // ----------------------------------------------------------
  // HTTP PUT
  // ----------------------------------------------------------

  http.begin(
    url
  );


  http.addHeader(
    "Content-Type",
    "application/json"
  );


  int httpResponseCode =
    http.PUT(
      jsonData
    );


  Serial.print(
    "Firebase HTTP code: "
  );

  Serial.println(
    httpResponseCode
  );


  if (
    httpResponseCode > 0
  )
  {
    Serial.println(
      "Firebase data uploaded!"
    );

    Serial.println(
      jsonData
    );
  }
  else
  {
    Serial.println(
      "Firebase upload failed."
    );
  }


  http.end();
}


// ============================================================
// SETUP
// ============================================================

void setup()
{
  Serial.begin(
    115200
  );


  // ==========================================================
  // I2C
  // ==========================================================

  Wire.begin(
    21,
    22
  );


  // ==========================================================
  // OLED
  // ==========================================================

  if (
    !display.begin(
      SSD1306_SWITCHCAPVCC,
      OLED_ADDRESS
    )
  )
  {
    Serial.println(
      "OLED not found!"
    );

    while (1);
  }


  display.clearDisplay();

  display.setTextColor(
    SSD1306_WHITE
  );

  display.setTextSize(2);

  display.setCursor(
    10,
    10
  );

  display.println(
    "AgriCDPR"
  );


  display.setTextSize(1);

  display.setCursor(
    20,
    38
  );

  display.println(
    "Starting..."
  );

  display.display();


  delay(2000);


  // ==========================================================
  // BH1750
  // ==========================================================

  if (
    lightMeter.begin()
  )
  {
    Serial.println(
      "BH1750 OK"
    );
  }
  else
  {
    Serial.println(
      "BH1750 ERROR"
    );
  }


  // ==========================================================
  // GAS
  // ==========================================================

  pinMode(
    GAS_PIN,
    INPUT
  );

  pinMode(
    METHANE_PIN,
    INPUT
  );


  // ==========================================================
  // ULTRASONIC
  // ==========================================================

  pinMode(
    TRIG1,
    OUTPUT
  );

  pinMode(
    ECHO1,
    INPUT
  );


  pinMode(
    TRIG2,
    OUTPUT
  );

  pinMode(
    ECHO2,
    INPUT
  );


  // ==========================================================
  // SIM800L
  // ==========================================================

  gsm.begin(
    9600,
    SERIAL_8N1,
    GSM_RX,
    GSM_TX
  );


  delay(3000);


  gsm.println(
    "AT"
  );


  // ==========================================================
  // WIFI
  // ==========================================================

  connectWiFi();


  // ==========================================================
  // READY
  // ==========================================================

  Serial.println();

  Serial.println(
    "================================"
  );

  Serial.println(
    "      AGRICDPR READY"
  );

  Serial.println(
    "================================"
  );
}


// ============================================================
// LOOP
// ============================================================

void loop()
{
  // Read sensors

  readSensors();


  // Display

  updateOLED();


  // Serial

  printSensorData();


  // SMS alerts

  checkGasAlert();

  checkMethaneAlert();


  // Firebase

  if (
    millis() -
    lastFirebaseUpload
    >=
    firebaseInterval
  )
  {
    lastFirebaseUpload =
      millis();

    sendToFirebase();
  }


  delay(1000);
}
