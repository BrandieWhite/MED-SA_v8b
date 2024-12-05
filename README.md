/*
                 Eight Channel Data Recorder  [  ECDR]

  The unit default is 8 [CH0 thru 7] differential inputs.
  Each input is sampled by an AD7124 [ 8ch 24bit adc]

 Operation:  ....3 MODES   mode1=Recorder    mode2=SD card playback  mode3=SD erase
  Mode select: 
      Mode1 is set by external switch sw#1=>on sw#2=>off sw#3=>off sw#8=>off=>on->off
      Mode2 is set by external switch sw#1=>off sw#2=>on sw#3=>off sw#8=>off=>on->off
      Mode3 is set by external switch sw#1=>off sw#2=>off sw#3=>on sw#8=>off=>on->off
  Mode1: RECORD   
  1. Turn power on [USB or Battery]
  2. Set MODE select for RECORD
  3. Observe blilnking Blue LED...--> one each log entry  per "Blink".
  4. Seal unit ready for deployment
  Data record= Timestamp per sample set, [ch ID, chdata] x 8  end of record
  **********************************
  5.  When complete with experiment, turm power switch off..

  Mode2:  PLAYBACK  
  1. Turn power on 
  2. Set MODE select for PLAYBACK
  3. Observe constant Blue LED...- 
  4. Output via USB , ASCII data output
  Data Playback= File names and sizes on SD, followed by recorded data
  **********************************

  Mode3:    ERASE/CLEAR
  1. Turn power on 
  2. Set MODE select for ERASE/CLEAR
  3. Observe constant Blue LED...- 
  4. Output via USB , ASCII data output
  Data ERASE= erase old ECDR_log.txt and seed a new EDCR_log.txt then
      show File names and sizes on SD  [verify old is gone and new is ready]
  *********************************
   
   Notes:
  a. if blinking RED => error in operation or setup, not ready for deployment.
  b. all Ch's are defined seperatly to allow for different Vsupply  
  c. PWR[on] times are adjustiable [1us to 'on' continous] to accomodate different sensors
  d. All channels can be configured for different Vref, Gain, and Offsets 
  e. Max file size is 2GB for current settings
  
        _______________      MODE and RESET switch
       |1|2|3|4|5|6|7|8|                  Mode
       |#|#|#|#|#|#|#|#| <--  "on"             Record  set 1->on,  2->off, 3-off , 8-on & 8-off toggle
       |#|#|#|#|#|#|#|#| <--  "off"            PlyBk   set 1->off, 2->on,  3-off , 8-on & 8-off toggle
       |_______________|                       Clear   set 1->off, 2->off, 3-on ,  8-on & 8-off toggle

  Reduced sampling frequency 1/30. line after sampling is delay in ms.BW





*/

#include <NHB_AD7124.h>
#include "RTClib.h"
#include <SD.h>
#include <Wire.h>
#include <Adafruit_Sensor.h>

RTC_DS3231 rtc;
 
File root;
 
#define CH_COUNT 8 // 8 differential ADC channels   

const uint8_t csPin = 5;    // ADC chip select
const int chipSelect = 4;   // SD card strob


int inh = 6; // mux  inh used for PWR pulse signal
int adda = 10; // mux add lsb
int addb = 11; // mux add  
int addc = 12; // mux add msb

int logPin =13; // blink LED that new Log was written





Ad7124 adc(csPin, 4000000);
 
// The ADC filter select bits determine the filtering and ouput data rate
//     1 = Minimum filter, Maximum sample rate
//  2047 = Maximum filter, Minumum sample rate
//
// one can take readings at a SLOWER rate than
// the output data rate. (i.e. logging a reading every  30 seconds)
//
// NOTE: Actual output data rates in single conversion mode will be slower
// than calculated using the formula in the datasheet. This is because of
// the settling time plus the time it takes to enable or change the channel.
// 
int filterSelBits = 1024; //  max is 2047

void setup() {

  //Initialize serial and wait for port to open:
  Serial.begin (115200);
  while (!Serial) {
    ; // wait for serial port to connect.  
  }

  
  Serial.println(" **************************************************************");
  Serial.println(" *********************  SD: ECDR_log file  ********************");
  Serial.println(" *******************  RECORD  PLAYBACK CLEAR ******************");
  Serial.println(" **************************************************************");
  // setup A1,2,3 with internal pullup
 pinMode(A1,INPUT_PULLUP); pinMode(A2,INPUT_PULLUP); pinMode(A3,INPUT_PULLUP); 
int m1 = A1;    // record mode input.......HIGH=not active,  LOW= selected
int m2 = A2;    // playback mode input.....HIGH=not active, LOW= selected
int m3 = A3;    // clear file mode input...HIGH=not active,  LOW= selected
 

     if (! rtc.begin()) {
    Serial.println("Couldn't find RTC");
    
    while (1);
     }

 

  Serial.println ("Eight Channel Data Recorder");

  // Initializes the AD7124 device
  adc.begin();

  // Configuring ADC in Full Power Mode (Fastest) 
  adc.setAdcControl (AD7124_OpMode_SingleConv, AD7124_FullPower, true);

 
  // Set the "setup" configurations for different channels. There are 8 independent channel "setups"
  // in the ADC that can be configured. 
  // Each setup holds settings for: the reference used, the gain setting, filter type, and rate

  adc.setup[0].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // TMP-A            :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[1].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // RES-A            :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[2].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // TMP-B            :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[3].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // RES-B            :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[4].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // not used         :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[5].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // not used         :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[6].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // Gref             :      Internal reference, Gain = 1, Bipolar = True
  adc.setup[7].setConfig(AD7124_Ref_Internal, AD7124_Gain_1, true);    // TMP DUT          :      Internal reference, Gain = 1, Bipolar = True

  // Filter settings for each setup. allows for seperate setup for each channel
  adc.setup[0].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[1].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[2].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[3].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[4].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[5].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[6].setFilter(AD7124_Filter_SINC3, filterSelBits);
  adc.setup[7].setFilter(AD7124_Filter_SINC3, filterSelBits);

  // Set channels, i.e. what setup they use and what pins they are measuring on
  //  setchannel(ch#, setup#, "-"input, "+"input,true)
  adc.setChannel(0, 0, AD7124_Input_AIN0, AD7124_Input_AIN1, true);   //Channel 0 - 
  adc.setChannel(1, 1, AD7124_Input_AIN2, AD7124_Input_AIN3, true);   //Channel 1 - 
  adc.setChannel(2, 2, AD7124_Input_AIN4, AD7124_Input_AIN5, true);   //Channel 2 - 
  adc.setChannel(3, 3, AD7124_Input_AIN6, AD7124_Input_AIN7, true);   //Channel 3 - 
  adc.setChannel(4, 4, AD7124_Input_AIN8, AD7124_Input_AIN9, true);   //Channel 4 - 
  adc.setChannel(5, 5, AD7124_Input_AIN10, AD7124_Input_AIN11, true); //Channel 5 -
  adc.setChannel(6, 6, AD7124_Input_AIN12, AD7124_Input_AIN13, true); //Channel 6 - 
  adc.setChannel(7, 7, AD7124_Input_AIN14, AD7124_Input_AIN15, true); //Channel 7 -
  
  
  // see if the SD card is present and can be initialized:
  if (!SD.begin(chipSelect)) {
    Serial.println("Card failed, or not present");
    // don't do anything more:
    while (1);
  }
  

 pinMode(logPin, OUTPUT);   // Blink at end of LOG cycle
 pinMode(6, OUTPUT);       // inh/ pwr pin
 pinMode(10, OUTPUT);       // adda   mux address
 pinMode(11, OUTPUT);       // addb
 pinMode(12, OUTPUT);       // addc

 digitalWrite(6, HIGH);  // TMP pwr off  


// If SD OK, what mode is selected?
  m1 =digitalRead(A1); //Record log file: ECDR_log.txt
  m2 =digitalRead(A2); //Playback log file: ECDR_log.txt
  m3 =digitalRead(A3); //clear log file: ECDR_log.txt

   // ************************************************   
  //  Playback routine
   if (m2 == 0) {
      Serial.println("Playback mode");
           Serial.println("File Size"); root = SD.open("/"); printDirectory(root, 0); 
           
         File dataFile = SD.open("ECDR_log.txt");    
          if (dataFile) {
     while (dataFile.available()) {
      Serial.write(dataFile.read());
    }
    Serial.println("End of **ECDR_log.txt**  Playback");
    Serial.println("*************** ");
    dataFile.close();

    while (m2 == 0); // Stop here wait for reset
                 }
    }
  // ************************************************               
  //  Clear routine
    if (m3 == 0) {
           Serial.println("Clear mode");
           Serial.println("*****  Beginning File Size  *****"); root = SD.open("/"); printDirectory(root, 0);
 
    SD.remove("ECDR_log.txt");  // Delete old-log file
    delay(100);
    SD.open("ECDR_log.txt", FILE_WRITE);  // create a new-log file same name

    // show resulting file list
           Serial.println("**");
           Serial.println("**** New File Size ****"); root = SD.open("/"); printDirectory(root, 0);
           Serial.println("End of Clear mode"); Serial.println("*************** ");
     while (m3 == 0);  // Stop here wait for reset
                 }
   // ************************************************   
   //  RECORD routine
    if (m1 == 0) {
    
     Serial.println("Record mode");
     Serial.println("Begining File Size"); root = SD.open("/"); printDirectory(root, 0);
     Serial.println("***********************");Serial.println(" ");
     Serial.println("*******Start datalogging loop******** ");
       }

    while (m1 == 1) {
      delay(200);
    }


   Serial.println("******end of setup Start   loop******** ");

  }

    // -----------------RTC set.....ADC set.....SD card set...--------------------------------
    // -----------------Start the DataLogging phase ------------------------------

 void loop() {
      //*********************************************************************
      //***************Start Operation***************************************
      //*********************************************************************

      double readings[CH_COUNT];

    // make a clear set of strings for assembling new log  , time  & data strings
      String timeString = ""; // clear string
      String refString = "" ; // clear unix time
      String voltString = ""; // measurments from ADC , TMP117 seperate
      String logString = "";  // one entire LOG record entry per timestamp
    
         digitalWrite(6, HIGH);  // TMP pwr off  
      
      // make the following number of LOG entries
      // max files size limits


      
      // Build a new TimeStamp and unix string for each entry
        timeString = ""; // clear for new time
        refString = "" ; // clear old unix value  sec from midnight 1/1/1970
      DateTime now = rtc.now();
        timeString += String("T=");
        timeString += String(now.year(), DEC);
        timeString += String(':');
        timeString += String(now.month(), DEC);
        timeString += String(':');
        timeString += String(now.day(), DEC);
        timeString += String(':');
        timeString += String(now.hour(), DEC);
        timeString += String(':');
        timeString += String(now.minute(), DEC);
        timeString += String(':');
        timeString += String(now.second(), DEC);
        timeString += String(':');
        delay(5);
        refString = String(" unixTime= ") + String(now.unixtime()) + String(" ");
      
        // Timestamp string is done
        // Serial.print("timestamp= "); Serial.println(timeString);


      
        // Read all ADC ports, display and LOG  
        // select mux ch .....turn pwr on....read ADC...pwr off...next ch...


        
        digitalWrite(12, LOW);digitalWrite(11, LOW);digitalWrite(10, LOW);  // set mux
        delay(10);
        digitalWrite(6, HIGH);
     //     digitalWrite(6, LOW);    //  enable prowe to TMP 
        delay(10);
              readings[0] = adc.readVolts(0);                                     // read ADC

        digitalWrite(12, LOW);digitalWrite(11, LOW);digitalWrite(10, HIGH);
        delay(10);      
        readings[1] = adc.readVolts(1); 
       

        digitalWrite(12, LOW);digitalWrite(11, HIGH);digitalWrite(10, LOW); 
        delay(10);      
        readings[2] = adc.readVolts(2);
      

        digitalWrite(12, LOW);digitalWrite(11, HIGH);digitalWrite(10, HIGH);
        delay(10);       
        readings[3] = adc.readVolts(3);
      

        digitalWrite(12, HIGH);digitalWrite(11, LOW);digitalWrite(10, LOW);
        delay(10);           
        readings[4] = adc.readVolts(4);
      

        digitalWrite(12, HIGH);digitalWrite(11, LOW);digitalWrite(10, HIGH);
        delay(10);       
        readings[5] = adc.readVolts(5);
      

        digitalWrite(12, HIGH);digitalWrite(11, HIGH);digitalWrite(10, LOW);
        delay(10);      
        readings[6] = adc.readVolts(6);
      

        digitalWrite(12, HIGH);digitalWrite(11, HIGH);digitalWrite(10, HIGH);
        delay(10);      
        readings[7] = adc.readVolts(7);
       

        delay(4);
        digitalWrite(12, LOW);digitalWrite(11, LOW);digitalWrite(10, LOW);
          digitalWrite(6, LOW);
       //   digitalWrite(6, HIGH);


          //    Serial.println("8 chs of ADC are sampled ");  

        // **********************************************************************
        // One now has the  Time stamp, unix Time , and Ch0--ch7 readings 
        // make a LOG string.....store it and display it ....
        // Blink when done...do-it-again
        // **********************************************************************
        

        logString = ""  ; // clear
        // Merge strings   Time stamp + unixtime + data  
        logString +=   timeString + refString;
              
        logString += String(": ch0 = ") + String(readings[0] ,6 ) ;
        logString += String(": ch1 = ") + String(readings[1] ,6 ) ;
        logString += String(": ch2 = ") + String(readings[2] ,6 ) ;
        logString += String(": ch3 = ") + String(readings[3] ,6 ) ;
        logString += String(": ch4 = ") + String(readings[4] ,6 ) ;
        logString += String(": ch5 = ") + String(readings[5] ,6 ) ;
        logString += String(": ch6 = ") + String(readings[6] ,6 ) ;
        logString += String(": ch7 = ") + String(readings[7] ,6 ) ;
        logString += String(":: ");
      
      // open the ECDR log file.  
      File data1File = SD.open("ECDR_log.txt", FILE_WRITE);

      // if the file is available, write to it:
      if (data1File) {

        data1File.println(logString);  // log data to file
        data1File.close();

        }
      
      Serial.println(logString);


      // ****************Blink for every LOG entry*************
      digitalWrite(logPin, LOW);   // turn the LOG LED on  #13
      delay(200);                  // wait  
      digitalWrite(logPin, HIGH);  // turn the LED off  
      delay(30000); // extend delay 30s to reduce sampling frequency. BW
      // end of Sample sequence
       
       
          
      
    }


 // ****************Define SD file listing routine ************************
void printDirectory(File dir, int numTabs) {
  while (true) {

    File entry =  dir.openNextFile();
    if (! entry) {
      // no more files
      break;
    }
    for (uint8_t i = 0; i < numTabs; i++) {
      Serial.print('\t');
    }
    Serial.print(entry.name());
    if (entry.isDirectory()) {
      Serial.println("/");
      printDirectory(entry, numTabs + 1);
    } else {
      // files have sizes, directories do not
      Serial.print("\t\t");
      Serial.println(entry.size(), DEC);
    }
    entry.close();
  }
}
    
