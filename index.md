# Music Controller and FFT Display
### Team Members: Annie Luo, Dimitry Jean-Laurent, Emmy Perez, Shyam Patel


# Project Description
For our project, we decided to create a music controller with an equalizer display. A number of small audio samples were placed in an SD card as .wav files to then be analyzed using Fast Fourier Transforms from within the mbed. The resulting frequency ratios were used to create 5 different outputs shown on the 8x8 LED matrix display that served as a visual representation of what was coming out of the speaker at that point in time. A C# GUI created in Visual Studio was used to select a song from the list stored in the SD card, which allowed the user to have control over what was being played.


## Library Used
  - mbed.h
  - SDFileSystem.h
  - wave_player.h
  - Adafruit_LEDBackpack.h
  - Adafruit_GFX.h
  - rtos.h
  - FFT.h


## Parts Used
  - Speaker
  - Class D Audio Amplifier
  - Potentiometer
  - SD Card Reader
  - Adafruit LuckyLight 8x8 LED Display
  - mbed LPC1768


## Wiring Diagram
![Wiring Diagram](./WiringDiagram.png)


## C# GUI
In the GUI, you can choose the COM port for the SD card to read in the song library and select what song you would like to listen to. After making your selection, the GUI then displays the song name while the song is playing.

```
using System;
      using System.Collections.Generic;
      using System.ComponentModel;
      using System.Data;
      using System.Drawing;
      using System.Linq;
      using System.Text;
      using System.Threading.Tasks;
      using System.Windows.Forms;
      using System.IO.Ports;
      using System.Diagnostics;

      namespace Final_Project_4180
      {
        public partial class Form1 : Form
        {
          private String[] portNames;
          private String[] songs = { "africa-toto", "around_the_world-atc", "beautiful_life-ace_of_base", "dont_speak-no_doubt", "my-love", "Song1_test" };
          private bool play = false;
          private String song = "";
      
          public Form1()
          {
            InitializeComponent();
          }

          private void comboBox1_SelectedValueChanged(object sender, EventArgs e)
          {
            serialPort1.PortName = comboBox1.Text;
          }

          private void Form1_Load(object sender, EventArgs e)
          {
            Debug.Print("hello 4180 project!");
            try
            {
                Debug.Print("init load");
                portNames = SerialPort.GetPortNames();
                comboBox1.Items.AddRange(portNames);
                comboBox1.SelectedIndex = 0;
                Debug.Print("end load");
            }
            catch (Exception err)
            {
                Debug.Print("loading error: " + err.Message);
            }
            Debug.Print("Connected Mbed to " + comboBox1.Text);


          }

          private void comboBox1_TextUpdate(object sender, EventArgs e)
          {
            if (serialPort1.IsOpen) serialPort1.Close();
            serialPort1.PortName = comboBox1.Text;
            serialPort1.Open();
          }

          private void Form1_FormClosed(object sender, FormClosedEventArgs e)
          {
            serialPort1.Close();
          }

          private void comboBox2_SelectedValueChanged(object sender, EventArgs e)
          {
            switch (comboBox2.SelectedItem.ToString())
            {
                case "Africa":
                    Debug.Print("africa-toto");
                    song = "!1";
                    break;
                case "Around the World":
                    Debug.Print("around_the_world-atc");
                    song = "!2";
                    break;
                case "Beautiful Life":
                    Debug.Print("beautiful_life-ace_of_base");
                    song = "!3";
                    break;
                case "Don't Speak":
                    Debug.Print("dont_speak-no_doubt");
                    song = "!4";
                    break;
                case "My Love":
                    Debug.Print("my-love");
                    song = "!5";
                    break;
                case "Song1_test":
                    Debug.Print("Song1_test");
                    song = "!6";
                    break;
                default:
                    break;
            }
            label3.Text = comboBox2.SelectedItem.ToString();
            if (serialPort1.IsOpen)
            {
                serialPort1.Write(song);
            }
            else
            {
                try
                {
                    serialPort1.Open();
                } catch (Exception err)
                {
                    Debug.Print("Error: " + err.Message);
                }
            }
          }

          private void button1_Click(object sender, EventArgs e)
          {
            play = !play;
            if (serialPort1.IsOpen)
            {
                if (play == true)
                {
                    serialPort1.Write("!P");
                }
                else
                {
                    serialPort1.Write("!S");
                }
            }
            else
            {
                try
                {
                    serialPort1.Open();
                } catch (Exception err)
                {
                    Debug.Print("Error: " + err.Message);
                }
            }
          }
        }
      }
```


## mbed Setup and FFT
We have 2 threads in our program. One that that takes in data from the C# GUI and plays the music selection on the speaker and another that runs an FFT analysis on the music and then determines what frequency bin that the output falls into and displays it. In order to do the FFT analysis on the data we ran into some issues grabbing data in the form of an array from our .wav file in order to run this through the FFT library that we were using. To fix this issue, we had to first create an .txt file containing an array of values from the .wav file using the WAVToCode software and turn these into global variables to access from within the code. From there, we were able to sample the data and perform our calculations for the output.

```
#include "mbed.h"
#include "SDFileSystem.h"
#include "wave_player.h"
#include "Adafruit_LEDBackpack.h"
#include "Adafruit_GFX.h"
#include "rtos.h"
#include "FFT.h"
#include <string>
#include "around_the_world-atc-array.h"
#include "dont_speak-no_doubt-array.h"
#include "my-love-array.h"
 
SDFileSystem sd(p5, p6, p7, p8, "sd"); // SD card
 
I2C i2c(p28, p27); // LED display
Adafruit_8x8matrix matrix = Adafruit_8x8matrix(&i2c);
 
AnalogOut DACout(p18); // speaker
wave_player waver(&DACout);
Mutex speaker_lock;
Mutex file_lock;
DigitalOut myled(LED1); // mbed LED
Serial pc (USBTX,USBRX);
string dir;         // "/sd/" + song + ".wav"
string song;
int song_data_length;
const unsigned short *song_data;
bool play = false;
bool song_selected = false;

#define BUFFER_SIZE 64
 
// states for display output
const int on[8][8] = {
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1
};
 
const int high[8][8] = {
    0, 0, 0, 1, 1, 0, 0, 0,
    0, 0, 0, 1, 1, 0, 0, 0,
    0, 0, 1, 1, 1, 1, 0, 0,
    0, 1, 1, 1, 1, 1, 1, 0,
    0, 1, 1, 1, 1, 1, 1, 0,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1
};
 
const int med[8][8] = {
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 1, 1, 0, 0, 0,
    0, 0, 1, 1, 1, 1, 0, 0,
    0, 0, 1, 1, 1, 1, 0, 0,
    0, 1, 1, 1, 1, 1, 1, 0,
    1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1
};
 
const int low[8][8] = {
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 1, 1, 1, 1, 0, 0,
    0, 1, 1, 1, 1, 1, 1, 0,
    1, 1, 1, 1, 1, 1, 1, 1
};
 
const int off[8][8] = {
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0
};
 
void display_fft(float sample[])
{
 
    int state = 0;
    float freq = 0;
    matrix.begin(0x70);        
    
    //calculates FFT value of passed in array
    //Value scaled down by 4 for visualization
    freq = sample[1]/4;
 
    // determine state based on the FFT output range
    if (freq < 10000) {
        state = 5; // OFF
    } else if (freq >= 10000 && freq < 25000) {
        state = 4; // LOW
    } else if (freq >= 25000 && freq < 40000) {
        state = 3; // MED
    } else if (freq >= 40000 && freq < 55000) {
        state = 2; // HIGH
    } else if (freq >= 55000) {
        state = 1; // ON
    }
 
    // switch statement based on the state
    switch (state) {
        case(1): // CASE HIGHEST
            matrix.clear();
            for (int i = 0; i < 8; i++) {
                for (int j = 0; j < 8; j++) {
                    if (on[i][j] == 1) {
                        matrix.drawPixel(i, j, LED_ON);
                    }
                }
            }
            matrix.writeDisplay();
            Thread::wait(10);
            break;
 
        case(2): // CASE HIGH
            matrix.clear();
            for (int i = 0; i < 8; i++) {
                for (int j = 0; j < 8; j++) {
                    if (high[i][j] == 1) {
                        matrix.drawPixel(i, j, LED_ON);
                    }
                }
            }
            matrix.writeDisplay();
            Thread::wait(10);
            break;
 
        case(3): // CASE MEDIUM
            matrix.clear();
            for (int i = 0; i < 8; i++) {
                for (int j = 0; j < 8; j++) {
                    if (med[i][j] == 1) {
                        matrix.drawPixel(i, j, LED_ON);
                    }
                }
            }
            matrix.writeDisplay();
            Thread::wait(10);
            break;
 
        case(4): // CASE LOW
            matrix.clear();
            for (int i = 0; i < 8; i++) {
                for (int j = 0; j < 8; j++) {
                    if (low[i][j] == 1) {
                        matrix.drawPixel(i, j, LED_ON);
                    }
                }
            }
            matrix.writeDisplay();
            Thread::wait(10);
            break;
 
        case(5): // CASE OFF
            matrix.clear();
            for (int i = 0; i < 8; i++) {
                for (int j = 0; j < 8; j++) {
                    if (off[i][j] == 1) {
                        matrix.drawPixel(i, j, LED_ON);
                    }
                }
            }
            matrix.writeDisplay();
            Thread::wait(10);
            break;
 
        default:
            break;
    }
 
 
}
 
//Reads SD card and passes value to frequency display
void display_thread(void const* args)
{
    while(1) {
        
        //Buffer which will hold 64 values
        float buffer[BUFFER_SIZE];
        
        //index for iteratiing through buffer
        int buffer_index = 0;
        
        //index for iterating through song array
        int song_array_index = 0;
        
        if(play) {
            while(play && song_array_index < song_data_length) {
            
                buffer[buffer_index] = song_data[song_array_index];
                song_array_index++;
                buffer_index++;

                if(buffer_index == BUFFER_SIZE) {
                    //fast fourier tranform function here
                    vRealFFT(buffer, 4);
                    display_fft(buffer);
                    buffer_index = 0;
                    memset(buffer,0,sizeof(buffer));
 
                }
 
            }
        }
        Thread::wait(300);
    }
 
}
 
void speaker_thread(void const* args)
{
    while(1) {

        if(play == true && song_selected == true) {
            speaker_lock.lock();
            FILE *wave_file;
            dir = "/sd/" + song + ".wav";
 
            file_lock.lock();
            wave_file=fopen(dir.c_str(),"r");
            if(wave_file == NULL)
            {
                pc.printf("Error opening music file\n");
            }
            file_lock.unlock();
 
            waver.play(wave_file);
 
            file_lock.lock();
            fclose(wave_file);
            file_lock.unlock();
            play = false;
            myled = 0;
            speaker_lock.unlock();
        } 
        Thread::wait(1000);
    }
 
} 
 
int main()
{
    //starting thread
    Thread th1(speaker_thread);
    Thread th2(display_thread);    
    
    char c;
    while(1) {
        while(!pc.readable()) {
            Thread::wait(1);
        }
        
        //selects the song
        if (pc.getc() == '!') {
            c = pc.getc();
            switch(c) {
                case '2':
                    song = "around_the_world-atc";
                    song_data_length = NUM_ELEMENTS_AROUND_THE_WORLD;
                    song_data = data_around_the_world;
                    song_selected = true;
                    break;
                case '4':
                    song = "dont_speak-no_doubt";
                    song_data_length = NUM_ELEMENTS_DONT_SPEAK_NO_DOUBT;
                    song_data = data_dont_speak_no_doubt;
                    song_selected = true;
                    break;
                case '5':
                    song = "my-love";
                    song_data_length = NUM_ELEMENTS_MY_LOVE;
                    song_data = data_my_love;
                    song_selected = true;
                    break;
                case 'P':
                    play = true;
                    myled =1;
                    break;
                case 'S':
                    play = false;
                    myled = 0;
                    break;
                default:
                    break;
            }
        }
        Thread::wait(100);
    }
 
} 
```


## Videos
### Demonstration
[![FFT Equalizer Demo](http://img.youtube.com/vi/_YG1I_EaH9k/0.jpg)](http://www.youtube.com/watch?v=_YG1I_EaH9k "FFT Equalizer Demo")
### Presentation 
[![FFT Equalizer Presentation](http://img.youtube.com/vi/gh7g6DiaTOI/0.jpg)](http://www.youtube.com/watch?v=gh7g6DiaTOI "FFT Equalizer Presentation")
