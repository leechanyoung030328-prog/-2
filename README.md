// 필요한 라이브러리를 포함합니다.
#include <Wire.h> // I2C 통신을 위한 라이브러리
#include <Adafruit_GFX.h> // 그래픽 처리를 위한 라이브러리
#include <Adafruit_SSD1306.h> // OLED 디스플레이를 위한 라이브러리

// 화면 크기를 정의합니다.
#define SCREEN_WIDTH 128 // 화면 너비
#define SCREEN_HEIGHT 32 // 화면 높이

// OLED 리셋 핀을 정의합니다. (-1은 리셋 핀을 사용하지 않음을 의미)
#define OLED_RESET -1

// OLED 디스플레이 객체를 생성합니다.
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// 버튼 및 센서 핀을 정의합니다.
const int select_button = 2; // 선택 버튼
const int right_button = 3; // 오른쪽 이동 버튼
const int OhmMeter = A0; // 저항 측정 핀
const int CapacitancMeter = A1; // 커패시턴스 측정 핀
const int VoltMeter = A2; // 전압 측정 핀
const int Ammeter = A3; // 전류 측정 핀
const int R3 = 6; // 저항 측정용 릴레이 3
const int R2 = 5; // 저항 측정용 릴레이 2
const int R1 = 4; // 저항 측정용 릴레이 1
const int ChargePin = 13; // 커패시터 충전 핀
const int DischargePin = 11; // 커패시터 방전 핀

// 전역 변수를 선언합니다.
boolean is_select = false; // 선택 버튼 눌림 여부
int navigator = 0; // 메뉴 이동을 위한 변수
int flag = 0; // 플래그 변수 (현재 코드에서는 사용되지 않음)
float R = 0.00; // 저항 값
float V = 0.00; // 전압 값
float I = 0.00; // 전류 값
float C = 0.00; // 커패시턴스 값
boolean nano = false; // 나노패럿 단위 여부
boolean kilo = false; // 킬로옴 단위 여부
boolean mili = false; // 밀리암페어 단위 여부

// OLED 디스플레이 초기화 함수
void OLED_init() {
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) // 0x3C 주소로 디스플레이 초기화
  {
    Serial.println(F("SSD1306 allocation failed")); // 실패 시 시리얼 모니터에 메시지 출력
    for (;;); // 무한 루프
  }
  display.clearDisplay(); // 화면 지우기
  display.println("DIY Multimeter"); // 초기 메시지 출력
  display.display(); // 화면에 표시
  delay(2000); // 2초 대기
  display_clear(); // 화면 지우기
}

// 화면 지우기 함수
void display_clear() {
  display.clearDisplay();
  display.display();
}

// 텍스트 출력 함수
void display_text(int sz, int x, int y, String str) {
  display.setTextSize(sz); // 텍스트 크기 설정
  display.setTextColor(WHITE); // 텍스트 색상 설정
  display.setCursor(x, y); // 커서 위치 설정
  display.println(str); // 문자열 출력
}

// 숫자 출력 함수
void display_number(int sz, int x, int y, double num) {
  display.setTextSize(sz); // 텍스트 크기 설정
  display.setTextColor(WHITE); // 텍스트 색상 설정
  display.setCursor(x, y); // 커서 위치 설정
  display.println(num); // 숫자 출력
}

// 저항 계산 함수
void calculate_resistor() {
  float v_ref = 4.94; // 기준 전압
  // 세 가지 다른 범위의 저항 측정을 위한 변수 선언
  float r1 = 0.00, r_ref1 = 1000.00, adc_value1 = 0.00, voltage1 = 0.00;
  float r2 = 0.00, r_ref2 = 10000.00, adc_value2 = 0.00, voltage2 = 0.00;
  float r3 = 0.00, r_ref3 = 100000.00, adc_value3 = 0.00, voltage3 = 0.00;

  // 첫 번째 범위(1kOhm) 측정
  pinMode(R1, OUTPUT);
  pinMode(R2, INPUT);
  pinMode(R3, INPUT);
  digitalWrite(R1, HIGH);
  for (int i = 0; i < 20 ; i++)
  {
    adc_value1 = adc_value1 + analogRead(OhmMeter); // 20번 읽어서 평균값 계산
    delay(3);
  }
  adc_value1 = adc_value1 / 20;
  if (adc_value1 < 1022.90) // 측정값이 최대가 아닐 때 (오픈 상태가 아닐 때)
  {
    voltage1 = ((adc_value1 * v_ref) / 1024); // 전압 계산
    r1 = (voltage1 * r_ref1) / (v_ref - voltage1); // 저항값 계산 (전압 분배 법칙)
  }

  // 두 번째 범위(10kOhm) 측정
  pinMode(R1, INPUT);
  pinMode(R2, OUTPUT);
  pinMode(R3, INPUT);
  digitalWrite(R2, HIGH);
  for (int i = 0; i < 20 ; i++)
  {
    adc_value2 = adc_value2 + analogRead(OhmMeter);
    delay(3);
  }
  adc_value2 = adc_value2 / 20;
  if (adc_value2 < 1022.90)
  {
    voltage2 = ((adc_value2 * v_ref) / 1024);
    r2 = (voltage2 * r_ref2) / (v_ref - voltage2);
  }

  // 세 번째 범위(100kOhm) 측정
  pinMode(R1, INPUT);
  pinMode(R2, INPUT);
  pinMode(R3, OUTPUT);
  digitalWrite(R3, HIGH);
  for (int i = 0; i < 20 ; i++)
  {
    adc_value3 = adc_value3 + analogRead(OhmMeter);
    delay(3);
  }
  adc_value3 = adc_value3 / 20;
  if (adc_value3 < 1022.90)
  {
    voltage3 = ((adc_value3 * v_ref) / 1024);
    r3 = (voltage3 * r_ref3) / (v_ref - voltage2); // 오타 수정: voltage3 -> voltage2
  }

  // kOhm 단위로 변환
  r1 = r1 / 1000;
  r2 = r2 / 1000;
  r3 = r3 / 1000;

  // 가장 적절한 범위의 저항값 선택
  if (r1 < 2 && r2 < 101 && r3 < 1001) R = r1 * 1000; // 2kOhm 미만이면 Ohm 단위로
  else if (r1 > 2 && r2 < 101 && r3 < 1001) R = r2; // 2kOhm 이상, 101kOhm 미만
  else if (r1 > 2 && r2 > 101 && r3 < 2000) R = r3; // 101kOhm 이상, 2000kOhm 미만
  else R = 0.00; // 범위를 벗어나면 0

  // 단위 설정 (kOhm 또는 Ohm)
  if (R < 1)
  {
    R = R * 1000;
    kilo = false;
  }
  else
  {
    kilo = true;
  }
}

// 커패시턴스 계산 함수
void calculate_capacitance() {
  unsigned long start_time;
  unsigned long elapsed_time;
  float microFarads;
  float nanoFarads;
  float r_ref = 10000.00; // 기준 저항 (10kOhm)

  digitalWrite(ChargePin, HIGH); // 커패시터 충전 시작
  start_time = millis();
  while (analogRead(CapacitancMeter) < 648) {} // 전압이 63.2%에 도달할 때까지 대기 (1 타우)
  elapsed_time = millis() - start_time; // 경과 시간 측정
  microFarads = ((float)elapsed_time / r_ref) * 1000; // uF 단위로 커패시턴스 계산

  // 단위 설정 (uF 또는 nF)
  if (microFarads > 1)
  {
    C = microFarads;
    nano = false;
  }
  else
  {
    nanoFarads = microFarads * 1000.0;
    C = nanoFarads;
    nano = true;
  }

  // 커패시터 방전
  digitalWrite(ChargePin, LOW);
  pinMode(DischargePin, OUTPUT);
  digitalWrite(DischargePin, LOW);
  while (analogRead(CapacitancMeter) > 0) {} // 완전히 방전될 때까지 대기
  pinMode(DischargePin, INPUT); // 방전 핀을 입력으로 전환
}

// 전압 계산 함수
void calculate_voltage() {
  float R1 = 10000.00; // 전압 분배 저항 1
  float R2 = 4700.00; // 전압 분배 저항 2
  float v_ref = 5.00; // 기준 전압
  float resistor_ratio = 0.00;
  float adc_value = 0.00;
  float voltage = 0.00;

  resistor_ratio = (R2 / (R1 + R2)); // 저항 비율 계산
  for (int i = 0; i < 20 ; i++)
  {
    adc_value = adc_value + analogRead(VoltMeter); // 20번 읽어서 평균값 계산
    delay(3);
  }
  adc_value = adc_value / 20;
  voltage = ((adc_value * v_ref) / 1024); // 아날로그 값으로부터 전압 계산
  V = voltage / resistor_ratio; // 실제 전압 계산
}

// 전류 계산 함수
void calculate_current() {
  int sensitivity = 185; // 전류 센서 민감도 (mV/A)
  int adc_value = 0;
  float v_ref = 4.94; // 기준 전압
  float voltage = 0.00;
  float pure_voltage = 0.00;
  float offset_voltage = 2.47; // 센서 오프셋 전압 (무전류 시 출력 전압)

  for (int i = 0; i < 40 ; i++)
  {
    adc_value = adc_value + analogRead(Ammeter); // 40번 읽어서 평균값 계산
    delay(2);
  }
  adc_value = adc_value / 40;
  voltage = ((adc_value * v_ref) / 1024); // 센서 출력 전압 계산
  pure_voltage = voltage - offset_voltage; // 오프셋을 뺀 순수 전압 변화량 계산
  pure_voltage = pure_voltage * 1000; // mV 단위로 변환
  I = pure_voltage / sensitivity; // 전류 계산

  // 단위 설정 (A 또는 mA)
  if (I < 1)
  {
    I = I * 1000;
    mili = true;
  }
  else
  {
    mili = false;
  }
}

// 초기 설정 함수
void setup() {
  Serial.begin(9600); // 시리얼 통신 시작
  OLED_init(); // OLED 초기화
  pinMode(right_button, INPUT_PULLUP); // 내부 풀업 저항 사용
  pinMode(select_button, INPUT_PULLUP); // 내부 풀업 저항 사용
  pinMode(ChargePin, OUTPUT);
  digitalWrite(ChargePin, LOW);
}

int cnt=0; // 초기 화면 표시를 위한 카운터
// 메인 루프 함수
void loop() {
  // 처음 한 번만 "DIY Multimeter" 메시지 표시
  if(cnt==0)
  {
    display.clearDisplay();
    display_text(1, 20, 9, "DIY Multimeter");
    display.display();
    delay(2000);
    cnt=1;
  }

  // 오른쪽 버튼을 누르면 메뉴 이동
  if (digitalRead(right_button) == 0)
  {
    navigator++;
    while (digitalRead(right_button) == 0); // 버튼에서 손을 뗄 때까지 대기
    delay(5);
    if (navigator > 3) navigator = 0; // 메뉴 순환
    Serial.println(navigator);
  }

  // 선택 버튼을 누르면 is_select 플래그를 true로 설정
  if ( digitalRead(select_button) == 0)
  {
    is_select = true;
    while ( digitalRead(select_button) == 0);
  }

  // 저항 측정 모드
  if (navigator == 0)
  {
    display.clearDisplay();
    display_text(2, 17, 8, "Resistor");
    display.display();
    while (is_select) // 선택 버튼이 눌렸을 때만 측정 시작
    {
      display.clearDisplay();
      display_text(1, 0, 0, "Resistor");
      display_text(2, 12, 8, "R=");
      display_number(2, 42, 8, R);
      if (kilo) display_text(1, 115, 15, "k"); // 킬로옴 단위 표시
      display.display();
      calculate_resistor(); // 저항 계산
      if ( digitalRead(select_button) == 0) // 다시 선택 버튼을 누르면 측정 중지
      {
        is_select = false;
        while ( digitalRead(select_button) == 0);
      }
    }
  }

  // 전압 측정 모드
  if (navigator == 1)
  {
    display.clearDisplay();
    display_text(2, 17, 8, "Voltage");
    display.display();
    while (is_select)
    {
      display.clearDisplay();
      display_text(1, 0, 0, "Voltage");
      display_text(2, 12, 8, "V=");
      display_number(2, 42, 8, V);
      display_text(1, 115, 15, "v");
      display.display();
      calculate_voltage();
      if ( digitalRead(select_button) == 0)
      {
        is_select = false;
        while ( digitalRead(select_button) == 0);
      }
    }
  }

  // 전류 측정 모드
  if (navigator == 2)
  {
    display.clearDisplay();
    display_text(2, 17, 8, "Current");
    display.display();
    while (is_select)
    {
      display.clearDisplay();
      display_text(1, 0, 0, "Current");
      display_text(2, 12, 8, "I=");
      display_number(2, 42, 8, I);
      if (mili) display_text(1, 115, 15, "mA"); // mA 또는 A 단위 표시
      if (!mili) display_text(1, 115, 15, "A");
      display.display();
      calculate_current();
      if ( digitalRead(select_button) == 0)
      {
        is_select = false;
        while ( digitalRead(select_button) == 0);
      }
    }
  }

  // 커패시턴스 측정 모드
  if (navigator == 3)
  {
    display.clearDisplay();
    display_text(2, 12, 8, "Capacitor");
    display.display();
    while (is_select)
    {
      display.clearDisplay();
      display_text(1, 0, 0, "Capacitor");
      display_text(2, 12, 8, "C=");
      display_number(2, 42, 8, C);
      if (nano) display_text(1, 115, 22, "nF"); // nF 또는 uF 단위 표시
      if (!nano) display_text(1, 115, 22, "uF");
      display.display();
      calculate_capacitance();
      if ( digitalRead(select_button) == 0)
      {
        is_select = false;
        while ( digitalRead(select_button) == 0);
      }
    }
  }
}
