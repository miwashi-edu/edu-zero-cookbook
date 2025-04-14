# edu-zero-cookbook

## Instructions

```bash
git clone
https://github.com/miwashi-edu/edu-zero-cookbook
```

## CMakeLists.txt

```
cat > CMakeLists.txt << EOF
cmake_minimum_required(VERSION 3.16)
project(myproject LANGUAGES CXX C)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(src)

install(TARGETS hello1 hello2 DESTINATION bin) # Added install target
EOF
```

```bash
cat > ./src/CMakeLists.txt << EOF
add_executable(tcp_client tcp_client.c)
add_executable(udp_client udp_client.c)
add_executable(http_client http_client.c)
EOF
```

## TCP Client

```bash
cat > ./src/tcp_client.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main(int argc, char *argv[]) {
    if (argc != 5 || strcmp(argv[1], "--ip") != 0 || strcmp(argv[3], "--port") != 0) {
        fprintf(stderr, "Usage: %s --ip <ip> --port <port>\n", argv[0]);
        return 1;
    }

    const char *ip = argv[2];
    int port = atoi(argv[4]);

    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    struct sockaddr_in server_addr = {
        .sin_family = AF_INET,
        .sin_port = htons(port)
    };
    inet_pton(AF_INET, ip, &server_addr.sin_addr);

    const char *message = "udp hello";
    if (sendto(sock, message, strlen(message), 0, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("sendto");
        close(sock);
        return 1;
    }

    printf("Sent UDP message: %s\n", message);
    close(sock);
    return 0;
}
EOF
```

## UDP Client

```bash
cat > ./src/udp_client.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main(int argc, char *argv[]) {
    if (argc != 5 || strcmp(argv[1], "--ip") != 0 || strcmp(argv[3], "--port") != 0) {
        fprintf(stderr, "Usage: %s --ip <ip> --port <port>\n", argv[0]);
        return 1;
    }

    const char *ip = argv[2];
    int port = atoi(argv[4]);

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    struct sockaddr_in server_addr = {
        .sin_family = AF_INET,
        .sin_port = htons(port)
    };
    inet_pton(AF_INET, ip, &server_addr.sin_addr);

    if (connect(sock, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sock);
        return 1;
    }

    const char *message = "tcp hello";
    if (send(sock, message, strlen(message), 0) < 0) {
        perror("send");
        close(sock);
        return 1;
    }

    printf("Sent TCP message: %s\n", message);
    close(sock);
    return 0;
}
EOF
```

## UDP Client

```bash
cat > ./src/http_client.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <netdb.h>
#include <arpa/inet.h>

int main(int argc, char *argv[]) {
    if (argc != 5 || strcmp(argv[1], "--ip") != 0 || strcmp(argv[3], "--port") != 0) {
        fprintf(stderr, "Usage: %s --ip <ip> --port <port>\n", argv[0]);
        return 1;
    }

    const char *ip = argv[2];
    int port = atoi(argv[4]);

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    struct sockaddr_in server_addr = {
        .sin_family = AF_INET,
        .sin_port = htons(port)
    };
    inet_pton(AF_INET, ip, &server_addr.sin_addr);

    if (connect(sock, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sock);
        return 1;
    }

    const char *body = "{\"message\":\"http hello\"}";
    char request[1024];
    snprintf(request, sizeof(request),
             "POST / HTTP/1.1\r\n"
             "Host: %s\r\n"
             "Content-Type: application/json\r\n"
             "Content-Length: %ld\r\n"
             "Connection: close\r\n"
             "\r\n"
             "%s",
             ip, strlen(body), body);

    if (send(sock, request, strlen(request), 0) < 0) {
        perror("send");
        close(sock);
        return 1;
    }

    printf("Sent HTTP POST:\n%s\n", request);
    close(sock);
    return 0;
}
EOF
```
