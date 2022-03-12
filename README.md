# LoRaMesher

## Introduction

The LoRaMesher library implements a distance-vector routing protocol for communicating messages among LoRa nodes. For the interaction with the LoRa radio chip, we leverage RadioLib, a versatile communication library which supports the SX127X LoRa series module available on the hardware we used, among others.

## Dependencies

You can check `library.json` for more details. Basically, we use a modded version of [Radiolib](https://github.com/jgromes/RadioLib) that supports class methods as callbacks and [FreeRTOS](https://freertos.org/index.html) for scheduling maintenance tasks.

## Configure LoRaMesher with PlatformIO and Visual Studio Code

1. Download Visual Studio Code.
2. Download PlatformIO inside Visual Studio Code.
3. Clone the LoraMesher repository.
4. Go to PlatformIO Home, click on the Projects button, then on "Add Existing", and find the examples/beta-sample source in the files.
5. Select the examples/beta-example project.
6. Build the project with PlatformIO.
7. Upload the project to the specified LoRa microcontroller. In our case, we use the TTGO T-Beam module.

## LoRaMesher Example

There is, in the source files of this first implementation, an example to test the new functionalities. This example is an implementation of a counter, sending a broadcast message every 10 seconds. To make it easier to understand, we will remove additional functions that are not necessary to make the microcontroller work with the LoRaMesher library.

### Defining the data type and the data counter

As a proof of concept, we will be sending a numeric counter over LoRa. Its value will be incremented every 10 seconds, then te packet will be transmitted. To start, we need to implement the type of data we will use.

In this case, we will only send a `uint32_t`, which is the counter itself.

```
uint32_t dataCounter = 0;
struct dataPacket {
  uint32_t counter = 0;
};

dataPacket* helloPacket = new dataPacket;
```

### LoRaMesh Initialization

To initialize the new implementation, LoRaMesher must be initialized with a function call. This function will be notified every time the microcontroller receives an incoming packet for the user.

```
Serial.begin(115200); //This configuration can be changed
Serial.println("initBoard");

//Create a LoraMesher with a processReceivedPackets function
radio = new LoraMesher(processReceivedPackets);
```

We can see that, when starting a new instance of LoRaMesher, we need to pass through a function.

### Received packets function

The function that gets a notification each time the library receives a packet for the user looks like this one:

```
/**
 * @brief Function that process the received packets
 *
 */
void processReceivedPackets(void*) {
  for (;;) {
    /* Wait for the notification of processReceivedPackets and enter blocking */
    ulTaskNotifyTake(pdPASS, portMAX_DELAY);

    //Iterate through all the packets inside the Received User Packets FiFo
    while (radio->ReceivedUserPackets->Size() > 0) {
      //Get the first element inside the Received User Packets FiFo
      LoraMesher::packetQueue<dataPacket>* helloReceived = radio->ReceivedUserPackets->Pop<dataPacket>();

      //Print the data packet
      printDataPacket(helloReceived->packet);

      //Delete the packet when used. It is very important to delete the packets.
        delete helloReceived;
      }
  }
}
```

There are some important things we need to be aware of:

1. This function should have a `void*` in the parameters.
2. The function should contain an endless loop.
3. Inside the loop, it is mandatory to have the `ulTaskNotifyTake(pdPASS,portMAX_DELAY)` or equivalent. This function allows the library to notify the function to process pending packets.
4. All the packets are stored inside `radio->ReceiveduserPackets`.
5. Every time you call Pop, you need to be sure to delete the Packet Queue after using it.

### Send data packet function

In this section we will present how you can create and send packets. in this example we will use the `dataPacket` data structure.

```
  void loop() {
        helloPacket->counter = dataCounter++;

        //Create packet and send it.
         radio->createPacketAndSend(BROADCAST_ADDR, helloPacket, 1);

        //Wait 10 seconds to send the next packet
        vTaskDelay(10000 / portTICK_PERIOD_MS);
  }
```

In the previous figure we can see that we are using the helloPacket, we add the counter inside it, and we create and send the packet using the LoRaMesher.

The most important part of this piece of code is the function that we call in the `radio->createPacketAndSend()`:

1. The first parameter is the destination, in this case the broadcast address.
2. And finally, the helloPacket (the packet we created) and the number of elements we are sending, in this case only 1 dataPacket.

### Print packet example

When receiving the packet, we need to understand what the Queue will return us. For this reason, in the next subsection, we will explain how to implement a simple packet processing.

```
/**
 * @brief Print the counter of the packet
 *
 * @param data
 */
void printPacket(dataPacket* data) {
  Log.verbose(F("Hello Counter received n %X" CR), data->counter);
}

/**
 * @brief Iterate through the payload of the packet and print the counter of the packet
 *
 * @param packet
 */
void printDataPacket(LoraMesher::packet<dataPacket>* packet) {
  //Get the payload to iterate through it
  dataPacket* packets = radio->getPayload(packet);

  for (size_t i = 0; i < radio->getPayloadLength(packet); i++) {
    //Print the packet
    printPacket(&packets[i]);
  }

  //Delete the packet. It is important to delete the packet.
  delete packet;
}
```

1. After receiving the packet in the `processReceivedPackets()` function, we call the `printDataPacket()` function.
2. We need to get the payload of the packet, as we have different types of packet. The payload is not always in the same memory position. We have the function `radio->getPayload(packet)` that will return a pointer to the payload.
3. We iterate through the `radio->getPayloadLength(packet)`. This will let us know how big the payload is, in dataPackets types, for a given packet. In our case, we always send only one dataPacket.
4. Get the payload and call the `printPacket()` function, that will print the counter received.
5. Lastly, we need to delete the packet.