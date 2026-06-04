#include <Arduino.h>
#include <avr/wdt.h>

#include "src/gpio/main.h"
#include "src/utils/main.h"
#include "src/i2c/main.h"
#include "src/oled/main.h"
#include "src/oled/graphics/main.h"
#include "src/uart/main.h"
#include "src/gsm/main.h"
#include "src/gsm/message/main.h"

#ifndef oledpinconfig
#define oledpinconfig
#define oledheight  32
#define oledwidth   128
#define oledclock   AN5
#define oleddata    AN4
#define oledaddress 0x78
#define oledfreq    100
#endif 

#ifndef alarmpinconfig
#define alarmpinconfig
#define alarm 11
#endif 

#ifndef radarpinconfig
#define radarpinconfig
#define radartrigger A0 
#define radarecho    A1
#endif 

#ifndef gsmpinconfig
#define gsmpinconfig
#define gsmtransmitter  1
#define gsmreceiver     0
#define gsmbaudrate     115200
#endif 

#ifndef userthreshconfig
#define userthreshconfig
#define binthresh     90  //level
#define mobilenumber  "8807699670"
#endif 

uint16_t distance;
uint8_t level;
bool isbinfilled;

uint16_t getdistance(void)
{
  uint32_t pulseduration = 0;
  digitalWrite(radartrigger, HIGH);
  delayMicroseconds(2);
  digitalWrite(radartrigger, LOW);
  delayMicroseconds(10);
  pulseduration = pulseIn(radarecho, HIGH, 25000);
  return (uint16_t)(pulseduration * 0.034F / 2);
}

void setup()
{
  gpio_output(alarm);
  gpio_low(alarm);

  pinMode(radartrigger, OUTPUT);
  pinMode(radarecho, INPUT);

  oled_initialize(oledfreq, oledheight, oledwidth);
  oled_print(" SMART DUST BIN ");
  oled_print(" WITH AUTO FILL ");
  oled_print("MESSAGE ALERTING");
  oled_print(" USING ARDUINO  ");
  oled_display(); delay_ms(2500);
  oled_fill_screen(black);

  gsm_initialize(gsmbaudrate);
  gsm_echo(false); gsm_found();
  gsm_msg_enable(true);

  oled_set_cursor(0, 0);
  if(gsm_found()) oled_print("GSM FOUND");
  else oled_print("GSM NOT FOUND");
  oled_display(); delay_ms(500);

  oled_set_cursor(0, 8);
  if(gsm_sim_status()) oled_print("SIM FOUND");
  else oled_print("SIM NOT FOUND");
  oled_display(); delay_ms(500);

  oled_set_cursor(0, 16);
  if(gsm_tower_wait()) oled_print("SIM TOWERED");
  else oled_print("N/W ERROR");
  oled_display(); delay_ms(500);

  oled_set_cursor(0, 24);
  if(gsm_msg_enable(true)) oled_print("MSG READY");
  else oled_print("MSG ERROR"); 
  oled_display(); delay_ms(500);

  serial_flush();
  oled_fill_screen(black);
  wdt_enable(WDTO_2S);
}

void loop()
{
  distance = getdistance();
  if(distance > 18) distance = 18;
  if(distance) level = map(distance, 0, 18, 100, 0);
  wdt_reset();

  oled_set_tsize(1, 2); oled_set_color(black);
  oled_fill_roundrect(0, 0, 100, 15, 2); 
  oled_set_color(white);
  oled_draw_roundrect(0, 0, 100, 15, 2); 
  oled_fill_roundrect(0, 0, level, 15, 2);

  oled_set_cursor(104, 0);
  oled_decimal(level, 2, DEC); oled_write('%');
  oled_set_cursor(0, 16); oled_set_tsize(1, 1);
  oled_print("LEVEL:"); 
  oled_decimal(distance, 3, DEC); oled_print("CM ");
  oled_decimal(level, 3, DEC); oled_write('%');

  oled_set_cursor(0, 24);
  if(isbinfilled) oled_print("BIN FILLED!!!");
  else oled_print("             ");
  oled_display();

  if(level > binthresh && !isbinfilled)
  {
    gpio_high(alarm);
    oled_set_cursor(0, 24);
    oled_print("SEND MESSAGE!");
    oled_display();

    wdt_disable(); gsm_msg_start(mobilenumber);
    serial_write("Bin(A471) filled replace ASAP...");
    serial_write("\r\nFilled: "); serial_decimal(level, DEC); serial_send('%');
    serial_write("\r\nThresh: "); serial_decimal(binthresh, DEC); serial_send('%');
    gsm_msg_end(); serial_flush(); wdt_enable(WDTO_2S);

    oled_set_cursor(0, 24);
    oled_print("MESSAGE SENT!");
    oled_display(); delay_ms(500);
    oled_fill_screen(black);
    isbinfilled = true;
  }
  else if(level < binthresh && isbinfilled)
  {
    gpio_low(alarm);
    oled_set_cursor(0, 24);
    oled_print("SEND MESSAGE!");
    oled_display();

    wdt_disable();
    gsm_msg_send(mobilenumber, "Bin(A471) is now cleared...");
    serial_flush(); wdt_enable(WDTO_2S);

    oled_set_cursor(0, 24);
    oled_print("MESSAGE SENT!");
    oled_display(); delay_ms(500);
    oled_fill_screen(black);
    isbinfilled = false; 
  }
}
