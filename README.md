# Embedded TCP/IP Project

## Purpose of this Project
The primary goal of this project is to learn and implement embedded TCP/IP communication. The focus is on gaining hands-on experience with networking concepts at the embedded system level. Additionally, Wireshark is used to analyze the network traffic, providing insights into the communication between the embedded device and other networked systems.

This project involved:
- Leveraging TCP/IP libraries to handle network communication and HTTP requests on an embedded system.
- Wiretapping HTTP traffic to explore and understand how data is processed through various layers of the network stack, from the driver layer to the application layer.
- Gained hands-on experience with embedded networking, including using industry-standard libraries for protocols like HTTP, and interacting with hardware to facilitate communication.
- This project demonstrated my ability to integrate networking protocols into embedded systems, work with TCP/IP stacks, and troubleshoot network-related issues on constrained hardware.


>I had a great time exploring a new area, which really boosted my confidence. It was definitely a huge learning experience, especially diving into the C code for the TCP/IP implementation 😅 while following RCP 1180 tutorial. It allowed me to take what I learned in school and apply it to real-world projects, giving me a deeper understanding of how everything works behind the scenes .


## Tools Used
- **STM32CubeIDE**
- **Nucleo-F756ZG**
- **Wireshark**
- **Mongoose Library**

## Overview Plan
1. Create a simple web server on a workstation to show request and response.
2. Move the web server onto the Nucleo-F756ZG board.
   - Set up skeleton firmware.
3. Run the HTTP request and capture it in Wireshark.
4. Show how every packet in the dump is handled by the server.

### Step 1. Set Up the Skeleton Firmware

1. Start Cube IDE. Choose **File** > **New** > **STM32 Project**.
2. In the "Part Number" field, type the microcontroller name (e.g., "F756ZG"). This will narrow the MCU/MPU list to a single row. Click on that row, then click **Next**.
3. In the **Project Name** field, type any name for the project and click **Finish**. Answer "Yes" if a pop-up dialog appears.
4. A configuration window appears. Click on the **Clock Configuration** tab. Find the system clock value and set it to the maximum (216 MHz for this MCU). Hit **Enter** and answer "Yes" to the auto-configuration prompt.
   
5. Switch to the **Pinout** tab, go to **Connectivity**, then enable the **USART3** controller and pins (as shown in the table). Choose **Asynchronous Mode**. This will allow communication via the built-in debugger (ST-Link) over the USB connection.
   ![Pins for Eth](https://i.ibb.co/LkKDWJJ/Capture.png)

6. Click on **Connectivity** > **ETH**, then choose **Mode** > **RMII**. Verify that the configured pins match the table above. If not, change the pins.
   ![Pins for Eth](https://i.ibb.co/xzY57WP/ethernet.png)

7. Click **Ctrl + S** to save the configuration. This generates the code and opens the `main.c` file.
8. In the `main()` function, add logging to the while loop. Insert your code between the `USER CODE` comments, as CubeIDE will preserve it during code regeneration.

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    printf("Tick: %lu\r\n", HAL_GetTick());
    HAL_Delay(500);
}

```
![In main.c](https://i.ibb.co/6vtXZ3Y/initalize-interrupt.png)

10. To redirect printf() to the UART in your STM32Cube project, you need to override the default _write() syscall function used by Newlib. 
11. Add mg_millis() function that returns a number of milliseconds since boot. This function will be required for accurate timer tracking. Add it just below the _write() function
```c
#include "main.h"

__attribute__((weak)) int _write(int file, char *ptr, int len) {
    if (file == 1 || file == 2) {
      extern UART_HandleTypeDef huart3;
      HAL_UART_Transmit(&huart3, (unsigned char *) ptr, len, 999);
    }
    return len;
  }

uint64_t mg_millis(void) {
  return HAL_GetTick();
}
```

Under usercode in main.c add this snippet. This enables ethernet interrupt and usart console message. Since this code uses printf we can now redirect it to USART under the system call section. Lastly a snippet for the timer to work with mongoose library. 

### Now, Flash the Firmware!
In your terminal, run the following command to check for a successful connection

Terminal input
```bash
cu -l /dev/cu.usbmodem21102 -s 115200
```
Expected output
```connected.
Tick: 29122
Tick: 29334
....
```
### Step 2. Network Intergration

1. To integrate Moongosee library it only takes two files [moongose.h](https://github.com/cesanta/mongoose/blob/master/mongoose.h) and [moongose.c](https://github.com/cesanta/mongoose/blob/master/mongoose.c) copy the two raw files into your working folder under src.
   
2. Now that we have the libary files we need to integrate by creatinga mongoose_config.h to your project and paste the following

```c
#pragma once
#define MG_ARCH MG_ARCH_NEWLIB     // For all ARM GCC based environments
#define MG_ENABLE_TCPIP 1          // Enables built-in TCP/IP stack
#define MG_ENABLE_CUSTOM_MILLIS 1  // We must implement mg_millis()
#define MG_ENABLE_TCPIP_PRINT_DEBUG_STATS 1  // Enable debug stats log

#define MG_ENABLE_DRIVER_STM32F 1
```
3.Complete by adding this to your main.c file under USER CODE BEGINS HERE
```c
#include "mongoose.h"
```
4. Add run_mongoose() function
```C 
// In RTOS environment, run this function in a separate task. Give it 8k stack
static void run_mongoose(void) {
  struct mg_mgr mgr;        // Mongoose event manager
  mg_mgr_init(&mgr);        // Initialise event manager
  mg_log_set(MG_LL_DEBUG);  // Set log level to debug
  for (;;) {                // Infinite event loop
    mg_mgr_poll(&mgr, 0);   // Process network events
  }
```

6. Update main() function to call run_mongoose() instead of running at infinite loop.
7. Rebuild the firmware, and flash it. Notice the log messages. You should see something like this:

```bash
7f5    1 mongoose.c:5089:onstatechange  Link up
7f9    3 mongoose.c:5189:tx_dhcp_discov DHCP discover sent. Our MAC: 02:2d:cf:46:29:04
915    3 mongoose.c:5168:tx_dhcp_reques DHCP req sent
a30    2 mongoose.c:5296:rx_dhcp_client Lease: 86400 sec (86402)
a36    2 mongoose.c:5084:onstatechange  READY, IP: 192.168.0.60
a3c    2 mongoose.c:5085:onstatechange         GW: 192.168.0.1
a42    2 mongoose.c:5086:onstatechange        MAC: 02:2d:cf:46:29:04
bcf    2 mongoose.c:5755:mg_tcpip_poll  Status: ready, IP: 192.168.0.60, rx:6, tx:3, dr:0, er:0
fb7    2 mongoose.c:5755:mg_tcpip_poll  Status: ready, IP: 192.168.0.60, rx:6, tx:3, dr:0, er:0
```

### Step 2. Simple Web Server that responds to ok to any HTTP request
In this step, an HTTP server is set up to listen on port 80 for incoming requests using the Mongoose library. The function mg_http_listen() initializes the listener with the address "http://0.0.0.0:80", allowing the server to accept requests from any IP address on the device. The event_handler function is passed as a callback to process these requests. The event_handler() function checks if the incoming event is an HTTP message (MG_EV_HTTP_MSG) and, if so, extracts the parsed HTTP request from the event data. It then sends a response back to the client with an HTTP status code of 200 (indicating success) and a message body that includes the text "ok, uptime: X", where X is the system uptime in milliseconds. This establishes a basic HTTP server that can handle and respond to client requests. 

e.g 
When you type the server's address (e.g., http://device-ip:80) into your browser, here's what happens:
**Request Sent:** Your browser sends an HTTP request to the server running on your device at port 80

**Server Listens:** The server, set up with the Mongoose library, is constantly "listening" for incoming requests on that address.

**Event Triggered:** When the server gets your request, it triggers the event_handler() function to process it.

**Response Generated:** Inside the event_handler(), the server sees it's an HTTP message and sends a reply with the message "ok, uptime: <number>". The <number> is the number of milliseconds since the server started running.

**Response Displayed:** Your browser gets this response and shows it on the screen, so you'll see something like:
```python
ok, uptime: 12345
```
### Step 2.1 Implementation of HTTP Listener and Event Handler
1. Before the event loop, add this line that creates HTTP listener with event_handler event handler function
```c
mg_http_listen(&mgr, "http://0.0.0.0:80", event_handler, NULL);
```

2. Add the event_hanler() function before run_mongoose():
```c
static void event_handler(struct mg_connection *c, int ev, void *ev_data) {
  if (ev == MG_EV_HTTP_MSG) {
    struct mg_http_message *hm = ev_data;  // Parsed HTTP request
    mg_http_reply(c, 200, "", "ok, uptime: %llu\r\n", mg_millis());
  }
}
```


**flash the firmware and open the browser and type the board's IP adress and see reponse message**

Theory
```lua
+-----------------------------+
|      Application Layer      |  <-- main.c (Your HTTP server logic)
+-----------------------------+
|         HTTP Library        |  <-- src/http.{c,h} (Handles HTTP requests)
+-----------------------------+
|       TCP/IP Stack          |  <-- src/net_builtin.{c,h} (Implements protocols)
+-----------------------------+
|      Hardware Driver        |  <-- src/drivers/stm32.{c,h} (Manages STM32 hardware)
+-----------------------------+
|         Ethernet            |  <-- Physical connection (RJ45/PHY chip)
+-----------------------------+

```
### Step 3. Analyze Each Layer With Wire Shark
The Driver Layer handles network frames (Ethernet, Wi-Fi, etc.) with link-layer headers (e.g., MAC addresses), parsing them before passing the data to the TCP/IP Stack.

The TCP/IP Stack processes transport protocols (TCP/UDP), adding or parsing transport-layer headers (e.g., source/destination ports, sequence numbers). It then forwards the data to the Library Layer.

The Library Layer parses higher-level protocols like HTTP, MQTT, or SMTP, each with its own application-layer headers (e.g., HTTP headers, message types in MQTT). Finally, the data is passed to the User Code, where the parsed content is used for application logic, with all headers removed.

Flow for Receiving:
  - Driver Layer : When data arrives, the Driver Layer strips off the link-layer header (like MAC addresses) to get to the actual data. It passes this data to the TCP/IP Stack for further processing.
  - TCP/IP Stack: The TCP/IP Stack checks the data for transport-layer headers (like source/destination ports, sequence numbers for TCP).After reading this info, it removes those headers and hands the actual data (application-level content) to the Library Layer.
  - Library Layer: The Library Layer then looks at the application-layer headers (like HTTP headers, message types for MQTT, etc.) and parses them to understand the data.
Once done, it gives the clean, parsed data to the User Code.
  - User Code: Processes the parsed data with headers stripped

So when you send data back imagine it adds on back the headers according to each layer user code prepares the data and passes to library layer which adds on application-layer headers. Then TCP/IP Stack adds the transport-layer headers (like source/destination ports and sequence numbers. Then Driver Layer adds the link-layer header (e.g., MAC address) and sends the complete packet over the network.

[Download Wireshark](https://www.wireshark.org/) dump the http request on wi-fi:en0 to capture the traffic
```lua
host 192.168.0.60
```
Add the IP address of your board. Make a new request and see analyze the dump on wireshark

![TCP/IP DUMP](https://i.ibb.co/pfgpqdF/wireshark-dump.png)
## HTTP Transaction Overview

This HTTP transaction involves multiple frames exchanged between the client and server. Here's a breakdown of the key steps:

1. **TCP Handshake**:
   - The first three frames are part of the **TCP handshake**:
     - **SYN**: The client initiates the connection.
     - **SYN-ACK**: The server acknowledges the connection request.
     - **ACK**: The client acknowledges the server's response, completing the handshake.

2. **HTTP Request and Response**:
   - After the handshake, the **curl** command sends a **GET** request to the server.
   - The server (board) replies with the requested data.

3. **TCP Teardown**:
   - The last four frames are part of the **four-way TCP teardown**:
     - This is the process used to gracefully close the connection between the client and server.

**One major point i learned from digging the code** 
```lua
#pragma once
```
C optimization is powerful for performance, but in TCP/IP networking, protocol correctness is often more important, especially when working with standardized protocols. Ensuring that your system follows the TCP/IP specification ensures reliable communication, while unnecessary optimization might sacrifice clarity or cause protocol issues.

 In many networking cases, sticking to the protocol and ensuring the system works correctly is more critical. Optimizing comes after that — and only when you see areas where performance is truly a bottleneck.

---
### Summary
The skills I've learned from setting up Mongoose TCP/IP on an embedded system are incredibly valuable, even as a beginner. 

**Networking Foundation:** I've gained hands-on experience with core networking concepts like TCP/IP and HTTP, which are essential for developing IoT and embedded systems.

**Using Libraries:** I’ve learned how to integrate and use libraries, a key skill that allows me to efficiently build complex systems without having to reinvent the wheel.

**Web Servers & IoT Communication:** By setting up an HTTP server on an embedded device, I’m now prepared to build IoT applications that require network communication.

**Low-Level Networking Understanding:** I’ve built a strong foundation in networking that helps me troubleshoot and understand how devices communicate over networks.

**Problem-Solving & Debugging** This experience has improved my ability to solve problems and use tools like Wireshark to debug network communication issues.




   
