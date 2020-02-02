# AVR C 1-Wire + DS18B20 temperature sensor

This implements Maxim's One Wire / 1-Wire protocol for AVR microcontrollers from the [DS18B20 datasheet](https://datasheets.maximintegrated.com/en/ds/DS18B20.pdf) and Maxim's other application notes.

## Usage

### Single DS18B20 device on bus

With just one DS18B20 on the 1-Wire bus, you can ignore slave addresses:

```c
#include "pindef.h"
#include "onewire.h"
#include "ds18b20.h"

// ...

// This is a hacked together interface to pass around a port/pin combination by reference
// Everything in pindef.h compiles to quite inefficient asm currently.
const gpin_t sensorPin = { &PORTD, &PIND, &DDRD, PD3 };

// Send a reset pulse. If we get a reply, read the sensor
if (onewire_reset(&sensorPin)) {

    // Not passing a specific address
    onewire_skiprom(&sensorPin);
    
    // Start a temperature reading
    ds18b20_convert(&sensorPin);
    
    // Wait for measurement to finish (750ms for 12-bit value)
    _delay_ms(750);
    
    // Get the raw 2-byte temperature reading
    int16_t reading = ds18b20_read_single(&sensorPin);
    
    if (reading != kDS18B20_CrcCheckFailed) {
        // Convert to floating point (or keep as a Q12.4 fixed point value)
        float temperature = ((float) reading) / 16;
    } else {
        // Handle bad temperature reading CRC 
        // The datasheet suggests to just try reading again
    }
}
```

### Multiple devices on the bus

With more than one DS18B20 on the bus, you'll need to query the unique 64-bit address of each device and address them directly when sending other commands.

```c
#include "onewire.h"

// ...

// Prepare a new device search
onewire_search_state search;
onewire_search_init(&search);

// Search and dump temperatures until we stop finding devices
while (onewire_search(&sensorPin, &search)) {
    if (!onewire_check_rom_crc(&search)) {
        // Handle ROM CRC check failed
        continue;
    }

    // Do something with the address
    // eg. memcpy(addresses[slaveIndex], search.address, 8);
}
```

To send commands to a specific slave, use `onewire_match_rom` instead of `onewire_skiprom`:

```c
onewire_match_rom(&sensorPin, address);
```

## Search ROM implementation

I found the Search ROM flowchart in Maxim's datasheets and application notes a little hard to follow. Below is a diagram I threw together to maintain sanity implementing this. It goes with p51-54 from [Maxim's application note 937: Book of iButton® Standards](https://pdfserv.maximintegrated.com/en/an/AN937.pdf).

Also see [application note 187: 1-Wire Search Algorithm](https://www.maximintegrated.com/en/app-notes/index.mvp/id/187). The implementation described there adds additional complexity/functionality to the search process, but also includes a (fairly verbose) implementation in C.
