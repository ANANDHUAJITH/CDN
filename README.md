#include <Servo.h>

#define BTN_START 2
#define LED_FILL 3
#define LED_DRAIN 4
#define LED_DONE 5

Servo drum;

bool started = false;

void setup() {
  pinMode(BTN_START, INPUT_PULLUP);
  pinMode(LED_FILL, OUTPUT);
  pinMode(LED_DRAIN, OUTPUT);
  pinMode(LED_DONE, OUTPUT);

  drum.attach(6); 
  drum.write(90);  
  Serial.begin(9600);
  Serial.println("Washing Machine Ready - Press Button to Start");
}

void loop() {
  if (!started && digitalRead(BTN_START) == LOW) {
    delay(200);  
    started = true;
    Serial.println("Cycle Started...");
    startCycle();
    started = false;
  }
}

void startCycle() {
  
  Serial.println("Filling Water...");
  digitalWrite(LED_FILL, HIGH);
  delay(3000);
  digitalWrite(LED_FILL, LOW);

  Serial.println("Washing...");
  for (int i = 0; i < 3; i++) {
    Serial.println("Drum Rotating CW...");
    drum.write(150);
    delay(1000);
    Serial.println("Drum Rotating CCW...");
    drum.write(30);
    delay(1000);
  }
  drum.write(90); 

  Serial.println("Draining...");
  digitalWrite(LED_DRAIN, HIGH);
  delay(2000);
  digitalWrite(LED_DRAIN, LOW);

  Serial.println("Refilling for Rinse...");
  digitalWrite(LED_FILL, HIGH);
  delay(2000);
  digitalWrite(LED_FILL, LOW);

  Serial.println("Rinsing...");
  drum.write(150);
  delay(1000);
  drum.write(30);
  delay(1000);
  drum.write(90);

  Serial.println("Final Drain...");
  digitalWrite(LED_DRAIN, HIGH);
  delay(2000);
  digitalWrite(LED_DRAIN, LOW);

  Serial.println("Spinning (Dry)...");
  for (int i = 0; i < 10; i++) {
    drum.write(150);
    delay(200);
    drum.write(30);
    delay(200);
  }
  drum.write(90);

  Serial.println("Cycle Complete!");
  for (int i = 0; i < 3; i++) {
    digitalWrite(LED_DONE, HIGH);
    delay(500);
    digitalWrite(LED_DONE, LOW);
    delay(500);
  }
  Serial.println("Ready for next cycle.");
}
