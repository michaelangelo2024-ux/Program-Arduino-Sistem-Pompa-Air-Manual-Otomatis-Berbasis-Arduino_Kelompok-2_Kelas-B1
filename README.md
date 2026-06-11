#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ======================
// LCD I2C
// ======================
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ======================
// PIN
// ======================
#define TRIG_PIN    2
#define ECHO_PIN    3

#define LED_MERAH   A0
#define LED_KUNING  A2
#define LED_HIJAU   A3

#define RELAY_PIN   7

// ======================
// RELAY ACTIVE LOW
// ======================
#define RELAY_ON    LOW
#define RELAY_OFF   HIGH

// ======================
// KALIBRASI TANGKI
// ======================

// Tinggi tangki dari sensor sampai dasar
const float tinggiTangki = 13.0;

// Saat penuh HC-SR04 biasanya masih membaca 2-3 cm
const float jarakPenuh = 3.0;

// ======================
// BATAS POMPA (%)
// ======================
const int persenON  = 0;  // nyala saat <= 20%
const int persenOFF = 100;  // mati saat >= 90%

bool pompaAktif = false;

// ======================
// BACA SENSOR RAW
// ======================
float bacaJarakRaw()
{
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(5);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);

  digitalWrite(TRIG_PIN, LOW);

  long durasi = pulseIn(ECHO_PIN, HIGH, 30000);

  if (durasi == 0)
    return -1;

  return durasi * 0.0343 / 2.0;
}

// ======================
// FILTER DATA
// ======================
float bacaJarak()
{
  const int jumlahSampel = 10;

  float data[jumlahSampel];
  int valid = 0;

  for (int i = 0; i < jumlahSampel; i++)
  {
    float jarak = bacaJarakRaw();

    if (jarak > 0 && jarak < 400)
    {
      data[valid++] = jarak;
    }

    delay(20);
  }

  if (valid < 3)
    return -1;

  // sorting
  for (int i = 0; i < valid - 1; i++)
  {
    for (int j = i + 1; j < valid; j++)
    {
      if (data[j] < data[i])
      {
        float temp = data[i];
        data[i] = data[j];
        data[j] = temp;
      }
    }
  }

  float total = 0;

  for (int i = 1; i < valid - 1; i++)
  {
    total += data[i];
  }

  return total / (valid - 2);
}

// ======================
// LED KUNING BERKEDIP
// ======================
void kedipKuning()
{
  for (int i = 0; i < 3; i++)
  {
    digitalWrite(LED_KUNING, HIGH);
    delay(200);

    digitalWrite(LED_KUNING, LOW);
    delay(200);
  }
}

// ======================
// SETUP
// ======================
void setup()
{
  Serial.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(LED_MERAH, OUTPUT);
  pinMode(LED_KUNING, OUTPUT);
  pinMode(LED_HIJAU, OUTPUT);

  pinMode(RELAY_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, RELAY_OFF);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Water Level");
  lcd.setCursor(0, 1);
  lcd.print("System Ready");

  delay(2000);
  lcd.clear();
}

// ======================
// LOOP
// ======================
void loop()
{
  float jarak = bacaJarak();

  if (jarak < 0)
  {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error   ");
    lcd.setCursor(0, 1);
    lcd.print("Cek HC-SR04    ");

    digitalWrite(RELAY_PIN, RELAY_OFF);
    return;
  }

  if (jarak > tinggiTangki)
    jarak = tinggiTangki;

  // ======================
  // HITUNG PERSEN AIR
  // ======================
  float persen;

  if (jarak <= jarakPenuh)
  {
    persen = 100;
  }
  else
  {
    persen =
      ((tinggiTangki - jarak) /
      (tinggiTangki - jarakPenuh)) * 100.0;
  }

  if (persen < 0) persen = 0;
  if (persen > 100) persen = 100;

  // ======================
  // LOGIKA POMPA
  // ======================
  if (!pompaAktif && persen <= persenON)
  {
    kedipKuning();

    pompaAktif = true;
    digitalWrite(RELAY_PIN, RELAY_ON);
  }

  if (pompaAktif && persen >= persenOFF)
  {
    kedipKuning();

    pompaAktif = false;
    digitalWrite(RELAY_PIN, RELAY_OFF);
  }

  // ======================
  // LED STATUS
  // ======================
  if (pompaAktif)
  {
    digitalWrite(LED_HIJAU, HIGH);
    digitalWrite(LED_MERAH, LOW);
    digitalWrite(LED_KUNING, LOW);
  }
  else
  {
    digitalWrite(LED_HIJAU, LOW);
    digitalWrite(LED_MERAH, HIGH);
    digitalWrite(LED_KUNING, LOW);
  }

  // ======================
  // SERIAL MONITOR
  // ======================
  Serial.print("Jarak: ");
  Serial.print(jarak);
  Serial.print(" cm | Air: ");
  Serial.print((int)persen);
  Serial.print("% | Pompa: ");

  if (pompaAktif)
    Serial.println("ON");
  else
    Serial.println("OFF");

  // ======================
  // LCD
  // ======================
  lcd.setCursor(0, 0);
  lcd.print("Air Tandon: ");
  lcd.print((int)persen);
  lcd.print("%     ");

  lcd.setCursor(0, 1);

  if (pompaAktif)
    lcd.print("S.Pompa   : ON      ");
  else
    lcd.print("S.Pompa   : OFF     ");

  delay(500);
}
