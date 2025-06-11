# Arduino-CYD-Esp32
Accurate Frequency Meter

#include <TFT_eSPI.h>
#include <SPI.h>

#define TFT_BLACK 0x0000
#define TFT_WHITE 0xFFFF
#define TFT_RED 0xF800
#define TFT_BLUE 0x001F
#define TFT_GREEN 0x07E0
#define TFT_YELLOW 0xFFE0
#define TFT_GRAY 0x8410  // Custom gray color

#define OUTPUT_GPIO 5  // Optional output

TFT_eSPI tft = TFT_eSPI();

volatile unsigned long lastPulseTime = 0;
volatile unsigned long newPulseTime = 0;
volatile unsigned long intervals[10];
volatile byte sampleIndex = 0;
volatile bool ready = false;

float averageInterval = 0;
const int waterfallHeight = 100;  // Space for the waterfall
const int displayWidth = 220; // Width of the waterfall region
int waterfallData[waterfallHeight]; // Array to store recent frequency values

void IRAM_ATTR sens() {
  newPulseTime = micros();
  if (lastPulseTime > 0) {
    intervals[sampleIndex] = newPulseTime - lastPulseTime;
    sampleIndex = (sampleIndex + 1) % 10;
    ready = true;
  }
  lastPulseTime = newPulseTime;
}

uint16_t getColorForFrequency(float freq) {
  if (freq < 500) return TFT_BLUE;
  if (freq < 1500) return TFT_GREEN;
  if (freq < 2500) return TFT_YELLOW;
  return TFT_RED;
}

void updateWaterfall(float freq) {
  for (int i = waterfallHeight - 1; i > 0; i--) {
    waterfallData[i] = waterfallData[i - 1]; // Shift previous values up
  }
  waterfallData[0] = (int)freq; // Add new frequency at bottom

  for (int i = 0; i < waterfallHeight; i++) {
    tft.fillRect(10, 150 + i, displayWidth, 1, getColorForFrequency(waterfallData[i]));
  }
}

void drawFrequencyScale() {
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.setTextSize(1);

  // Hz markers along the lower axis with corrected spacing
  tft.setCursor(10, 260);  tft.print("0KHz");
  tft.setCursor(40, 260);  tft.print("0.5KHz");
  tft.setCursor(80, 260);  tft.print("1KHz");
  tft.setCursor(120, 260); tft.print("1.5KHz");
  tft.setCursor(160, 260); tft.print("2KHz");
  tft.setCursor(200, 260); tft.print("2.5KHz");
  tft.setCursor(240, 260); tft.print("3KHz");
  tft.setCursor(270, 260); tft.print("3.4KHz"); // Adjusted placement

  // Vertical reference lines with corrected alignment
  tft.drawLine(10, 130, 10, 250, TFT_GRAY);
  tft.drawLine(40, 130, 40, 250, TFT_GRAY);
  tft.drawLine(80, 130, 80, 250, TFT_GRAY);
  tft.drawLine(120, 130, 120, 250, TFT_GRAY);
  tft.drawLine(160, 130, 160, 250, TFT_GRAY);
  tft.drawLine(200, 130, 200, 250, TFT_GRAY);
  tft.drawLine(240, 130, 240, 250, TFT_GRAY);
  tft.drawLine(270, 130, 270, 250, TFT_GRAY); // Adjusted placement for 3.4KHz
}

void checkFrequencyAlert(float freq) {
  // Clear previous flag area before updating
  tft.fillRect(10, 120, displayWidth, 20, TFT_BLACK);

  if (freq <= 0.1) {  
    tft.setTextColor(TFT_RED, TFT_BLACK);
    tft.setCursor(10, 120);
    tft.print("ALERT: LOW FREQ!");
  }
  else if (freq >= 2400) {  
    tft.setTextColor(TFT_RED, TFT_BLACK);
    tft.setCursor(10, 120);
    tft.print("ALERT: HIGH FREQ!");
  }
}

void setup() {
  tft.init();
  tft.setRotation(0);
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE, TFT_BLACK);

  // Adjust header font size
  tft.setTextSize(1);  
  tft.drawCentreString("Accurate Frequency", 120, 5, 2);  

  Serial.begin(9600);
  attachInterrupt(digitalPinToInterrupt(3), sens, RISING);
  pinMode(OUTPUT_GPIO, OUTPUT);
}

void loop() {
  static unsigned long lastUpdate = 0;
  if (millis() - lastUpdate >= 500 && ready) {
    lastUpdate = millis();

    noInterrupts();
    unsigned long sum = 0;
    byte count = 0;
    for (int i = 0; i < 10; i++) {
      if (intervals[i] > 0) {
        sum += intervals[i];
        count++;
      }
    }
    interrupts();

    if (count > 0) {
      averageInterval = float(sum) / count;
      float freq = 1e6 / averageInterval;

      Serial.print((int)freq); Serial.print(" ");
      Serial.print((int)freq, BIN); Serial.print(" ");
      Serial.println((int)freq, HEX);

      digitalWrite(OUTPUT_GPIO, ((int)freq) % 2);

      // TFT Display Update
      tft.fillRect(10, 40, displayWidth, 80, TFT_BLACK);
      tft.setCursor(10, 40);
      tft.setTextSize(2);  
      tft.setTextColor(TFT_RED, TFT_BLACK);   tft.print("Hz:  ");  tft.println((int)freq);
      tft.setCursor(10, 60);
      tft.setTextColor(TFT_BLUE, TFT_BLACK);  tft.print("Bin: ");  tft.println((int)freq, BIN);
      tft.setCursor(10, 90);
      tft.setTextColor(TFT_GREEN, TFT_BLACK); tft.print("Hex: ");  tft.println((int)freq, HEX);

      // Update waterfall, check alerts, and draw scale
      updateWaterfall(freq);
      checkFrequencyAlert(freq); // **Calling the frequency alert function**
      drawFrequencyScale();  // **Calling the frequency scale function**
    }
  }
}

