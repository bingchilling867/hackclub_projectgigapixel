#include <Servo.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ====== SERVO PINS ======
#define PAN_PIN 9
#define TILT_PIN 10
#define SHUTTER_PIN 11

// ====== BUTTON PINS ======
#define ROW_PLUS 2
#define ROW_MINUS 3
#define COL_PLUS 4
#define COL_MINUS 5

// ====== CONSTANTS ======
const float SENSOR_MP = 12.0;
const float FOV_DEG = 20.0;
const float OVERLAP = 0.10;
const float STEP_DEG = FOV_DEG * (1.0 - OVERLAP);   // 18 degrees

const int SHUTTER_REST = 90;
const int SHUTTER_FIRE = SHUTTER_REST + 20;

unsigned long lastInputTime = 0;
const unsigned long AUTO_RUN_DELAY = 10000; // 10 seconds

// ====== STATE ======
int rows = 3;
int cols = 3;

Servo panServo;
Servo tiltServo;
Servo shutterServo;

// ====== FUNCTIONS ======

void updateDisplay() {
  float effectiveMP = rows * cols * SENSOR_MP * 0.81; // 0.9 * 0.9 overlap
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(0, 0);
  display.print("R:");
  display.print(rows);
  display.print(" C:");
  display.print(cols);

  display.setCursor(0, 30);
  display.print((int)effectiveMP);
  display.print(" MP");

  display.display();
}

void fireShutter() {
  shutterServo.write(SHUTTER_FIRE);
  delay(200);
  shutterServo.write(SHUTTER_REST);
  delay(200);
}

void moveServos(float panAngle, float tiltAngle) {
  panServo.write(panAngle);
  tiltServo.write(tiltAngle);
  delay(500);
}

void runCapture() {
  float panStart = 90 - ((cols - 1) * STEP_DEG / 2.0);
  float tiltStart = 90 - ((rows - 1) * STEP_DEG / 2.0);

  for (int r = 0; r < rows; r++) {
    for (int c = 0; c < cols; c++) {
      float panA = panStart + c * STEP_DEG;
      float tiltA = tiltStart + r * STEP_DEG;

      moveServos(panA, tiltA);
      fireShutter();
    }
  }

  // Return to center
  moveServos(90, 90);
}

void checkButtons() {
  bool changed = false;

  if (!digitalRead(ROW_PLUS)) { rows++; changed = true; delay(200); }
  if (!digitalRead(ROW_MINUS)) { if (rows > 1) rows--; changed = true; delay(200); }
  if (!digitalRead(COL_PLUS)) { cols++; changed = true; delay(200); }
  if (!digitalRead(COL_MINUS)) { if (cols > 1) cols--; changed = true; delay(200); }

  if (changed) {
    lastInputTime = millis();
    updateDisplay();
  }
}

void setup() {
  // Buttons
  pinMode(ROW_PLUS, INPUT_PULLUP);
  pinMode(ROW_MINUS, INPUT_PULLUP);
  pinMode(COL_PLUS, INPUT_PULLUP);
  pinMode(COL_MINUS, INPUT_PULLUP);

  // Servos
  panServo.attach(PAN_PIN);
  tiltServo.attach(TILT_PIN);
  shutterServo.attach(SHUTTER_PIN);

  panServo.write(90);
  tiltServo.write(90);
  shutterServo.write(SHUTTER_REST);

  // OLED
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.display();

  lastInputTime = millis();
  updateDisplay();
}

void loop() {
  checkButtons();

  if (millis() - lastInputTime > AUTO_RUN_DELAY) {
    runCapture();
    lastInputTime = millis();
  }
}
