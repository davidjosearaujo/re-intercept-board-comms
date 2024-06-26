# re-intercept-board-comms

## Authors
- Tiago Silvestre - 103554
- David Araújo - 93444

## Tools used
- Pulseview
- pterm

## Introduction

## Binaries static analysis

We used `strings` to find some strings in the binary. 

<img src="./docs/images/16_strings.png">

Although it does not reveal much, we can deduce that the system uses the I2C protocol. It appears to have keys for controlling the temperature and features for displaying temperatures in text format, among other functions.

We decided to do some static analysis of the binary elf file. We found some important functions such as the one that is called when reset is done.

Our main objective with deobfuscation was to associate some parts of the binary with his behaviour.

Deobfuscated code is present in the /guidra/1233412.gpr file.

As an example the following code is responsible for changing temperature with the keyboard.

<img src="./docs/images/09_handle_temperature_change_via_keyboard.png">


## I2C
We started by analyzing I2C communications in both pins 5 and 6.

With pulse view opened and channels 0 and 1 configured to read signals from pin 5 and 6 we got the following result.

In our case channel 0 represents the pin 6 and pin 1 reads signals from ping 5. 

<img src="./docs/images/04_period_of_comms.png">

We can't see any differences between both signals because the resolution is too low. So, we did some zoom to have the same order of magnitude of the signal.

<img src="./docs/images/05_i2c_analyzer.png">

The clock is present in channel 0 because it's signal is more uniform when compared with channel 1.

We found that the signal period was around 5us (200kHz).

A I2C decoder was applied given that channel 0 was dedicated to SDK and channel 1 to SDA.

Now to interpret the I2C messaged that are being captured, we can see that the master is writting in 0x48 (72) and 0x4c (76) the value 0. After each write operation, a read operation is followed for the same address.

The value that is read is the same as the left and right temperature sensors. So, we assume that both 0x4c and 0x48 are the I2C addresses of the temperature sensors.


## RS232
We also intercepted messages from RS232 signals, the same procedure was done as I2C (pulse view, connecting the pins accordingly, ...).

When we applied RS232 decoder from pulseview the program detected some corrupted frames (bad parity, etc ...) so we had to do some trial and error until we discoreved the configuration that worked for the signal.

To discover the baudrate we picked the smallest period (104us) that we found and calculated the baudrate dividing 1e6 / 104 which resulted ~9615 which is closer for 9600 standard baud rate value for this protocol.

<img src="./docs/images/07_RS232_serial_capture_specs.png">


We were not finding anything until we pressed reset button which revealed the following string.

<img src="./docs/images/08_RS232_reset.png">

Allowing us to guarantee that the configuration is set up correctly and to continue exploring.


## SPI
We also analyzed SPI protocol, at first tries we were not able to find anything relevant because the frequency of capture was set too low. After checking the memory chip documentation (MICROCHIP 25LC040A), we found that the frequency was in the order of MHZ so we did a capture with 24Mhz. After that the capture seemed to be correct because the clock signal was stable.

<img src="./docs/images/11_SPI_frequency.png">

To identify the clock signal, we choose the signal that had a duty cycle of 50% and behaved like a clock.
The clock period of the clock is ~2.7us which results in a frequency of ~369kHz.

The logic to identify chip select signal was to find the signal that was up when clock was disabled and was down when other signals were changing.

The last two signals MOSI and MISO were harder to identify, but we found two conclusions to support our argument.

1. Due the protocl being a master slave one, a communication should start by having master requesting data to the slave (memory chip). Which means that the first signal that changes should be the master (MOSI). In our case is the channel 3.

2. The CHIP [documentation](https://ww1.microchip.com/downloads/aemDocuments/documents/MPD/ProductDocuments/DataSheets/25AA040A-25LC040A4-Kbit-SPI-Bus-Serial-EEPROM-20001827J.pdf) reveals the instruction set and how a write sequence is done (in section 3.3). Reading how write works, we found that MOSI signal must be the channel 3 because we found a write sequence in this channel, it should start with 0x06 and then setting chip select to high and low, after that the master sends 0x02 and the address and value that should be written memory with additional two bytes. This sequence was not found in the remaining channels.

<img src="./docs/images/12_CHIP_IS.png">
<img src="./docs/images/13_SPI_WRITE.png">

The start capture was decoded according with the chip documentation the following operations are done.

```
1. READ SR 0x00
2. READ 0x45 0x40
3. READ 0x46 0x20
4. WRITE 0x22 0x1E
5. READ SR 0x03 (~20 times)
6. WRITE 0x23 0X1E
7. READ SR 0x03 (~20 times)
8. WRITE 0x46 0x22
```

As we can see it starts by reading address 0x45 and 0x46. And then the value 0x1E is written in 0x22 and 0x23 which seems to be the temperature values.
After value 0x22 is set in 0x46.

In the middle of these operations some read status register operations are done. The value 0x03 os status register means that WIP (busy write) and WEL (Write Enable Latch) bits are active, these instructions are repeated around ~20 times because the system is waiting for the write operation to be done in order to make another write.

## OC1 and OC2 outputs
We also captured OC1 and OC2 outputs.

The following image is a capture of OC1 when temperature was 40, 50 and 60.

<img src="./docs/images/14_OC1.png">

We did another capture for OC2 with same characteristics, temperature of 40, 50 and 60.

<img src="./docs/images/15_OC2.png">

As we can see, as the temperature increases, the duty cycle of the signal increases.

For OC1 signal the duty cycle is ~43% when temperature is 40 and ~81% when temperature is 50.

For OC2 signal, it's almost close from values captured in OC1. ~43% when temperature is 40 and 86 when it's 50.

We did multiple captures to confirm that the duty cycle reaches 100% when temperature is around 60.

The behavior of this signal is similar to a PWM (Pulse Width Modulation) signal, where the duty cycle increases as the temperature rises. This means the temperature is regulated by this signal. To increase the temperature, the duty cycle must be increased, which in turn raises the average power.
