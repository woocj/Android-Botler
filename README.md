# Android-Botler
#include <avr/io.h>
#include <avr/interrupt.h>

/*********************
* Function Prototypes
*******************/
void setupSerial(void);
void setupTimer(void);
void byteTx(uint8_t);
void turn90(void);
void turn180(void);
void stop(void);
void drive(void);
void delayMs(uint16_t);
void clearReceiveBuffer(void);
uint8_t byteRx(void);
/**********************
* Variable Declarations
*********************/
volatile uint16_t timerCount = 0;
volatile uint8_t timerRunning = 0;
volatile int16_t sensor=0;
volatile int16_t sensor2=0;
int hit=0;
int main(void){
	/**************************
	* Run intialization methods.
	**************************/
	cli();
	//Properly set serial pins.
	setupSerial();
	//Create the 1 microsecond interrupt.
	setupTimer();
	sei();
	

byteTx(128); //Start
byteTx(132); //Full mode

int hit=0;

while (hit<3){ //While the iRobot has not reached the end of the maze
		
	drive();
	byteTx(142);
	byteTx(7);
	uint8_t obstacle=byteRx();
	
	if (obstacle!=0){ //If the robot hits an obstacle
		hit++;
	}
			
			
	if (obstacle!=0 && sensor<=1500){ //If the robot hits the obstacle within the first 1.5s
		hit++;
		sensor=5000;
	}
	
	if (obstacle!=0 && sensor2<=1500){ //If the robot hits the obstacle within the first 1.5s
		hit++;
		sensor2=5000;
	}
		

	if(hit==1){
		turn90();
		sensor=0;
		obstacle=0;
		hit=0;
	}
	
	if (hit==2){
		turn180();
		sensor2=0;
		sensor=0;
		obstacle=0;
		hit=0;
	}
	
}	
	stop();
	
}

	


//Any methods you want to write can go here!

/*****************************
*READ ABOUT THESE FUNCTIONS!!!
****************************/

//This is a one millisecond interrupt function. That means that
//every 1ms, this method gets called by the program, regardless of what
//else is happening. This makes it ideal for timing functions.
//In fact, it is already configured to work properly as a delay timer.
//The DelayMS function utilizes this function to decrease timerRunning once
//every millisecond. Once timerRunning reaches 0, the delayMS function quits 
//and the program resumes.
SIGNAL(SIG_OUTPUT_COMPARE1A) {	\
	if(timerRunning) {
		if(timerCount != 0) {
			timerCount--;
		} else {
				timerRunning = 0;
			}
	}
	
	sensor++;
	sensor2++;
}

//This is the function for sending data, one byte at a time,
//to the iRobot. It waits until the transmission lines are clear
//and then transmits the byte, value, to the robot.
//So if you wanted to send the number 128 to the iRobot, you would use
//byteTx(128).
void byteTx(uint8_t value) {
	// Transmit one byte to the robot.
	// Wait for the buffer to be empty.
	while(!(UCSR0A & 0x20)) {
	// Do nothing.
	}
	// Send the byte.
	UDR0 = value;
}

//This clears old data out of the input registers.
//Without calling this method before attempting to read in input the
//iRobot will behave very differently than expected. USE THIS BEFORE REQUESTING
//EVERY BYTE OF DATA!!
void clearReceiveBuffer(){
	uint8_t empty; //Buffer drain.
	while(UCSR0A & 0x80) {/* waits for the buffer to empty */
		empty = UDR0;
	}
	
}
//This is the counterpart to byteTx(uint8_t). Where byteTx
//sends a byte, byteRx receives a byte.
//It is important to realize that byteRx will wait until there
//is a byte to receive before doing anything. As a result,
//if you use byteRx, your program will pause until it is sent
//a byte from the iRobot.
uint8_t byteRx(void){	
	while(!(UCSR0A & 0x80)) ;
	/* wait until a byte is received */
	return UDR0;
}

/*****************************************************
*	Initialization Code. (dont worry about this stuff)
****************************************************/
void setupTimer(void) {
	// Set up the timer 1 interupt to be called every 1ms.
	TCCR1A = 0x00;
	TCCR1B = 0x0C;
	OCR1A = 71;
	TIMSK1 = 0x02;
}
void delayMs(uint16_t timeMs) {
	timerCount = timeMs;
	timerRunning = 1;
	//timerCount gets decremented every 1ms by the 
	//interrupt function above. Once the interrupt determines
	//that timerCount has been reduced to 0, it sets timerRunning
	//to 0, which will terminate the following while loop.
	while(timerRunning) {
	// do nothing
	}
}

void setupSerial(void) {
	// Set the transmission speed to 57600 baud, which is what the Create expects,
	// unless we tell it otherwise.
	UBRR0 = 19;
	// Enable both transmit and receive.
	UCSR0B = 0x18;
	// Set 8-bit data.
	UCSR0C = 0x06;
}

void turn90(void) {
byteTx(137); //Driving to turn 90 degrees
	byteTx(255); //Velocity high byte
	byteTx(56); //Velocity low byte
	byteTx(0); //Radius high byte
	byteTx(1); //Radius low byte
	delayMs(1080);
}

void turn180(void) {
	byteTx(137); //Driving to turn 180 degrees
	byteTx(255); //Velocity high byte
	byteTx(56); //Velocity low byte			
	byteTx(0); //Radius high byte
	byteTx(1); //Radius low byte
	delayMs(2190);
}

void stop(void) {
	byteTx(137); //Stop Driving
	byteTx(0); //No right high velocity
	byteTx(0); //No right low velocity
	byteTx(0); //No left high velocity
	byteTx(0); //No left low velocity
}

void drive(void) {
	byteTx(137); //Drive
	byteTx(1); //high velocity 
	byteTx(90); //low velocity 
	byteTx(0x80); //Radius high byte
	byteTx(0x00); //Radius low byte
}
