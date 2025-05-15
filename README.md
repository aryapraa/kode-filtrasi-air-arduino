#include <Wire.h>
#include <LiquidCrystal_I2C.h>

#define TDS_PIN A1
#define TURBIDITY_PIN A0
#define PH_PIN A2

#define RELAY_BERSIH 2
#define RELAY_FILTER 3
#define RELAY_DOSING 4
#define RELAY_DRAINASE 5     // IN4
#define RELAY_POMPA_AWAL 6   // IN5

#define TRIG_PIN 6
#define ECHO_PIN 7

LiquidCrystal_I2C lcd(0x27, 16, 2);

float VREF = 5.0;
float tdsFactor = 0.5;
float maxKedalaman = 15.0;

unsigned long dosingStartTime = 0;
bool dosingActive = false;
bool delayAfterDosing = false;
unsigned long delayStartTime = 0;

const unsigned long dosingDuration = 50000;      // 50 detik
const unsigned long delayDuration = 600000;      // 10 menit
const unsigned long drainaseDuration = 3000;     // 3 detik

unsigned long lastDrainaseTime = 0;
bool drainaseActive = false;

float mapfloat(float x, float in_min, float in_max, float out_min, float out_max) {
  return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min;
}

float getNTU(float voltage) {
  float ntu;
  if (voltage >= 3.1) ntu = 0;
  else if (voltage >= 2.8) ntu = mapfloat(voltage, 2.8, 3.1, 10, 0);
  else if (voltage >= 2.2) ntu = mapfloat(voltage, 2.2, 2.8, 100, 10);
  else if (voltage >= 1.5) ntu = mapfloat(voltage, 1.5, 2.2, 500, 100);
  else if (voltage >= 1.0) ntu = mapfloat(voltage, 1.0, 1.5, 1000, 500);
  else ntu = 3000;
  return constrain(ntu, 0, 3000);
}

float getPH(float analogValue) {
  float voltage = analogValue * (5.0 / 1023.0);
  return -6.383 * voltage + 34.694;
}

float readAverageAnalog(int pin, int sampleCount = 10) {
  long sum = 0;
  for (int i = 0; i < sampleCount; i++) {
    sum += analogRead(pin);
    delay(10);
  }
  return (float)sum / sampleCount;
}

String statusGabungan(float ntu, float tds, float pH) {
  bool bersih = (ntu < 10) && (tds <= 300) && (pH >= 6.5 && pH <= 8.5);
  int keruhCount = 0;
  if (ntu >= 25 && ntu <= 70) keruhCount++;
  if (tds > 300 && tds <= 500) keruhCount++;
  if (pH >= 9 && pH <= 10) keruhCount++;
  int kotorCount = 0;
  if (ntu > 70) kotorCount++;
  if (tds > 500) kotorCount++;
  if (pH > 10) kotorCount++;
  if (bersih) return "Bersih";
  if (kotorCount >= 2) return "Kotor";
  if (keruhCount >= 2) return "Keruh";
  if (ntu > 70) return "Kotor";
  if (ntu >= 25) return "Keruh";
  return "Bersih";
}

void kontrolRelay(String status) {
  digitalWrite(RELAY_BERSIH, LOW);
  digitalWrite(RELAY_FILTER, LOW);
  digitalWrite(RELAY_DOSING, LOW);

  if (status == "Bersih") {
    digitalWrite(RELAY_BERSIH, HIGH);
  } else if (status == "Keruh") {
    digitalWrite(RELAY_FILTER, HIGH);
  } else if (status == "Kotor") {
    digitalWrite(RELAY_DOSING, HIGH);
    dosingActive = true;
    dosingStartTime = millis();
  }
}

float bacaTinggiAir(float maxKedalaman) {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long durasi = pulseIn(ECHO_PIN, HIGH);
  float jarak = durasi * 0.034 / 2;
  float tinggiAir = maxKedalaman - jarak;
  if (tinggiAir < 0) tinggiAir = 0;
  return tinggiAir;
}

void cekDrainase(float tinggiAir) {
  if (!drainaseActive && tinggiAir >= 15.0) {
    digitalWrite(RELAY_DRAINASE, HIGH);
    drainaseActive = true;
    lastDrainaseTime = millis();
  }

  if (drainaseActive && millis() - lastDrainaseTime >= drainaseDuration) {
    digitalWrite(RELAY_DRAINASE, LOW);
    drainaseActive = false;
  }
}

void setup() {
  Serial.begin(9600);
  lcd.init();
  lcd.backlight();

  pinMode(RELAY_BERSIH, OUTPUT);
  pinMode(RELAY_FILTER, OUTPUT);
  pinMode(RELAY_DOSING, OUTPUT);
  pinMode(RELAY_DRAINASE, OUTPUT);
  pinMode(RELAY_POMPA_AWAL, OUTPUT);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  digitalWrite(RELAY_BERSIH, LOW);
  digitalWrite(RELAY_FILTER, LOW);
  digitalWrite(RELAY_DOSING, LOW);
  digitalWrite(RELAY_DRAINASE, LOW);
  digitalWrite(RELAY_POMPA_AWAL, HIGH);  // Pompa awal selalu aktif
}

void loop() {
  if (dosingActive) {
    if (millis() - dosingStartTime >= dosingDuration) {
      digitalWrite(RELAY_DOSING, LOW);
      dosingActive = false;
      delayAfterDosing = true;
      delayStartTime = millis();
    }
    return;
  }

  if (delayAfterDosing) {
    if (millis() - delayStartTime >= delayDuration) {
      delayAfterDosing = false;
    } else {
      return;
    }
  }

  float tdsRaw = readAverageAnalog(TDS_PIN);
  float tdsVoltage = tdsRaw * VREF / 1023.0;
  float tdsValue = (133.42 * pow(tdsVoltage, 3) - 255.86 * pow(tdsVoltage, 2) + 857.39 * tdsVoltage) * tdsFactor;

  float turbRaw = readAverageAnalog(TURBIDITY_PIN);
  float turbVoltage = turbRaw * VREF / 1023.0;
  float ntu = getNTU(turbVoltage);

  float phRaw = readAverageAnalog(PH_PIN);
  float pH = getPH(phRaw);

  float tinggiAir = bacaTinggiAir(maxKedalaman);
  String status = statusGabungan(ntu, tdsValue, pH);

  kontrolRelay(status);
  cekDrainase(tinggiAir);

  Serial.println("===========================");
  Serial.println("=== Pembacaan Sensor ===");
  Serial.print("TDS        : "); Serial.print(tdsValue, 1); Serial.println(" ppm");
  Serial.print("Turbidity  : "); Serial.print(ntu, 1); Serial.println(" NTU");
  Serial.print("pH         : "); Serial.println(pH, 2);
  Serial.print("Tinggi Air : "); Serial.print(tinggiAir, 1); Serial.println(" cm");
  Serial.print("Status Air : "); Serial.println(status);
  Serial.println("===========================\n");

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("N:"); lcd.print(ntu, 0);
  lcd.print(" T:"); lcd.print(tdsValue, 0);

  lcd.setCursor(0, 1);
  lcd.print("pH:"); lcd.print(pH, 1);
  lcd.print(" "); lcd.print(status.substring(0,6));

  delay(5000);
}
