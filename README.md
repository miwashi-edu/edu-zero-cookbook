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

void send_udp(const std::string& ip, int port, const std::string& msg) {
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) { perror("socket"); exit(1); }

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
    if (sock < 0) { perror("socket"); exit(1); }

    sockaddr_in server{};
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server.sin_addr);

    if (connect(sock, (sockaddr*)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sock);
        exit(1);
    }

    send(sock, msg.c_str(), msg.size(), 0);
    std::cout << "Sent TCP: " << msg << "\n";
    close(sock);
}

void send_http(const std::string& ip, int port, const std::string& msg) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); exit(1); }

    sockaddr_in server{};
    server.sin_family = AF_INET;
    server.sin_port = htons(port);
    inet_pton(AF_INET, ip.c_str(), &server.sin_addr);

    if (connect(sock, (sockaddr*)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sock);
        exit(1);
    }

    std::string body = "{\"message\":\"" + msg + "\"}";
    std::string request =
        "POST / HTTP/1.1\r\n"
        "Host: " + ip + "\r\n"
        "Content-Type: application/json\r\n"
        "Content-Length: " + std::to_string(body.size()) + "\r\n"
        "Connection: close\r\n\r\n" + body;

    send(sock, request.c_str(), request.size(), 0);
    std::cout << "Sent HTTP POST:\n" << request << "\n";
    close(sock);
}

int main(int argc, char** argv) {
    std::string protocol, ip, message;
    int port;

    CLI::App app{"Unified Client"};
    app.add_option("--protocol", protocol, "tcp | udp | http")->required();
    app.add_option("--ip", ip, "Target IP")->required();
    app.add_option("--port", port, "Target Port")->required();
    app.add_option("--message", message, "Message to send")->default_val("hello");

    CLI11_PARSE(app, argc, argv);

    if (protocol == "udp") send_udp(ip, port, message);
    else if (protocol == "tcp") send_tcp(ip, port, message);
    else if (protocol == "http") send_http(ip, port, message);
    else {
        std::cerr << "Unknown protocol: " << protocol << "\n";
        return 1;
    }

    return 0;
}
EOF
```

## Build

```
cmake -B build
sudo make -C build install
```

## Cron

```
sudo apt install -y cron # One time only
sudo /usr/sbin/cron # Start manually as we are in docker, not zero
crontab -e # After this an editor is opened, add the following line to end
* * * * * /bin/net_client --protocol udp --ip 192.168.0.10 --port 9001 --message "hello" >> /var/log/net_client.log 2>&1
tail -f /var/log/net_client.log
```

## Test

```bash
./bin/net_client --protocol udp --ip 127.0.0.1 --port 9001 --message "hi from udp"
./bin/net_client --protocol tcp --ip 127.0.0.1 --port 9000 --message "hi from tcp"
./bin/net_client --protocol http --ip 127.0.0.1 --port 3000 --message "hi from http"
```

## Run

```bash
```

