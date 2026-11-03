# biovista-smart-device
 smart medical device prototype for real-time monitoring
📁 PRODUCT DOCUMENTATION PACKAGE: DIAGO SMART GLOVE
1. 📋 Executive Product Overview
The Diago Smart Glove is an advanced, wearable IoT biomedical and anthro-monitoring system designed for real-time health data acquisition and environmental diagnostics. Powered by an ESP32 Dev Module, the system features a multi-screen graphical user interface on an OLED display, dynamic feedback via a Grove LED bar, and interactive manual screen control via a debounced physical hardware interrupt.
2. 🔌 System Architecture & Full Hardware Interconnect Map
The system architecture has been explicitly optimized to run seamlessly without interfering with the ESP32 flashing constraints by utilizing the J3 expansion port for heavy-current peripherals.
Component	Pin Label	Wire Color	ESP32 GPIO / Rail	Description
Power Supply	Battery (+)	Red	Switch Pin 1	Master Power Toggle Line
	Switch Pin 2	-	TP4056 B+ Pad	Charge-protected Battery input
	Battery (-)	Black	TP4056 B- Pad	Shared Power Ground
OLED Display (SSD1306)	VCC / GND	-	3.3V / GND Rail	Standard I2C Logic Power
	SDA	-	GPIO 21	Hardware I2C Data Line
	SCL	-	GPIO 22	Hardware I2C Clock Line
Grove LED Bar v2.1	GND	Yellow	J3 Ground Rail	Main Ground Line
	VCC	Blue	J3 3.3V Rail	Dedicated Power Line
	DCKI (Clock)	Orange	GPIO 17	Serial Latch Clock Input
	DI (Data)	White	GPIO 16	Serial Data Input
UI Toggle Button	Signal / GND	-	GPIO 12 / GND	Hardware Interrupt UI Selector
Ultrasonic Sensor (HC-SR04)	TRIG	-	GPIO 13	Ultrasonic Pulse Trigger
	ECHO	-	GPIO 25	Pulse-width Duration Input
DHT11 Sensor	DATA	-	GPIO 4	Direct Bit-Bang Data Bus
KY-039 Pulse Sensor	Analog Out	-	GPIO 32	Capillary Density Raw Input
BioAmp EXG Pill	Analog Out	-	GPIO 35	Biopotential Electrocardiograph Input
🖥️ 3. Firmware Source Code (smdp1.ino)
This is the optimized production code running on your Diago Smart Glove project.
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ==========================================
// 🛠️ HARDWARE CONFIGURATION & PIN MAPPINGS
// ==========================================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define I2C_ADDRESS   0x3C  

// I2C Pins
#define OLED_SDA_PIN     21
#define OLED_SCL_PIN     22

// UPDATED: Grove LED Bar v2.1 Pins mapped to J3 Port Configuration
#define BAR_DATA_PIN     16  // J3 Data (DI)
#define BAR_CLK_PIN      17  // J3 Clock (DCKI)

// RESTORED: Standard pins (Safe now that GPIO 12/13 are free)
#define BUTTON_PIN       12  // High-reliability Pin 12
#define TRIG_PIN         13  // Ultrasonic Trigger Pin 13
#define HEART_PIN        32  
#define DHT11_PIN         4  
#define BIOAMP_PIN       35  
#define ECHO_PIN         25  

// ==========================================
// 📊 GLOBAL VARIABLES & STATE MANAGEMENT
// ==========================================
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

volatile int currentScreen = 0;
const int totalScreens = 4;
volatile unsigned long lastButtonPress = 0;
const unsigned long debounceDelay = 200; 

float lpfBPM = 72.0; 
int animationFrame = 0;

// ==========================================
// 🔄 FUNCTION PROTOTYPES
// ==========================================
void sendLedBarCmd(unsigned int data);
void setLedBarLevel(int level);
bool readDirectDHT11(float &temp, float &hum);
float readUltrasonicHeight();
void drawHeaderBanner(const char* title);
void renderHeartbeatScreen();
void renderTemperatureScreen();
void renderWaveformScreen();
void renderHeightScreen();

// ==========================================
// 🕹️ INTERRUPT SERVICE ROUTINE 
// ==========================================
void IRAM_ATTR handleButtonInterrupt() {
  unsigned long currentTime = millis();
  if (currentTime - lastButtonPress>debounceDelay) {
currentScreen = (currentScreen + 1) % totalScreens;
lastButtonPress = currentTime;
  }
}

// ==========================================
// 🚀 SETUP ROUTINE
// ==========================================
void setup() {
Serial.begin(115200);

Wire.begin(OLED_SDA_PIN, OLED_SCL_PIN);

if(!display.begin(SSD1306_SWITCHCAPVCC, I2C_ADDRESS)) {
Serial.println(F("SSD1306 allocation failed."));
  }

display.clearDisplay();
display.drawRect(5, 5, 118, 54, SSD1306_WHITE);
display.drawRect(8, 8, 112, 48, SSD1306_WHITE);
display.setTextColor(SSD1306_WHITE);
display.setTextSize(1);
display.setCursor(18, 24);
display.print("DIAGO SMART GLOVE");
display.setCursor(32, 38);
display.print("J3 BOOT ONLINE...");
display.display();
delay(1500);

pinMode(BAR_DATA_PIN, OUTPUT);
pinMode(BAR_CLK_PIN, OUTPUT);
pinMode(BUTTON_PIN, INPUT_PULLUP);
pinMode(TRIG_PIN, OUTPUT);
pinMode(ECHO_PIN, INPUT);

attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), handleButtonInterrupt, FALLING);

setLedBarLevel(0);
}

void loop() {
display.clearDisplay();
animationFrame = (animationFrame + 1) % 4;

  switch (currentScreen) {
    case 0: renderHeartbeatScreen(); break;
    case 1: renderTemperatureScreen(); break;
    case 2: renderWaveformScreen(); break;
    case 3: renderHeightScreen(); break;
  }

display.display();
delay(60); 
}

// ==========================================
// 🎛️ NATIVE GROVE LED BAR CONTROLLER ENGINE
// ==========================================
void sendLedBarCmd(unsigned int data) {
  for (unsigned char i = 0; i< 16; i++) {
digitalWrite(BAR_DATA_PIN, (data & 0x8000) ?HIGH : LOW);
digitalWrite(BAR_CLK_PIN, LOW);
delayMicroseconds(20); 
digitalWrite(BAR_CLK_PIN, HIGH);
delayMicroseconds(20);
    data <<= 1;
  }
}

void setLedBarLevel(int level) {
  if (level < 0) level = 0;
  if (level > 10) level = 10;

sendLedBarCmd(0x0000); 
  for (int i = 1; i<= 10; i++) {
    if (i<= level) sendLedBarCmd(0x00FF); 
    else sendLedBarCmd(0x0000); 
  }
sendLedBarCmd(0x0000);
sendLedBarCmd(0x0000);
}

// ==========================================
// 🎨 UI DESIGN COMPONENTS (WITH BATTERY STATUS)
// ==========================================
void drawHeaderBanner(const char* title) {
display.fillRect(0, 0, 128, 13, SSD1306_WHITE);
display.setTextColor(SSD1306_BLACK, SSD1306_WHITE);
display.setTextSize(1);
display.setCursor(3, 3);
display.print(title);

display.drawRect(110, 3, 14, 7, SSD1306_BLACK);
display.fillRect(112, 5, 10, 3, SSD1306_BLACK);
display.drawFastVLine(124, 5, 3, SSD1306_BLACK);

display.setTextColor(SSD1306_WHITE);
}

// ==========================================
// 🔬 SENSOR READING UTILITIES
// ==========================================
bool readDirectDHT11(float &temp, float &hum) {
  uint8_t bits[5] = {0, 0, 0, 0, 0};
  uint8_t cnt = 7; uint8_t idx = 0;

pinMode(DHT11_PIN, OUTPUT);
digitalWrite(DHT11_PIN, LOW); delay(18);
digitalWrite(DHT11_PIN, HIGH); delayMicroseconds(40);
pinMode(DHT11_PIN, INPUT);

  unsigned int loopCnt = 10000;
  while(digitalRead(DHT11_PIN) == LOW) if (loopCnt-- == 0) return false;
loopCnt = 10000;
  while(digitalRead(DHT11_PIN) == HIGH) if (loopCnt-- == 0) return false;

  for (int i=0; i<40; i++) {
loopCnt = 10000;
    while(digitalRead(DHT11_PIN) == LOW) if (loopCnt-- == 0) return false;
    unsigned long t = micros();
loopCnt = 10000;
    while(digitalRead(DHT11_PIN) == HIGH) if (loopCnt-- == 0) return false;
    if ((micros() - t) > 40) bits[idx] |= (1 <<cnt);
    if (cnt == 0) { cnt = 7; idx++; } else cnt--;
  }

  if ((bits[0] + bits[1] + bits[2] + bits[3]) == bits[4]) {
    hum = bits[0]; temp = bits[2]; return true;
  }
  return false;
}

float readUltrasonicHeight() {
digitalWrite(TRIG_PIN, LOW); delayMicroseconds(2);
digitalWrite(TRIG_PIN, HIGH); delayMicroseconds(10);
digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH, 30000); 
  if (duration == 0) return -1;
  return (duration * 0.0343) / 2.0;
}

// ==========================================
// 🖥️ OLED USER INTERFACE SCREENS
// ==========================================
void renderHeartbeatScreen() {
drawHeaderBanner("CARDIO");
  int rawPulse = analogRead(HEART_PIN);
  int ledLevel = map(rawPulse, 1800, 3000, 1, 10);
setLedBarLevel(ledLevel);

display.drawRoundRect(64, 18, 60, 42, 4, SSD1306_WHITE);
display.setTextSize(1);
display.setCursor(72, 24); display.print("PULSE");
display.setCursor(72, 44); display.print(rawPulse);

display.setTextSize(2);
display.setCursor(4, 22); display.print("BPM");
display.setCursor(4, 42); display.print((int)lpfBPM);
}

void renderTemperatureScreen() {
drawHeaderBanner("CLIMATE");
  float localTemp = 0, localHum = 0;
  bool active = readDirectDHT11(localTemp, localHum);

  if(active) {
display.drawRect(4, 18, 56, 42, SSD1306_WHITE);
display.setTextSize(1);
display.setCursor(10, 22); display.print("TEMP");
display.setTextSize(2);
display.setCursor(12, 38); display.print((int)localTemp); display.write(247);

display.drawRect(68, 18, 56, 42, SSD1306_WHITE);
display.setTextSize(1);
display.setCursor(74, 22); display.print("HUMID");
display.setTextSize(2);
display.setCursor(78, 38); display.print((int)localHum); display.print("%");

    int tempBar = map((int)localTemp, 20, 40, 1, 10);
setLedBarLevel(tempBar);
  } else {
display.setCursor(12, 32);
display.setTextSize(1);
display.println("[BUS ERROR]");
setLedBarLevel(1); 
  }
}

void renderWaveformScreen() {
drawHeaderBanner("BIOAMP");
display.drawRect(0, 16, 128, 48, SSD1306_WHITE);

  int baseGraphicY = 40;
  for (int x = 2; x < 126; x += 3) {
    int val = analogRead(BIOAMP_PIN);
    int yOffset = map(val, 0, 4095, -18, 18);
display.drawPixel(x, baseGraphicY + yOffset, SSD1306_WHITE);
  }

  int rawBio = analogRead(BIOAMP_PIN);
  int bioLevel = map(rawBio, 1500, 3000, 0, 10);
setLedBarLevel(bioLevel);
}

void renderHeightScreen() {
drawHeaderBanner("ANTHRO");
  float objectDistance = readUltrasonicHeight();

display.setTextSize(1);
display.setCursor(4, 18);
display.print("SCANNING");

display.setCursor(55, 18);
for(int j=0; j<=animationFrame; j++) display.print(".");

display.drawFastHLine(4, 29, 120, SSD1306_WHITE);

  if (objectDistance> 0) {
display.setTextSize(2);
display.setCursor(4, 38);
display.print("D: "); display.print(objectDistance, 1); display.print("cm");
    int distBar = map((int)objectDistance, 5, 100, 10, 1); 
setLedBarLevel(distBar);
  } else {
display.setTextSize(1);
display.setCursor(4, 42);
display.print("OUT OF RANGE");
setLedBarLevel(0);
  }
}
