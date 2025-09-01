#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "BluetoothSerial.h"

// RFID pins
#define SS_PIN 5
#define RST_PIN 2

// Button and Buzzer pins
#define DEC_BUTTON 13
#define RETURN_BUTTON 14
#define BUZZER_PIN 12

// Motor pins
#define MOTOR1_PIN1 32
#define MOTOR1_PIN2 33
#define MOTOR2_PIN1 27
#define MOTOR2_PIN2 4

// LEDs pin
#define LED_PIN 23

// Ultrasonic sensor pins
#define TRIG_PIN 25
#define ECHO_PIN 26

// Bluetooth setup
BluetoothSerial SerialBT;
float totalBill = 0;
bool decrementMode = false;
bool lastDecButtonState = HIGH;
bool lastReturnButtonState = HIGH;
int itemCount[4] = {0, 0, 0, 0};

// RFID setup
MFRC522 rfid(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// RFID active state
bool rfidActive = false;  // Declare rfidActive globally

// Grocery items
struct GroceryItem {
  String name;
  float price;
};

byte tag1UID[] = {0xF2, 0xFD, 0x34, 0x54};
byte tag2UID[] = {0x43, 0xF8, 0x81, 0x50};
byte tag3UID[] = {0x73, 0xF5, 0xCE, 0xFA};
byte tag4UID[] = {0x70, 0x03, 0x64, 0x14};

GroceryItem groceries[] = {
  {"Rasam pkt", 10.0},
  {"Milk", 40.0},
  {"Bread", 30.0},
  {"Eggs", 60.0}
};

// Function prototypes
void handleRFID();
void handleBluetooth();
void processRFIDItem(int itemIndex);
int checkUID();
void moveForward();
void moveBackward();
void turnLeft();
void turnRight();
void stopMotors();
bool isObstacleDetected();
void beepBuzzer();

// Timer for non-blocking delay
unsigned long previousMillis = 0;
const long interval = 250; // Interval for delay replacement

// Setup function
void setup() {
  Serial.begin(9600);
  SerialBT.begin("NextGenCart"); // Bluetooth name
  Serial.println("Bluetooth is ready to pair.");

  // Initialize pins
  pinMode(MOTOR1_PIN1, OUTPUT);
  pinMode(MOTOR1_PIN2, OUTPUT);
  pinMode(MOTOR2_PIN1, OUTPUT);
  pinMode(MOTOR2_PIN2, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(DEC_BUTTON, INPUT_PULLUP);
  pinMode(RETURN_BUTTON, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  
  // RFID setup
  SPI.begin();
  rfid.PCD_Init();

  // LCD setup
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Welcome!");
  delay(2000);  // Allow time for the welcome message
  lcd.clear();
  lcd.print("Select mode");
}

// Main loop
void loop() {
  handleRFID();  // Handle RFID functionality
  handleBluetooth();  // Handle Bluetooth commands

  // Non-blocking motor control and obstacle detection
  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;
    // Ensure motors work with non-blocking delays
    if (isObstacleDetected()) {
      stopMotors();
      SerialBT.print("R"); // Cart is at rest
    }
  }
}

// Handle RFID operations
void handleRFID() {
  bool currentDecButtonState = digitalRead(DEC_BUTTON);
  bool currentReturnButtonState = digitalRead(RETURN_BUTTON);

  // Toggle decrement mode
  if (lastDecButtonState == HIGH && currentDecButtonState == LOW) {
    decrementMode = true;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Deduct Mode");
    delay(2000);
  }

  // Toggle addition mode
  if (lastReturnButtonState == HIGH && currentReturnButtonState == LOW) {
    decrementMode = false;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Add Mode");
    delay(2000);
  }

  lastDecButtonState = currentDecButtonState;
  lastReturnButtonState = currentReturnButtonState;

  // Only scan and display info if RFID is ON and Bluetooth 'D' is pressed
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    if (rfidActive) {
      int itemIndex = checkUID();
      if (itemIndex >= 0) {
        processRFIDItem(itemIndex);
      } else {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Unknown Tag");
        delay(2000);
      }
    }
    rfid.PICC_HaltA();
  }
}

// Process RFID item for shopping cart
void processRFIDItem(int itemIndex) {
  String itemName = groceries[itemIndex].name;
  float itemPrice = groceries[itemIndex].price;

  if (!decrementMode) {
    if (itemCount[itemIndex] > 0) {
      beepBuzzer(); // Buzzer for repeated item
    }
    itemCount[itemIndex]++;
    totalBill += itemPrice;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Added: ");
    lcd.print(itemName.substring(0, 8));
    lcd.setCursor(0, 1);
    lcd.print("Price: ");
    lcd.print(itemPrice, 2);
    delay(2000);
  } else {
    if (itemCount[itemIndex] > 0) {
      itemCount[itemIndex]--;
      totalBill -= itemPrice;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Removed: ");
      lcd.print(itemName.substring(0, 8));
      lcd.setCursor(0, 1);
      lcd.print("Price: ");
      lcd.print(itemPrice, 2);
      delay(2000);
    } else {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Item Not Found");
      delay(2000);
    }
  }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Total Bill: ");
  lcd.setCursor(0, 1);
  lcd.print(totalBill, 2);
  delay(3000);
}

// Check UID for RFID tags
int checkUID() {
  for (int i = 0; i < 4; i++) {
    bool match = true;
    for (byte j = 0; j < rfid.uid.size; j++) {
      if (rfid.uid.uidByte[j] != (i == 0 ? tag1UID[j] : i == 1 ? tag2UID[j] : i == 2 ? tag3UID[j] : tag4UID[j])) {
        match = false;
        break;
      }
    }
    if (match) {
      return i;
    }
  }
  return -1;
}

// Handle Bluetooth commands for RFID and motor movement
void handleBluetooth() {
  static bool motorActive = false;

  if (SerialBT.available()) {
    char command = SerialBT.read(); // Read Bluetooth command

    // Toggle RFID control
    if (command == 'D') {
      rfidActive = true;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("RFID ON");
      delay(1000);
    } 
    if (command == 'f') {
      rfidActive = false;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("RFID OFF");
      delay(1000);
    }

    // Toggle motor control
    if (command == 'O') {
      motorActive = true;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Motor ON");
      delay(1000);
    } 
    if (command == 'o') {
      motorActive = false;
      stopMotors();  // Ensure motors stop if turned off
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Motor OFF");
      delay(1000);
    }

    // Motor movement commands
    if (motorActive) {
      switch (command) {
        case 'F': 
          moveForward();
          SerialBT.print("M");  // Cart is in motion
          break;
        case 'B': 
          moveBackward();
          SerialBT.print("M");  // Cart is in motion
          break;
        case 'L': 
          turnLeft();
          SerialBT.print("M");  // Cart is in motion
          break;
        case 'R': 
          turnRight();
          SerialBT.print("M");  // Cart is in motion
          break;
        case 'S': 
          stopMotors();
          SerialBT.print("R");  // Cart is at rest
          break;
        default: 
          break;
      }
    }
  }
}

// Motor movement functions
void moveForward() {
  digitalWrite(MOTOR1_PIN1, HIGH);
  digitalWrite(MOTOR1_PIN2, LOW);
  digitalWrite(MOTOR2_PIN1, HIGH);
  digitalWrite(MOTOR2_PIN2, LOW);
}

void moveBackward() {
  digitalWrite(MOTOR1_PIN1, LOW);
  digitalWrite(MOTOR1_PIN2, HIGH);
  digitalWrite(MOTOR2_PIN1, LOW);
  digitalWrite(MOTOR2_PIN2, HIGH);
}

void turnLeft() {
  digitalWrite(MOTOR1_PIN1, LOW);
  digitalWrite(MOTOR1_PIN2, HIGH);
  digitalWrite(MOTOR2_PIN1, HIGH);
  digitalWrite(MOTOR2_PIN2, LOW);
}

void turnRight() {
  digitalWrite(MOTOR1_PIN1, HIGH);
  digitalWrite(MOTOR1_PIN2, LOW);
  digitalWrite(MOTOR2_PIN1, LOW);
  digitalWrite(MOTOR2_PIN2, HIGH);
}

void stopMotors() {
  digitalWrite(MOTOR1_PIN1, LOW);
  digitalWrite(MOTOR1_PIN2, LOW);
  digitalWrite(MOTOR2_PIN1, LOW);
  digitalWrite(MOTOR2_PIN2, LOW);
}

// Function to check if obstacle is detected using the ultrasonic sensor
bool isObstacleDetected() {
  long duration;
  float distance;

  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  duration = pulseIn(ECHO_PIN, HIGH);
  distance = (duration * 0.0343) / 2; // Convert to cm

  return distance <= 40.0 && distance > 0;
}

// Function to beep the buzzer for repeated items
void beepBuzzer() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(500);  // Buzzer duration
  digitalWrite(BUZZER_PIN, LOW);
}