#---------------------------------- author data ----------------------------------
#Engineering Student: Joel E Martin
#University: Washington State University, WSU
#Email: JMartinUSN@gmail.com or Joel.e.Martin@wsu.edu
#---------------------------------- documentation ----------------------------------
#Information: This code facilitates daisy chaining any number of pico's together via uart connections
#Each pico can have two pico's connected, the user only need to insure that the myIdent value is different for every Pico and is four ascii
#characters, Example: myIdent = "1001" or myIdent = "PIC1", exception do not use "0000" that is reserved for a broadcast message
#sample connection: PICO1(uart0) >> << (uart0)PICO2(uart1) >> << (uart1)PICO3(uart0) >> << (uart1)PICO4, end pico's require no termination on open ports
#pico's need not be in any order, and any port can be used.
#every pico generates a high on GPIO15:DTR which should be wired to the uart channel CTS of the opposing pico
#GPIO14 for UART0 CTS and GPIO17 for UART1 CTS, if no CTS is wired reception will not occur on that port
#---------------------------------- get modules ----------------------------------
import sys
import _thread
import array
import machine
from machine import WDT
from machine import Pin
from machine import UART
import utime
from time import sleep
import math
from PicoSensor import Temperature
#---------------------------------- setup constants and gpio ----------------------------------
sys.implementation
led_onboard = machine.Pin(25, machine.Pin.OUT)

myIdent = "1001"

uart0cts = Pin(14, mode=Pin.IN, pull=Pin.PULL_DOWN)		#default pin to low
uart1cts = Pin(17, mode=Pin.IN, pull=Pin.PULL_DOWN)		#default pin to low
dtr = Pin(15, mode=Pin.OUT, pull=Pin.PULL_DOWN)			#default pin to low
dtr.off()												#state of dtr to not ready

rx_message = ""
tx_message = ""
my_rx_message = []

sensor = Temperature()									#optional, use for demo
#---------------------------------- halt and catch fire ----------------------------------
#if you have this code saved as main.py, the pico will auto execute
#breaking into the pico is difficult if trying to edit the code
#to cause a premature break, connect pin 3.3v to pin 16 which will halt the main
#checking halt during a second thread helps stop both threads
halt = Pin(16, mode=Pin.IN, pull=Pin.PULL_DOWN)			#default pin to low
if halt.value() == True:
        led_onboard.toggle()
        sleep(.05)
        led_onboard.toggle()
        sleep(.05)
        led_onboard.toggle()
        sleep(.05)
        sys.exit(0)
#----------------------------------watchdog declaration----------------------------------        
#watchdog = WDT(timeout=500)  # Timeout in milliseconds (adjust as needed)

#----------------------------------uart declaration---------------------------------- 
# Create UART object with baud rate (adjust as needed), 115200 works best
uart0 = UART(0,baudrate=115200, tx=Pin(0), rx=Pin(1), bits=7, parity=None, stop=0, txbuf=1024)
uart1 = UART(1,baudrate=115200, tx=Pin(4), rx=Pin(5), bits=7, parity=None, stop=0, txbuf=1024)

#----------------------------------tx uart 0/1 function---------------------------------- 
#txdata sould be sent in the following format
#"TX" + myIdent + "!" + "RX" + receiver ident + "!MID" + message indent (time or number) + "!MDA" + ASCII message
#Setting RX0000 is a broadcast message that will be sent though all ports to all daisy chained pico's
#do not add newline at end of message, send_data function does that for you...
# Sample after constuction: "TX1003!RX1002!MID224!MDA:ASCII message"
#i is the UART port 0 or 1, usually transmit the message on both.... initially
def send_data(txdata,i):
  # Send data string with newline character
    if i == 0:
        uart0.write(txdata + "\n")
    else:
        uart1.write(txdata + "\n")
    #uart0.write(bytes(txdata,'utf-8'))
#----------------------------------rx uart 0/1 function---------------------------------- 
def receive_data(i)->str:
    new_line = False
    rxdata = ""
    if i==0:
        now = utime.ticks_ms()
        while (not new_line) and (utime.ticks_diff(utime.ticks_ms(), now) < 1000):
            if (uart0.any() > 0):
                rxdata = rxdata + uart0.read().decode('utf-8')
                if '\n' in rxdata:
                    new_line = True
                    rxdata = rxdata.strip('\n')
                    return rxdata
            else:
                return ""
    else:
        now = utime.ticks_ms()
        while (not new_line) and (utime.ticks_diff(utime.ticks_ms(), now) < 1000):
            if (uart1.any() > 0):
                rxdata = rxdata + uart1.read().decode('utf-8')
                if '\n' in rxdata:
                    new_line = True
                    rxdata = rxdata.strip('\n')
                    return rxdata
            else:
                return ""
#----------------------------------second thread----------------------------------           
def second_thread():
    while halt.value() == False:
        pass
        #do stuff

    _thread.exit()

#----------------------------------main thread---------------------------------- 
if __name__ == "__main__":
    sleep(2)											#stall the main so everything is initialized
    #second_thread = _thread.start_new_thread(second_thread, ())
    #watchdog.feed()  									#enable this if you want the WDT to reboot on program crash
    sleep(1)											#stall the main so everything is initialized, may need to increase if 2nd thread started
    counter = 0											#intialize loop counter
    txcounter = 0										#intialize tx message counter
    rxcounter = 0										#intialize rx message counter
    rx_message = ""										#intialize rx message variable
    now = utime.ticks_ms()
    led = utime.ticks_ms()   
    dtr.on()											#this tells other picos I am ready to communicate
#----------------------------------loop to transmitt and receive, and process
    while halt.value() == False:
        counter = counter +1							#count the number of times this loop is executed
#----------------------------------check the decoded message buffer
        if len(my_rx_message) > 0:
            rxcounter = rxcounter + 1
            print(my_rx_message[0])						#print the decoded message, this command is not required, execution will be faster without it
            #insert your operation based on message here
            del my_rx_message[0]						#delete the array element just printed or processed
#----------------------------------send some stuff out, all thesse messages are samples, you must obey predicates to MDA       
        if utime.ticks_diff(utime.ticks_ms(), led) > 700:  
            led_onboard.toggle()						#optional
            led = utime.ticks_ms()
            if uart0cts.value() == True:				#transmit same message on uart 1 if connected
                send_data("TX"+myIdent+"!RX1003!MID:" + str(txcounter) + "!MDA:Temp" + str(sensor.ReadTemperature()),0)
                sleep(.01)								#always put a small delay between transmit messages
                send_data("TX"+myIdent+"!RX0000!MID:" + str(txcounter) + "!MDA:Messages RX'd:"+str(rxcounter),0)
                txcounter = txcounter + 1
            if uart1cts.value() == True:				#transmit same message on uart 1 if connected
                send_data("TX"+myIdent+"!RX1003!MID:" + str(txcounter) + "!MDA:Temp" + str(sensor.ReadTemperature()),1)
                sleep(.01)								#always put a small delay between transmit messages
                send_data("TX"+myIdent+"!RX0000!MID:" + str(txcounter) + "!MDA:Messages RX'd:"+str(rxcounter),1)
            
                     
#----------------------------------check uart0 for incomming messages will be ignored if gpio14 is low   
        if uart0cts.value() == True:
            rx_message = receive_data(0)
            if rx_message:
                if rx_message[9:13] == myIdent:
                    #take action on this message and dont re-transmit this message 
                    #because it's a message to me and not from me
                    my_rx_message.append(rx_message)
                    rx_message = ""
                elif rx_message[2:6] == myIdent:
                    #discard this message because i sent it and it's echo'd back to me
                    rx_message = ""
                elif rx_message[9:13] == "0000":
                    #take action on this message and re-transmit this message 
                    #because it's a broadcast message to everyone and it's not from me
                    my_rx_message.append(rx_message)
                    send_data(rx_message,1)
                    rx_message = ""       
                else:
                    #don't take action on this message and re-transmit this message 
                    #because it's a specific message to someon else but it's not from me or to me
                    send_data(rx_message,1)
                    rx_message = ""
                    
#----------------------------------check uart0 for incomming messages will be ignored if gpio17 is low                
        if uart1cts.value() == True:
            rx_message = receive_data(1)
            if rx_message:
                if rx_message[9:13] == myIdent:
                    #take action on this message and dont re-transmit this message 
                    #because it's a message to me and not from me
                    my_rx_message.append(rx_message)
                    rx_message = ""
                elif rx_message[2:6] == myIdent:
                    #discard this message because i sent it and it's echo'd back to me
                    rx_message = ""
                elif rx_message[9:13] == "0000":          
                    #take action on this message and re-transmit this message 
                    #because it's a broadcast message to everyone and it's not from me
                    my_rx_message.append(rx_message)
                    send_data(rx_message,0)
                    rx_message = ""       
                else:
                    #don't take action on this message and re-transmit this message 
                    #because it's a specific message to someon else but it's not from me or to me
                    send_data(rx_message,0)
                    rx_message = ""   

#----------------------------------halt main thread----------------------------------
#assume GPIO16 has been set to high
sleep(1) 											#give time for second thread to halt
sys.exit(0)											#exit the program




















