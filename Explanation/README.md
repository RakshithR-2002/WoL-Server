# Wake on Lan Code

Let's break down this C++ code for a Wake-on-LAN (WoL) server step by step:

```c++
#include <iostream>
#include <cstring>
#include <sys/socket.h> 
#include <netinet/in.h> 
#include <arpa/inet.h> 
#include <unistd.h> 

// Defines
#define BROADCAST_ADDR "255.255.255.255" // Standard broadcast address
#define WOL_PORT 9                     // Well-known port for WoL

int main(int argc, char *argv[]) {
    // 1. Argument Handling
    if (argc < 3) { 
        std::cerr << "Usage: " << argv[0] << " <MAC Address> <Broadcast IP (optional)>" << std::endl;
        return 1; // Indicate an error
    }

    // 2. Extract MAC Address
    unsigned char mac[6]; 
    if (sscanf(argv[1], "%hhx:%hhx:%hhx:%hhx:%hhx:%hhx", 
               &mac[0], &mac[1], &mac[2], &mac[3], &mac[4], &mac[5]) != 6) {
        std::cerr << "Invalid MAC address format. Use format: XX:XX:XX:XX:XX:XX" << std::endl;
        return 1; 
    }

    // 3. Determine Broadcast IP
    const char *broadcast_ip = (argc > 3) ? argv[2] : BROADCAST_ADDR; 

    // 4. Create UDP Socket
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) {
        std::cerr << "Error creating socket" << std::endl;
        return 1; 
    }

    // 5. Enable Broadcasting
    int broadcastEnable = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &broadcastEnable, sizeof(broadcastEnable));

    // 6. Construct Magic Packet
    unsigned char packet[102]; 
    std::memset(packet, 0xFF, 6);       // First 6 bytes are FF
    std::memcpy(packet + 6, mac, 6);    // Copy MAC address 16 times
    for (int i = 1; i < 16; i++) {
        std::memcpy(packet + 6 * (i + 1), mac, 6); 
    }

    // 7. Prepare Socket Address Structure
    sockaddr_in addr; 
    addr.sin_family = AF_INET;            // IPv4 address family
    addr.sin_port = htons(WOL_PORT);      // Convert port to network byte order
    inet_pton(AF_INET, broadcast_ip, &(addr.sin_addr)); // Convert IP to binary form

    // 8. Send the Magic Packet
    ssize_t bytesSent = sendto(sockfd, packet, sizeof(packet), 0, (struct sockaddr *)&addr, sizeof(addr));
    if (bytesSent != sizeof(packet)) {
        std::cerr << "Error sending magic packet" << std::endl;
    } else {
        std::cout << "Wake-on-LAN packet sent successfully!" << std::endl;
    }

    // 9. Close the Socket
    close(sockfd); 
    return 0; // Indicate success
}
```

**Explanation:**

1. **Include Headers:** Includes necessary headers for input/output, string manipulation, networking, and socket operations.

2. **Argument Handling:** Ensures the program is run with at least the MAC address as an argument. A second, optional argument can be used to specify a different broadcast IP address.

3. **Extract MAC Address:**
   - Retrieves the MAC address from the command line argument (`argv[1]`).
   - Uses `sscanf` to parse the MAC address string and store it as 6 bytes in the `mac` array.
   - Handles potential errors with an invalid MAC address format.

4. **Determine Broadcast IP:**
   - Sets the `broadcast_ip` to either the provided IP (if given as the third argument) or uses the default broadcast address (`255.255.255.255`).

5. **Create UDP Socket:**
   - `socket(AF_INET, SOCK_DGRAM, 0)` creates a UDP socket:
     - `AF_INET`: Uses IPv4 addresses.
     - `SOCK_DGRAM`: Specifies a datagram socket (for UDP).
     - `0`: Uses the default protocol for this socket type.

6. **Enable Broadcasting:**
   - `setsockopt` configures the socket to allow broadcasting:
     - `SOL_SOCKET`: Applies the option at the socket level.
     - `SO_BROADCAST`: Enables broadcasting.

7. **Construct Magic Packet:**
   - Creates a 102-byte `packet` array to hold the magic packet.
   - Sets the first 6 bytes to `0xFF`.
   - Copies the MAC address 16 times into the packet (this is the standard WoL magic packet format).

8. **Prepare Socket Address Structure:**
   - Creates a `sockaddr_in` structure to specify the destination address and port:
     - `sin_family`: Set to `AF_INET` for IPv4.
     - `sin_port`: Set to the WoL port (9) using `htons` to convert to network byte order.
     - `sin_addr`: The broadcast IP address is converted to network byte order using `inet_pton`.

9. **Send the Magic Packet:**
   - `sendto` sends the `packet` to the specified address and port:
     - If successful, it returns the number of bytes sent, which should match the size of the packet.
     - Prints an error message if sending fails.

10. **Close the Socket:**
   - `close(sockfd)` closes the UDP socket, releasing resources.

**In essence, this code crafts a specific network packet (the WoL magic packet) containing the target device's MAC address and broadcasts it on the network. When the target device, even if powered off, receives this packet, it recognizes the magic packet pattern and initiates a power-on sequence.** 

# 
