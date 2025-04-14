# edu-zero-cookbook

## Instructions

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

install(TARGETS net_client DESTINATION bin)
install(FILES net_client.service DESTINATION lib/systemd/system)
EOF
```

```bash
cat > ./src/CMakeLists.txt << EOF
add_executable(net_client main.cpp)

target_link_libraries(net_client PRIVATE CLI11::CLI11)
EOF
```

## Net Client

```bash
cat > ./src/main.cpp << EOF
#include <CLI/CLI.hpp>
#include <iostream>
#include <string>
#include <cstring>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <thread>
#include <chrono>

void send_udp(const std::string& ip, int port, const std::string& msg) {
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) { perror("socket"); return; }

    sockaddr_in server{};
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server.sin_addr);

    sendto(sock, msg.c_str(), msg.size(), 0, (sockaddr*)&server, sizeof(server));
    std::cout << "Sent UDP: " << msg << "\n";
    close(sock);
}

void send_tcp(const std::string& ip, int port, const std::string& msg) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); return; }

    sockaddr_in server{};
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server.sin_addr);

    if (connect(sock, (sockaddr*)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sock);
        return;
    }

    send(sock, msg.c_str(), msg.size(), 0);
    std::cout << "Sent TCP: " << msg << "\n";
    close(sock);
}

void send_http(const std::string& ip, int port, const std::string& msg) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); return; }

    sockaddr_in server{};
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server.sin_addr);

    if (connect(sock, (sockaddr*)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sock);
        return;
    }

    std::string body = "{\"message\":\"" + msg + "\"}";
    std::string request =
        "POST / HTTP/1.1\r\n"
        "Host: " + ip + "\r\n"
        "Content-Type: application/json\r\n"
        "Content-Length: " + std::to_string(body.size()) + "\r\n"
        "Connection: close\r\n\r\n" + body;

    send(sock, request.c_str(), request.size(), 0);
    std::cout << "Sent HTTP POST: " << msg << "\n";
    close(sock);
}

int main(int argc, char** argv) {
    std::string protocol, ip, base_message;
    int port;

    CLI::App app{"Repeating Protocol Client"};
    app.add_option("--protocol", protocol, "tcp | udp | http")->required();
    app.add_option("--ip", ip, "Target IP")->required();
    app.add_option("--port", port, "Target Port")->required();
    app.add_option("--message", base_message, "Base message to send")->default_val("hello");

    CLI11_PARSE(app, argc, argv);

    int count = 1;
    while (true) {
        std::string message = base_message + " " + std::to_string(count++);

        if (protocol == "udp") send_udp(ip, port, message);
        else if (protocol == "tcp") send_tcp(ip, port, message);
        else if (protocol == "http") send_http(ip, port, message);
        else {
            std::cerr << "Unknown protocol: " << protocol << "\n";
            return 1;
        }

        std::this_thread::sleep_for(std::chrono::seconds(5));
    }

    return 0;
}
EOF
```

```bash
cat > net_client.service << EOF
[Unit]
Description=Repeating Network Client
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/client --protocol udp --ip 127.0.0.1 --port 9001 --message hello
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## Build

```
cmake -B build
sudo make -C build install
```

## Test

```bash
net_client --protocol udp --ip 127.0.0.1 --port 9001 --message "ping"
```

## Service

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable net_client
sudo systemctl start net_client
##
sudo systemctl status client
journalctl -u net_client -f
```

