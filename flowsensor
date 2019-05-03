#include <Wire.h> // Wire Bibliothek einbinden
#include <LiquidCrystal_I2C.h> // Vorher hinzugefügte LiquidCrystal_I2C Bibliothek einbinden

#include <EEPROM.h>

#define FLOWSENSOR 2
#define MOTOR_PWM 5
#define MOTOR_DIRECTION 6
#define PRES_INP A0
#define BATT A1

#define PRES_MIN 20
#define PRES_SCNT 20 //was 50, testing....
#define PRES_DELAY 5000
#define PRES_BUFCNT 30
#define PRES_BUFTIME 50

#define MODE_IDLE 0
#define MODE_CAL 1
#define MODE_FILL 2
#define MODE_REVERSE 3
#define MODE_FULL 4
#define MODE_EMPTY 5
#define MODE_ERROR 6
#define MODE_LIMIT 7
#define MODE_BATT 8

String modeText[] = { "Idle ",
                      "Cali ",
                      "Fill ",
                      "Revr ",
                      "Full ",
                      "Empt ",
                      "Erro ",
                      "Limi ",
                      "Batt "
                    };

#define MENU_START 0
#define MENU_MAX 1
#define MENU_PWR 2
#define MENU_TRIG 3
#define MENU_TRIGDELAY 4
#define MENU_REVERSE 5
#define MENU_FLOW 6
#define MENU_BATT 7
#define MENU_PRES 8

#define MENU_FIRST MENU_START
#define MENU_LAST MENU_PRES

#define P_BUFFER_SIZE 20

int pres_offset = 0;
int pres_buffer[PRES_BUFCNT];
long pres_buf_time = 0;
int mode = MODE_IDLE;
long tank_start = 0;
int count = 0;
int count_raw = 0;

int menu_max;
int menu_trig;
int menu_reverse;
int menu_flow;
int menu_batt;
int menu_pwr;
int menu_trigdelay;
int menu_mode = MENU_START;

long disp_prevdraw = millis();

int p_index = 0;
int p_buffer[P_BUFFER_SIZE];

long btn_last = 0;
long btn_lastupdate = 0;
int btn_delay = 0;

LiquidCrystal_I2C lcd(0x27, 16, 2); //Hier wird festgelegt um was für einen Display es sich handelt

void setup()
{
  pinMode(MOTOR_PWM, OUTPUT);
  pinMode(MOTOR_DIRECTION, OUTPUT);
  setMotor();

  //lcd.begin(16, 2);
  //lcd.clear();
  lcd.init(); //Im Setup wird der LCD gestartet
  lcd.backlight(); //Hintergrundbeleuchtung einschalten (lcd.noBacklight(); schaltet die Beleuchtung aus).
  setupCharacters();

  pinMode(BATT, INPUT);
  pinMode(PRES_INP, INPUT);
  pinMode(FLOWSENSOR, INPUT_PULLUP);

  delay(300);

  pres_offset = getPressure(false);
  loadData();

  attachInterrupt(digitalPinToInterrupt(FLOWSENSOR), flowTick, FALLING);
}

int getPressure(bool buffered)
{
  double val = analogRead(PRES_INP);

  for (int i = 0; i < PRES_SCNT; i++)
  {
    val += (analogRead(PRES_INP));
  }

  val = (val / PRES_SCNT) - pres_offset;

  p_buffer[p_index] = val;
  p_index++;

  if (p_index >= P_BUFFER_SIZE)
  {
    p_index = 0;
  }

  double total = 0;

  for (int i = 0; i <= P_BUFFER_SIZE; i++)
  {
    total += p_buffer[i];
  }

  if (buffered)
  {
    return total / P_BUFFER_SIZE;
  }
  else
  {
    return val;
  }
}

double getBatt()
{
  return (double)analogRead(BATT) / 50;
}

void flowTick()
{
  if (mode == MODE_REVERSE || mode == MODE_EMPTY)
  {
    count_raw--;
  }
  else
  {
    count_raw++;
  }
}

void setMotor()
{
  int mPwr = 255 - ((double)menu_pwr * 2.55);

  if (mode == MODE_FILL || mode == MODE_CAL)
  {
    analogWrite(MOTOR_PWM, mPwr);
    digitalWrite(MOTOR_DIRECTION, LOW);
  }
  else if (mode == MODE_EMPTY || mode == MODE_REVERSE)
  {
    analogWrite(MOTOR_PWM, mPwr);
    digitalWrite(MOTOR_DIRECTION, HIGH);
  }
  else
  {
    digitalWrite(MOTOR_PWM, HIGH);
    digitalWrite(MOTOR_DIRECTION, HIGH);
  }
}

void drawFixed(String text, int total)
{
  int printed = lcd.print(text);

  for (int i = printed; i < total; i++)
  {
    lcd.print(" ");
  }
}

void drawLcd(int pressure, double batt, int delta)
{
  lcd.setCursor(0, 0);

  if (menu_mode == MENU_START)
  {
    lcd.print(modeText[mode]);

    lcd.setCursor(5, 0);
    drawFixed(String(count) + "ml", 7);

    lcd.setCursor(12, 0);
    drawFixed(String(menu_pwr) + "%", 4);
  }
  else if (menu_mode == MENU_MAX)
  {
    lcd.print("Limit=");
    if (menu_max > 0)
    {
      drawFixed(String(menu_max) + "ml", 10);
    }
    else
    {
      drawFixed("off", 10);
    }
  }
  else if (menu_mode == MENU_PWR)
  {
    drawFixed("Power=" + String(menu_pwr) + "%", 16);
  }
  else if (menu_mode == MENU_FLOW)
  {
    drawFixed("Flow=" + String((double)menu_flow / 100) + " #" + count_raw, 16);
  }
  else if (menu_mode == MENU_REVERSE)
  {
    lcd.print("Reverse=");
    if (menu_reverse > 0)
    {
      drawFixed(String((double)menu_reverse / 10) + "sec", 8);
    }
    else
    {
      drawFixed("off", 8);
    }
  }
  else if (menu_mode == MENU_TRIG)
  {
    lcd.print("Trigger=");
    if (menu_trig > 0)
    {
      drawFixed(String(menu_trig) + "^", 8);
    }
    else
    {
      drawFixed("off", 8);
    }
  }
  else if (menu_mode == MENU_TRIGDELAY)
  {
    lcd.print("TrigDelay=");
    if (menu_trigdelay > 0)
    {
      drawFixed(String(menu_trigdelay) + "ml", 6);
    }
    else
    {
      drawFixed("off", 6);
    }
  }
  else if (menu_mode == MENU_BATT)
  {
    drawFixed("BattLow=" + String((double)menu_batt / 10) + "V", 16);
  }
  else if (menu_mode == MENU_PRES)
  {
    drawFixed("Pressure " + String(pressure) + "/" + pres_offset, 16);
  }

  lcd.setCursor(0, 1);

  if (mode != MODE_IDLE && menu_max > 0)
  {
    int fill = abs(((double)count / (double)menu_max) * 16);
    for (int i = 0; i < fill; i++)
    {
      lcd.print("#");
    }
    for (int i = fill; i < 16; i++)
    {
      lcd.print("_");
    }
  }
  else if (mode != MODE_IDLE && menu_trig > 0)
  {
    int lowest = 0;
    for (int i = 0; i < PRES_BUFCNT - 16; i++)
    {
      lowest += pres_buffer[i];
    }

    lowest = lowest / (PRES_BUFCNT - 16);

    for (int i = PRES_BUFCNT - 16; i < PRES_BUFCNT; i++)
    {
      int diff = pres_buffer[i] - lowest;
      diff = 4 + ((double)diff / (double)menu_trig * 4);

      if (diff > 8) diff = 8;
      else if (diff < 0) diff = 0;

      if (diff == 0)
      {
        lcd.print(" ");
      }
      else
      {
        lcd.write(diff - 1);
      }
    }
  }
  else
  {
    drawFixed("Battery " + String(batt) + "V", 16);
  }
}

void readBtn()
{
  // digitalWrite(8, HIGH);

  pinMode(11, INPUT);
  pinMode(9, INPUT);
  pinMode(12, INPUT);

  //delay(4);

  bool btn1 = !digitalRead(11);
  bool btn2 = !digitalRead(9);
  bool btn3 = !digitalRead(12);

  if (btn1 && btn2 && !btn3)
  {
    //btn 4
    btnUp();
  }
  else if (!btn1 && btn2 && !btn3)
  {
    //btn 2
    btnEnt();
  }
  else if (btn1 && !btn2 && !btn3)
  {
    //btn 1
    btnDown();
  }
  else if (!btn1 && !btn2 && btn3)
  {
    //btn 3
    btnEnt2();
  }

  pinMode(11, OUTPUT);
  pinMode(9, OUTPUT);
  pinMode(12, OUTPUT);

  digitalWrite(8, LOW);

  //Re init
  lcd.print(" ");
}

void checkMode(double batt, int delta)
{
  if (mode == MODE_IDLE)
  {
    tank_start = 0;
  }
  else if (mode != MODE_IDLE && batt < ((double)menu_batt / 10))
  {
    mode = MODE_BATT;
  }
  else if (mode == MODE_CAL)
  {
    if (tank_start == 0)
    {
      tank_start = millis();
    }
    else if (millis() > (tank_start + PRES_DELAY ))
    {
      mode = MODE_FILL;
    }
  }
  else if (mode == MODE_FILL)
  {
    if (count > menu_trigdelay && menu_trig > 0 && delta > menu_trig)
    {
      if (menu_reverse > 0)
      {
        //Allow H-bridge to idle
        mode = MODE_IDLE;
        setMotor();
        delay(100);

        mode = MODE_REVERSE;
        tank_start = millis();
      }
      else
      {
        mode = MODE_FULL;
      }
    }
    else if (menu_max > 0 && count >= menu_max)
    {
      mode = MODE_LIMIT;
    }
  }
  else if (mode == MODE_REVERSE)
  {
    if (millis() > tank_start + (menu_reverse * 100))
    {
      mode = MODE_FULL;
    }
  }
}

long lastDraw = 0;

void loop()
{
  int pressure = getPressure(true);

  if (millis() > pres_buf_time + PRES_BUFTIME)
  {
    pres_buf_time = millis();
    for (int i = 0; i < PRES_BUFCNT - 1; i++)
    {
      pres_buffer[i] = pres_buffer[i + 1];
    }

    pres_buffer[PRES_BUFCNT - 1] = pressure;
  }

  int delta = pres_buffer[PRES_BUFCNT - 1] - pres_buffer[0];
  double batt = getBatt();

  count = (double)count_raw / (menu_flow / (double)100);

  int prevMode = mode;
  int prevPwr = menu_pwr;
  checkMode(batt, delta);
  readBtn();

  if (mode != prevMode || prevPwr != menu_pwr)
  {
    setMotor();
  }

  lastDraw = millis();
  drawLcd(pressure, batt, delta);
}

bool doBtn(bool repeat)
{
  long diff = millis() - btn_last;
  btn_last = millis();

  if (diff > 100)
  {
    btn_delay = 300;
    btn_lastupdate = millis();
    return true;
  }

  if (!repeat)
  {
    return false;
  }

  if (millis() - btn_lastupdate > btn_delay)
  {
    if (btn_delay > 150)
    {
      btn_delay = 150;
    }

    if (btn_delay > 20)
    {
      btn_delay = btn_delay - 5;
    }

    if (btn_delay > 0)
    {
      btn_delay = btn_delay - 1;
    }

    btn_lastupdate = millis();
    return true;
  }

  return false;
}

void btnUp()
{
  if (doBtn(true))
  {
    if (mode == MODE_EMPTY)
    {
      menu_pwr++;

      if (menu_pwr > 100)
      {
        menu_pwr = 100;
      }

      saveData();
    }
    else if (mode == MODE_FILL || mode == MODE_CAL)
    {
      menu_pwr++;

      if (menu_pwr > 100)
      {
        menu_pwr = 100;
      }

      mode = MODE_CAL;

      tank_start = 0;
      //pres_tank = 0;

      saveData();
    }
    else if (mode == MODE_FULL)
    {
      mode = MODE_CAL;
      tank_start = 0;
    }
    else if (menu_mode == MENU_START)
    {
      if (mode == MODE_IDLE)
      {
        count_raw = 0;
        mode = MODE_CAL;
      }
    }
    else if (menu_mode == MENU_MAX)
    {
      if (btn_delay == 0)
      {
        menu_max += 10;
      }
      else
      {
        menu_max++;
      }
      saveData();
    }
    else if (menu_mode == MENU_TRIG)
    {
      menu_trig++;
      saveData();
    }
    else if (menu_mode == MENU_TRIGDELAY)
    {
      if (btn_delay == 0)
      {
        menu_trigdelay += 10;
      }
      else
      {
        menu_trigdelay++;
      }
      saveData();
    }
    else if (menu_mode == MENU_REVERSE)
    {
      menu_reverse++;
      saveData();
    }
    else if (menu_mode == MENU_FLOW)
    {
      menu_flow++;
      saveData();
    }
    else if (menu_mode == MENU_PRES && mode == MODE_IDLE)
    {
      pres_offset = 0;
      pres_offset = getPressure(false);
    }
    else if (menu_mode == MENU_BATT)
    {
      menu_batt++;

      if (menu_batt > 126)
      {
        menu_batt = 126;
      }

      saveData();
    }
    else if (menu_mode == MENU_PWR)
    {
      menu_pwr++;

      if (menu_pwr > 100)
      {
        menu_pwr = 100;
      }

      saveData();
    }
  }
}

void btnDown()
{
  if (doBtn(true))
  {
    if (mode == MODE_EMPTY || mode == MODE_FILL || mode == MODE_CAL)
    {
      menu_pwr--;

      if (menu_pwr < 0)
      {
        menu_pwr = 0;
      }

      saveData();
    }
    else if (menu_mode == MENU_START)
    {
      if (mode == MODE_IDLE)
      {
        count_raw = 0;
        mode = MODE_EMPTY;
      }
    }
    else if (menu_mode == MENU_MAX)
    {
      if (btn_delay == 0)
      {
        menu_max -= 10;
      }
      else
      {
        menu_max--;
      }

      if (menu_max < 0)
      {
        menu_max = 0;
      }

      saveData();
    }
    else if (menu_mode == MENU_FLOW)
    {
      menu_flow--;

      if (menu_flow < 1)
      {
        menu_flow = 1;
      }

      saveData();
    }
    else if (menu_mode == MENU_REVERSE)
    {
      menu_reverse--;

      if (menu_reverse < 0)
      {
        menu_reverse = 0;
      }

      saveData();
    }
    else if (menu_mode == MENU_TRIG)
    {
      menu_trig--;

      if (menu_trig < 0)
      {
        menu_trig = 0;
      }

      saveData();
    }
    else if (menu_mode == MENU_TRIGDELAY)
    {
      if (btn_delay == 0)
      {
        menu_trigdelay -= 10;
      }
      else
      {
        menu_trigdelay--;
      }

      if (menu_trigdelay < 0)
      {
        menu_trigdelay = 0;
      }

      saveData();
    }
    else if (menu_mode == MENU_BATT)
    {
      menu_batt--;

      if (menu_batt < 90)
      {
        menu_batt = 90;
      }

      saveData();
    }
    else if (menu_mode == MENU_PWR)
    {
      menu_pwr--;

      if (menu_pwr < 0)
      {
        menu_pwr = 0;
      }

      saveData();
    }
  }
}

void btnEnt()
{
  if (doBtn(false))
  {
    if (mode == MODE_IDLE)
    {
      menu_mode++;

      if (menu_mode > MENU_LAST)
      {
        menu_mode = MENU_FIRST;
      }
    }
    else
    {
      mode = MODE_IDLE;
    }
  }
}

void btnEnt2()
{
  if (doBtn(false))
  {
    if (mode == MODE_IDLE)
    {
      menu_mode = MENU_START;
    }
    else
    {
      mode = MODE_IDLE;
    }
  }
}

void saveData()
{
  writeEEPROMValue(0, menu_max);
  writeEEPROMValue(1, menu_trig);
  writeEEPROMValue(2, menu_reverse);
  writeEEPROMValue(3, menu_flow);
  writeEEPROMValue(4, menu_batt);
  writeEEPROMValue(5, menu_pwr);
  writeEEPROMValue(6, menu_trigdelay);
}

void writeEEPROMValue(int pos, int value)
{
  EEPROM.write(pos * 2, lowByte(value));
  EEPROM.write((pos * 2) + 1, highByte(value));
}

int readEEPROMValue(int pos, int value)
{
  if (EEPROM.read(pos * 2) == 255 && EEPROM.read((pos * 2) + 1) == 255)
  {
    return value;
  }
  else
  {
    return EEPROM.read(pos * 2) + (EEPROM.read((pos * 2) + 1) * 256);
  }
}

void loadData()
{
  menu_max = readEEPROMValue(0, 0);
  menu_trig = readEEPROMValue(1, 5);
  menu_reverse = readEEPROMValue(2, 30);
  menu_flow = readEEPROMValue(3, 570);
  menu_batt = readEEPROMValue(4, 105);
  menu_pwr = readEEPROMValue(5, 100);
  menu_trigdelay = readEEPROMValue(6, 0);
}

void setupCharacters()
{
  lcd.createChar(0, new byte[8] {
    B00000,
    B00000,
    B00000,
    B00000,
    B00000,
    B00000,
    B00000,
    B11111,
  });
  lcd.createChar(1, new byte[8] {
    B00000,
    B00000,
    B00000,
    B00000,
    B00000,
    B00000,
    B11111,
    B11111,
  });
  lcd.createChar(2, new byte[8] {
    B00000,
    B00000,
    B00000,
    B00000,
    B00000,
    B11111,
    B11111,
    B11111,
  });
  lcd.createChar(3, new byte[8] {
    B00000,
    B00000,
    B00000,
    B00000,
    B11111,
    B11111,
    B11111,
    B11111,
  });
  lcd.createChar(4, new byte[8] {
    B00000,
    B00000,
    B00000,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
  });
  lcd.createChar(5, new byte[8] {
    B00000,
    B00000,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
  });
  lcd.createChar(6, new byte[8] {
    B00000,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
  });
  lcd.createChar(7, new byte[8] {
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
  });
}
