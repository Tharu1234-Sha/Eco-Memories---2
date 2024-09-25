#include <WiFi.h>
#include <FirebaseESP32.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>
#include <Arduino.h>
#include <ESP32Servo.h>  // Servo library for ESP32

// Replace these with your network credentials
#define WIFI_SSID "Tharu"
#define WIFI_PASSWORD "fnnc7173"

// Firebase objects
FirebaseData firebaseData;
FirebaseAuth auth;
FirebaseConfig config;

LiquidCrystal_I2C lcd(0x27, 20, 4);

// Define the PIR sensor pin
#define PIR_PIN 35

// Define the IR sensor pin
#define IR_PIN 34

// Keypad configuration
const byte ROWS = 4; // Four rows
const byte COLS = 4; // Four columns
char keys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};

// GPIO Pins for ESP32 for rows and columns
byte rowPins[ROWS] = {32, 33 , 25 , 26};  // ESP32 GPIO pins for rows
byte colPins[COLS] = {27, 14, 12, 13};  // ESP32 GPIO pins for columns


// Create Keypad object
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Define a variable to track the system state (0 = wait for motion, 1 = input bottle count, 2 = bottle counting)
int systemState = 0; 

// Define variables for bottle count
int enteredBottleCount = 0;
int actualBottleCount = 0;
int Bottle = 0;
String bottleCountStr = "";  // To accumulate the digits entered
int Nkeytag = 10;
int Distance = 0;

// Define char key
char key;

// Define the servo object and pin
Servo myServo;  // Create a Servo object
int servoPin = 15;  // Define which GPIO pin is connected to the servo

// Define motor driver
const int IN1 = 23;
const int IN2 = 2;


// Speed of motor (0 to 255)
int motorSpeed = 250;  // Adjust the motor speed here

// Ultrasonic Sensor Pins
const int trigPin = 5;  // Changed to avoid conflict
const int echoPin = 18;  // Set to the pin you connected to the Echo pin

void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.backlight();
  
  pinMode(trigPin, OUTPUT); // Trig pin as output
  pinMode(echoPin, INPUT);  // Echo pin as input

  // Set PIR_PIN and IR_PIN as inputs
  pinMode(PIR_PIN, INPUT);
  pinMode(IR_PIN, INPUT);
  
  // Attach the servo to the defined pin
  myServo.attach(servoPin);
  
  // Print a startup message
  Serial.println("PIR Motion Detector Initialized");
  lcd.setCursor(7, 0);
  lcd.print("ECO-MEMO");
  delay(1000);
  lcd.clear();

  // Set motor control pins as output
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
 
  
  // Wi-Fi connection
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.println(" Connected!");

  // Firebase configuration
  config.host = "https://eco-memories-default-rtdb.firebaseio.com/";
  config.signer.tokens.legacy_token = "AIzaSyA1UAQ0d5AjrJZvoljpXbT7QrUI9wr_eRI"; // Use your API key here
  
  // Connect to Firebase
  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
}

void loop() {
  // State 0: Detect motion
  if (systemState == 0) {
    // Read the state of the PIR sensor
    int pirState = digitalRead(PIR_PIN);
    
    if (pirState == HIGH) {
      // If motion is detected
      Serial.println("Motion Detected!");
      
      // Print message on LCD
      lcd.clear(); // Clear any previous content
      lcd.setCursor(0, 0); // Set cursor to column 0, line 0
      lcd.print("Hello,");
      lcd.setCursor(0, 1); // Set cursor to column 0, line 1
      lcd.print("How Many Bottles ?");
      
      // Output the same message to the Serial Monitor
      Serial.println("How many bottles are you going to put?");
      
      // Set system state to 1 to stop motion detection and move to bottle count input
      systemState = 1;
    } else {
      // No motion detected
      Serial.println("No Motion Detected");
    }
  }
  
  // State 1: Enter bottle count via keypad
  else if (systemState == 1) {
    char key = keypad.getKey();  // Get the key pressed

    if (key) {
      // Print the key to Serial Monitor and show on LCD
      Serial.print("Putting Bottle: ");
      Serial.println(key);

      if (key >= '0' && key <= '9') {
        // Accumulate digits
        bottleCountStr += key;

        // Display the key press on the LCD (append to the second line)
        lcd.setCursor(0, 2); // Set cursor to the next line
        lcd.print("Count: ");
        lcd.print(bottleCountStr);  // Display entered number so far
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Enter A to confirme");
        lcd.setCursor(0, 1);
        lcd.print("Enter * to Reset");
      } 
       
      // If the user presses 'A', confirm the bottle count and move to bottle counting process
      else if (key == 'A') {
        if (bottleCountStr.length() > 0) {  // Make sure something was entered
          enteredBottleCount = bottleCountStr.toInt();  // Convert string to integer
          Serial.println("Bottle count confirmed.");


          // Move to the bottle counting state
          systemState = 2;

          // Reset actual bottle count to 0
          actualBottleCount = 0;
          
            // Activate the servo motor (e.g., move to 90 degrees)
        
          myServo.write(90);  // Move the servo to 90°
          Serial.println("Door open");
          delay(1000);  // Keep the servo in position for 1 second
          
          // Clear the LCD
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Insert bottles...");
        } else {
          lcd.setCursor(0, 2);
          lcd.print("Invalid count!");
        }
        bottleCountStr = "";  // Reset the string for next input
      }
      // If '*' is pressed, reset the bottle count and ask the user to re-enter it
      else if (key == '*') {
        bottleCountStr = "";  // Clear the bottle count input
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("How many bottles?");
        lcd.print(key);
        Serial.println("Bottle count reset, please enter again.");
      }
      
      else if (key == 'D') {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Keytags are full");
        Serial.println("Keytags are full");
        Nkeytag = 10 ;
         systemState = 4;
      }
      else if (key == 'B'){
        Bottle = 0 ;
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Bin is Empty");
        Serial.println("Bottle Count Rsset");
         systemState = 4;
      }

      // Small delay to avoid bouncing issues
      delay(200);
    }
  }
  
  // State 2: Bottle counting process
  else if (systemState == 2) {
    Serial.print("Checking bottle count...\n");

    // Check for bottles until the actual count equals the entered count
    while (actualBottleCount < enteredBottleCount) {
      // Continuously check the IR sensor for bottle detection
      int irState = digitalRead(IR_PIN);
        
      if (irState == LOW) {
        // Bottle detected, increment the actual count
        actualBottleCount++;
        Serial.print("Bottle detected. Count: ");
        Serial.println(actualBottleCount);
        lcd.clear();
        lcd.setCursor(0,1);
        lcd.print("Count - ");
        lcd.print(actualBottleCount);
        delay(3000);

        
        
      }
    }
    // Reset the servo to its original position
    myServo.write(0);  // Move the servo back to 0°
    delay(1000);  // Wait for 1 second
        lcd.clear(); 
        lcd.setCursor(0, 1);
        lcd.print(" Done ");
        
    systemState=3;
  }

  else if(systemState ==3){
    Bottle += enteredBottleCount;  
    Serial.println("Collective Bottles -   ");
    Serial.print(Bottle);

    // Dispense key tag if conditions are met
    if (Bottle % 3 == 0 && Bottle > 0 && Nkeytag > 0) {
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      delay(1000);
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      
      Serial.println("You Win a Keytag");
      lcd.clear();
      lcd.setCursor(0, 1);
      lcd.print(" You Won a Keytag ");
      delay(1000);  // Run motor for 3 seconds
      lcd.clear();
      lcd.setCursor(0,2);
      lcd.print("Thank you !");
     

      Nkeytag--;
      Serial.println("No of keytags remaining:");
      Serial.print(Nkeytag);
      
    systemState = 4;
    
  }
    else{
      lcd.clear();
      lcd.setCursor(0, 1);
      lcd.print("OOps!");
      lcd.setCursor(0, 2);
      lcd.print("You Lost The Chance");
      delay(1000);
      lcd.clear();
      lcd.setCursor(0,2);
      lcd.print("Thank you !");
     
      systemState = 4;
      }
  
  }

  // Ultrasonic sensor process
  else if (systemState == 4) {
    // Trigger the ultrasonic sensor and measure distance
    long duration, distance;
digitalWrite(trigPin, LOW);
delayMicroseconds(2);
digitalWrite(trigPin, HIGH);
delayMicroseconds(10);
digitalWrite(trigPin, LOW);

duration = pulseIn(echoPin, HIGH);
distance = duration * 0.0344 / 2;

Serial.println("Distance: " + String(distance) + " cm");

String binstate;
if (distance > 10) {
   binstate = "false";
} else {
   binstate = "ture";
}

// Send data to Firebase
if (Firebase.setInt(firebaseData, "/IsFull", binstate == "false" ? 0 : 1)) {
   Serial.println("Bin State uploaded successfully!");
} else {
   Serial.println("Firebase error: " + firebaseData.errorReason());
}

if (Firebase.setInt(firebaseData, "/Keytag", Nkeytag)) {
   Serial.println("keytag uploaded successfully!");
} else {
   Serial.println("keytag Firebase error: " + firebaseData.errorReason());
}

if (Firebase.setInt(firebaseData, "/Bottle",Bottle )) {
   Serial.println("Bottle count  uploaded successfully!");
} else {
   Serial.println("keytag Firebase error: " + firebaseData.errorReason());
}


    

    // Reset to state 0 after the process
    systemState = 0;
  }
}
