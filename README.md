# edu-zero-cookbook

## Instructions

> TCP Lyssnare `nc -l -p 9000`
> UDP Lyssnare `nc -u -l -p 9001`

```bash
git clone
https://github.com/miwashi-edu/edu-zero-cookbook
```

## CMakeLists.txt

> If typing, don't type escape characters '\'.

```
cat > CMakeLists.txt << EOF
cmake_minimum_required(VERSION 3.16)
project(myproject LANGUAGES CXX C)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY \${CMAKE_SOURCE_DIR}/bin)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

FetchContent_Declare(
  CLI11
  GIT_REPOSITORY https://github.com/CLIUtils/CLI11.git
  GIT_TAG v2.3.2
)

FetchContent_MakeAvailable(CLI11)

add_subdirectory(src)

install(TARGETS tcp_client udp_client http_client DESTINATION bin)
EOF
```

```bash
cat > ./src/CMakeLists.txt << EOF
add_executable(udp_client udp_client.cpp)
add_executable(tcp_client tcp_client.cpp)
add_executable(http_client http_client.cpp)

target_link_libraries(udp_client PRIVATE CLI11::CLI11)
target_link_libraries(tcp_client PRIVATE CLI11::CLI11)
target_link_libraries(http_client PRIVATE CLI11::CLI11)
EOF
```

## TCP Client

```bash
cat > ./src/tcp_client.cpp << EOF
#include <CLI/CLI.hpp>
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <arpa/inet.h>

int main(int argc, char** argv) {
    std::string ip;
    int port;

    CLI::App app{"TCP Hello Sender"};
    app.add_option("--ip", ip, "Target IP")->required();
    app.add_option("--port", port, "Target Port")->required();
    CLI11_PARSE(app, argc, argv);

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server_addr.sin_addr);

    if (connect(sock, (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sock);
        return 1;
    }

    const char* message = "tcp hello";
    if (send(sock, message, strlen(message), 0) < 0) {
        perror("send");
        close(sock);
        return 1;
    }

    std::cout << "Sent TCP message: " << message << "\n";
    close(sock);
    return 0;
}
EOF
```

## UDP Client

```bash
cat > ./src/udp_client.cpp << EOF
#include <CLI/CLI.hpp>
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <arpa/inet.h>

int main(int argc, char** argv) {
    std::string ip;
    int port;

    CLI::App app{"UDP Hello Sender"};
    app.add_option("--ip", ip, "Target IP")->required();
    app.add_option("--port", port, "Target Port")->required();
    CLI11_PARSE(app, argc, argv);

    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server_addr.sin_addr);

    const char* message = "udp hello";
    if (sendto(sock, message, strlen(message), 0,
               (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("sendto");
        close(sock);
        return 1;
    }

    std::cout << "Sent UDP message: " << message << "\n";
    close(sock);
    return 0;
}
EOF
```

## UDP Client

```bash
cat > ./src/http_client.cpp << EOF
#include <CLI/CLI.hpp>
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <netdb.h>
#include <arpa/inet.h>

int main(int argc, char** argv) {
    std::string ip;
    int port;

    CLI::App app{"HTTP Hello Poster"};
    app.add_option("--ip", ip, "Target IP")->required();
    app.add_option("--port", port, "Target Port")->required();
    CLI11_PARSE(app, argc, argv);

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server_addr.sin_addr);

    if (connect(sock, (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sock);
        return 1;
    }

    const char* body = "{\"message\":\"http hello\"}";
    char request[1024];
    snprintf(request, sizeof(request),
             "POST / HTTP/1.1\r\n"
             "Host: %s\r\n"
             "Content-Type: application/json\r\n"
             "Content-Length: %ld\r\n"
             "Connection: close\r\n"
             "\r\n"
             "%s",
             ip.c_str(), strlen(body), body);

    if (send(sock, request, strlen(request), 0) < 0) {
        perror("send");
        close(sock);
        return 1;
    }

    std::cout << "Sent HTTP POST:\n" << request << "\n";
    close(sock);
    return 0;
}
EOF
```

## Build

```
cmake -B build
sudo make -C build install
```
