#include <Servo.h>

// Pines
#define RELAY_PIN 53
#define SERVO_PIN 52

#define TRIG_FRONT 31
#define ECHO_FRONT 30
#define TRIG_BACK 46
#define ECHO_BACK 47
#define TRIG_RIGHT 44
#define ECHO_RIGHT 45
#define TRIG_LEFT 43
#define ECHO_LEFT 42

// Sensor de color
#define COLOR_OUT 38
#define COLOR_S2 39
#define COLOR_S3 36
#define COLOR_LED 34
#define COLOR_S0 37
#define COLOR_S1 35

Servo steeringServo;

// Configuracion de relay
#define RELAY_ACTIVE_STATE LOW  // Cambia a HIGH si tu relay se activa con HIGH

// Variables estabilidad color
String lastColor = "NONE";
int colorCount = 0;
const int colorThreshold = 5; // confirmaciones

long readDistance(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duration = pulseIn(echoPin, HIGH, 30000);
  long distance = duration * 0.034 / 2;
  return distance;
}

void setSteering(int angle) {
  steeringServo.write(angle);
}

String detectColor() {
  digitalWrite(COLOR_S2, LOW);
  digitalWrite(COLOR_S3, LOW);
  int red = pulseIn(COLOR_OUT, LOW, 30000);

  digitalWrite(COLOR_S2, HIGH);
  digitalWrite(COLOR_S3, HIGH);
  int green = pulseIn(COLOR_OUT, LOW, 30000);

  if (red > 0 && green > 0) {
    if (red < green * 0.6) return "RED";
    else if (green < red * 0.6) return "GREEN";
  }
  return "NONE";
}

void stableColorCheck(String currentColor) {
  if (currentColor == lastColor && currentColor != "NONE") colorCount++;
  else {
    colorCount = 0;
    lastColor = currentColor;
  }
}

void setup() {
  Serial.begin(9600);

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, RELAY_ACTIVE_STATE); // encender motor

  steeringServo.attach(SERVO_PIN);
  setSteering(90);

  pinMode(TRIG_FRONT, OUTPUT); pinMode(ECHO_FRONT, INPUT);
  pinMode(TRIG_BACK, OUTPUT); pinMode(ECHO_BACK, INPUT);
  pinMode(TRIG_RIGHT, OUTPUT); pinMode(ECHO_RIGHT, INPUT);
  pinMode(TRIG_LEFT, OUTPUT); pinMode(ECHO_LEFT, INPUT);

  pinMode(COLOR_OUT, INPUT);
  pinMode(COLOR_S2, OUTPUT);
  pinMode(COLOR_S3, OUTPUT);
  pinMode(COLOR_LED, OUTPUT);
  pinMode(COLOR_S0, OUTPUT);
  pinMode(COLOR_S1, OUTPUT);

  digitalWrite(COLOR_LED, HIGH);
  digitalWrite(COLOR_S0, HIGH);
  digitalWrite(COLOR_S1, HIGH);

  Serial.println("Sistema iniciado.");
}

void loop() {
  long distFront = readDistance(TRIG_FRONT, ECHO_FRONT);
  long distBack = readDistance(TRIG_BACK, ECHO_BACK);
  long distRight = readDistance(TRIG_RIGHT, ECHO_RIGHT);
  long distLeft = readDistance(TRIG_LEFT, ECHO_LEFT);

  Serial.print("F: "); Serial.print(distFront);
  Serial.print(" B: "); Serial.print(distBack);
  Serial.print(" R: "); Serial.print(distRight);
  Serial.print(" L: "); Serial.println(distLeft);

  String color = detectColor();
  stableColorCheck(color);

  if (distFront < 20 && distFront > 0) {
    Serial.println("Obstáculo frontal: Deteniendo.");
    digitalWrite(RELAY_PIN, !RELAY_ACTIVE_STATE);
    delay(700);
    digitalWrite(RELAY_PIN, RELAY_ACTIVE_STATE);
    setSteering(90);
    return;
  }

  if (distBack < 15 && distBack > 0) {
    Serial.println("Obstáculo trasero detectado, evitando retroceso.");
    digitalWrite(RELAY_PIN, RELAY_ACTIVE_STATE);
  }

  if (colorCount >= colorThreshold) {
    Serial.print("Color estable: "); Serial.println(color);

    if (color == "RED" && distRight > 25) {
      Serial.println("Cambio de carril a la derecha.");
      setSteering(160);
      delay(900);
      setSteering(90);
    }
    else if (color == "GREEN" && distLeft > 25) {
      Serial.println("Cambio de carril a la izquierda.");
      setSteering(20);
      delay(900);
      setSteering(90);
    }

    colorCount = 0;
  } else {
    if (distRight < 15 && distRight > 0 && distLeft >= 15) {
      Serial.println("Obstáculo derecha: girando a la izquierda.");
      setSteering(40);
      delay(400);
      setSteering(90);
    }
    else if (distLeft < 15 && distLeft > 0 && distRight >= 15) {
      Serial.println("Obstáculo izquierda: girando a la derecha.");
      setSteering(140);
      delay(400);
      setSteering(90);
    }
    else if (distLeft < 15 && distRight < 15 && distLeft > 0 && distRight > 0) {
      Serial.println("Obstáculos en ambos lados: Deteniendo.");
      digitalWrite(RELAY_PIN, !RELAY_ACTIVE_STATE);
      delay(1000);
      digitalWrite(RELAY_PIN, RELAY_ACTIVE_STATE);
      setSteering(90);
    } else {
      setSteering(90);
    }
  }
  delay(100);
}
