使用 C/C++ 开发 `SMBTunnel` 工具可以显著减小编译后的程序大小，并且可以更好地控制性能和资源使用。为了实现这一目标，我们将使用以下库：

1. **libsmbclient**：用于 SMB 协议的实现。
2. **OpenSSL**：用于加密和解密。
3. **libconfig**：用于配置文件解析。
4. **spdlog**：用于日志记录。

### 1. 安装依赖

首先，确保你已经安装了所需的库。你可以使用包管理器来安装这些库。例如，在 Ubuntu 上，你可以使用以下命令：

```sh
sudo apt-get update
sudo apt-get install libsmbclient-dev libssl-dev libconfig-dev libspdlog-dev
```

### 2. 配置文件 (`config.cfg`)

创建一个配置文件 `config.cfg`，内容如下：

```ini
server = {
    address = "192.168.1.100";
    port = 445;
    username = "admin";
    password = "password";
};

tunnel = {
    type = "tcp";  # 可选值: "tcp", "udp"
    local_address = "127.0.0.1:1080";
    remote_address = "192.168.1.200:80";
};

encryption = {
    enabled = true;
    algorithm = "aes-256";
};

logging = {
    level = "info";
    file = "smbtunnel.log";
};
```

### 3. 主模块 (`main.cpp`)

创建 `main.cpp` 文件，内容如下：

```cpp
#include <iostream>
#include <string>
#include <thread>
#include <signal.h>
#include <spdlog/spdlog.h>
#include <libconfig.h++>
#include <openssl/evp.h>
#include <libsmbclient.h>

using namespace std;
namespace cfg = libconfig;

// 全局变量
cfg::Config config;
spdlog::logger* logger;
bool running = true;

// 配置加载函数
void loadConfig(const string& filename) {
    try {
        config.readFile(filename.c_str());
    } catch (const cfg::FileIOException& fioex) {
        cerr << "I/O error while reading file." << endl;
        exit(EXIT_FAILURE);
    } catch (const cfg::ParseException& pex) {
        cerr << "Parse error at " << pex.getFile() << ":" << pex.getLine()
             << " - " << pex.getError() << endl;
        exit(EXIT_FAILURE);
    }
}

// 日志初始化函数
void initLogger(const string& levelStr, const string& logFile) {
    auto loggerPtr = spdlog::basic_logger_mt("smbtunnel", logFile);
    if (levelStr == "debug") {
        loggerPtr->set_level(spdlog::level::debug);
    } else if (levelStr == "info") {
        loggerPtr->set_level(spdlog::level::info);
    } else if (levelStr == "warning") {
        loggerPtr->set_level(spdlog::level::warn);
    } else if (levelStr == "error") {
        loggerPtr->set_level(spdlog::level::err);
    } else {
        loggerPtr->set_level(spdlog::level::info);
    }
    logger = loggerPtr.get();
}

// 信号处理函数
void signalHandler(int signum) {
    logger->info("Shutting down SMBTunnel...");
    running = false;
}

// SMB 客户端类
class SMBClient {
public:
    SMBClient(const string& address, int port, const string& username, const string& password) {
        string url = "smb://" + address + ":" + to_string(port);
        int rc = smbclient_init(NULL, 0);
        if (rc != 0) {
            throw runtime_error("Failed to initialize SMB client");
        }

        rc = smbclient_set_context(NULL);
        if (rc != 0) {
            throw runtime_error("Failed to set SMB context");
        }

        smbclient_auth_fn auth_callback = [](const char* server, const char* share, char* workgroup, int wgmaxlen, char* username, int unmaxlen, char* password, int pwmaxlen) -> int {
            strncpy(workgroup, "WORKGROUP", wgmaxlen);
            strncpy(username, "admin", unmaxlen);
            strncpy(password, "password", pwmaxlen);
            return 0;
        };

        smbclient_set_auth_fn(auth_callback);

        rc = smbclient_open(url.c_str(), O_RDONLY, 0666);
        if (rc < 0) {
            throw runtime_error("Failed to connect to SMB server");
        }
    }

    ~SMBClient() {
        smbclient_closeall();
    }

    void send(const string& data) {
        // 实现 SMB 数据发送逻辑
    }

    string receive() {
        // 实现 SMB 数据接收逻辑
        return "";
    }
};

// 隧道类
class Tunnel {
public:
    Tunnel(const string& type, const string& localAddr, const string& remoteAddr, SMBClient& smbClient, bool encryptEnabled, const string& algorithm)
        : type(type), localAddr(localAddr), remoteAddr(remoteAddr), smbClient(smbClient), encryptEnabled(encryptEnabled), algorithm(algorithm) {
        if (encryptEnabled) {
            key = "0123456789abcdef0123456789abcdef"; // 示例密钥，实际应用中应使用更安全的方式生成
            iv = "0123456789abcdef"; // 示例 IV，实际应用中应使用更安全的方式生成
        }
    }

    void start() {
        listener = new TCPServer(localAddr);
        listener->start([this](TCPSocket* clientSocket) {
            this->handleConnection(clientSocket);
        });
    }

    void stop() {
        delete listener;
    }

private:
    string type;
    string localAddr;
    string remoteAddr;
    SMBClient& smbClient;
    bool encryptEnabled;
    string algorithm;
    string key;
    string iv;
    TCPServer* listener;

    void handleConnection(TCPSocket* localSocket) {
        TCPSocket* remoteSocket = new TCPSocket(remoteAddr);
        if (!remoteSocket->connect()) {
            logger->error("Failed to connect to remote server");
            delete localSocket;
            delete remoteSocket;
            return;
        }

        logger->info("Established connection from {} to {}", localSocket->getPeerAddress(), remoteAddr);

        thread t1([this, localSocket, remoteSocket]() {
            this->copyData(localSocket, remoteSocket);
        });

        thread t2([this, localSocket, remoteSocket]() {
            this->copyData(remoteSocket, localSocket);
        });

        t1.join();
        t2.join();

        delete localSocket;
        delete remoteSocket;
    }

    void copyData(TCPSocket* src, TCPSocket* dst) {
        char buffer[4096];
        while (running) {
            int bytesRead = src->read(buffer, sizeof(buffer));
            if (bytesRead <= 0) {
                break;
            }

            string data(buffer, bytesRead);
            if (encryptEnabled) {
                data = encrypt(data);
            }

            dst->write(data.c_str(), data.size());
        }
    }

    string encrypt(const string& data) {
        EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
        if (!ctx) {
            throw runtime_error("Failed to create EVP_CIPHER_CTX");
        }

        if (EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, (unsigned char*)key.c_str(), (unsigned char*)iv.c_str()) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw runtime_error("Failed to initialize encryption");
        }

        int len;
        int ciphertext_len;
        unsigned char ciphertext[4096 + EVP_MAX_BLOCK_LENGTH];

        if (EVP_EncryptUpdate(ctx, ciphertext, &len, (unsigned char*)data.c_str(), data.size()) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw runtime_error("Failed to encrypt data");
        }
        ciphertext_len = len;

        if (EVP_EncryptFinal_ex(ctx, ciphertext + len, &len) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw runtime_error("Failed to finalize encryption");
        }
        ciphertext_len += len;

        EVP_CIPHER_CTX_free(ctx);

        return string((char*)ciphertext, ciphertext_len);
    }

    string decrypt(const string& data) {
        EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
        if (!ctx) {
            throw runtime_error("Failed to create EVP_CIPHER_CTX");
        }

        if (EVP_DecryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, (unsigned char*)key.c_str(), (unsigned char*)iv.c_str()) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw runtime_error("Failed to initialize decryption");
        }

        int len;
        int plaintext_len;
        unsigned char plaintext[4096];

        if (EVP_DecryptUpdate(ctx, plaintext, &len, (unsigned char*)data.c_str(), data.size()) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw runtime_error("Failed to decrypt data");
        }
        plaintext_len = len;

        if (EVP_DecryptFinal_ex(ctx, plaintext + len, &len) != 1) {
            EVP_CIPHER_CTX_free(ctx);
            throw runtime_error("Failed to finalize decryption");
        }
        plaintext_len += len;

        EVP_CIPHER_CTX_free(ctx);

        return string((char*)plaintext, plaintext_len);
    }
};

// TCP 套接字类
class TCPSocket {
public:
    TCPSocket(const string& address) {
        addrinfo hints = {};
        hints.ai_family = AF_UNSPEC;
        hints.ai_socktype = SOCK_STREAM;
        hints.ai_protocol = IPPROTO_TCP;

        int rc = getaddrinfo(address.c_str(), NULL, &hints, &res);
        if (rc != 0) {
            throw runtime_error("Failed to resolve address");
        }

        sock = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
        if (sock < 0) {
            freeaddrinfo(res);
            throw runtime_error("Failed to create socket");
        }

        rc = bind(sock, res->ai_addr, res->ai_addrlen);
        if (rc < 0) {
            close(sock);
            freeaddrinfo(res);
            throw runtime_error("Failed to bind socket");
        }
    }

    TCPSocket(const string& address, int port) {
        addrinfo hints = {};
        hints.ai_family = AF_UNSPEC;
        hints.ai_socktype = SOCK_STREAM;
        hints.ai_protocol = IPPROTO_TCP;

        string portStr = to_string(port);
        int rc = getaddrinfo(address.c_str(), portStr.c_str(), &hints, &res);
        if (rc != 0) {
            throw runtime_error("Failed to resolve address");
        }

        sock = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
        if (sock < 0) {
            freeaddrinfo(res);
            throw runtime_error("Failed to create socket");
        }
    }

    ~TCPSocket() {
        close(sock);
        freeaddrinfo(res);
    }

    bool connect() {
        int rc = ::connect(sock, res->ai_addr, res->ai_addrlen);
        if (rc < 0) {
            return false;
        }
        return true;
    }

    bool listen(int backlog) {
        int rc = ::listen(sock, backlog);
        if (rc < 0) {
            return false;
        }
        return true;
    }

    TCPSocket* accept() {
        sockaddr_storage peerAddr;
        socklen_t peerAddrLen = sizeof(peerAddr);
        int peerSock = ::accept(sock, (struct sockaddr*)&peerAddr, &peerAddrLen);
        if (peerSock < 0) {
            return nullptr;
        }
        return new TCPSocket(peerSock, peerAddr);
    }

    string getPeerAddress() {
        char ipstr[INET6_ADDRSTRLEN];
        inet_ntop(AF_INET, &(((sockaddr_in*)res->ai_addr)->sin_addr), ipstr, sizeof(ipstr));
        return string(ipstr);
    }

    int read(void* buffer, size_t length) {
        return ::recv(sock, buffer, length, 0);
    }

    int write(const void* buffer, size_t length) {
        return ::send(sock, buffer, length, 0);
    }

private:
    int sock;
    addrinfo* res;

    TCPSocket(int sock, sockaddr_storage& peerAddr) : sock(sock) {
        res = nullptr;
    }
};

// TCP 服务器类
class TCPServer {
public:
    TCPServer(const string& address) {
        serverSocket = new TCPSocket(address);
        serverSocket->listen(10);
    }

    ~TCPServer() {
        delete serverSocket;
    }

    void start(function<void(TCPSocket*)> callback) {
        while (running) {
            TCPSocket* clientSocket = serverSocket->accept();
            if (clientSocket) {
                callback(clientSocket);
            }
        }
    }

private:
    TCPSocket* serverSocket;
};

int main(int argc, char* argv[]) {
    if (argc < 2) {
        cerr << "Usage: " << argv[0] << " <config_file>" << endl;
        return EXIT_FAILURE;
    }

    string configFilename = argv[1];
    loadConfig(configFilename);

    const cfg::Setting& serverSetting = config.lookup("server");
    string serverAddress = serverSetting["address"];
    int serverPort = serverSetting["port"];
    string username = serverSetting["username"];
    string password = serverSetting["password"];

    const cfg::Setting& tunnelSetting = config.lookup("tunnel");
    string tunnelType = tunnelSetting["type"];
    string localAddress = tunnelSetting["local_address"];
    string remoteAddress = tunnelSetting["remote_address"];

    const cfg::Setting& encryptionSetting = config.lookup("encryption");
    bool encryptEnabled = encryptionSetting["enabled"];
    string algorithm = encryptionSetting["algorithm"];

    const cfg::Setting& loggingSetting = config.lookup("logging");
    string logLevel = loggingSetting["level"];
    string logFile = loggingSetting["file"];

    initLogger(logLevel, logFile);

    logger->info("Starting SMBTunnel...");

    try {
        SMBClient smbClient(serverAddress, serverPort, username, password);
        Tunnel tunnel(tunnelType, localAddress, remoteAddress, smbClient, encryptEnabled, algorithm);
        tunnel.start();
    } catch (const exception& e) {
        logger->error("Failed to start SMBTunnel: {}", e.what());
        return EXIT_FAILURE;
    }

    signal(SIGINT, signalHandler);
    signal(SIGTERM, signalHandler);

    while (running) {
        this_thread::sleep_for(chrono::seconds(1));
    }

    logger->info("SMBTunnel stopped.");
    return EXIT_SUCCESS;
}
```

### 4. 编译和运行

确保你已经安装了所有依赖库，然后在项目根目录下创建一个 `Makefile`，内容如下：

```makefile
CXX=g++
CXXFLAGS=-std=c++11 -Wall -Wextra -O2
LDFLAGS=-lsmbclient -lssl -lcrypto -lconfig++ -lspdlog

TARGET=smbtunnel

all: $(TARGET)

$(TARGET): main.o
	$(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)

main.o: main.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

clean:
	rm -f *.o $(TARGET)
```

然后运行以下命令来编译和运行 `SMBTunnel`：

```sh
make
./smbtunnel config.cfg
```

### 5. 注意事项

1. **密钥管理**：示例代码中使用了一个固定的密钥和 IV。在实际应用中，应使用更安全的方式生成和管理密钥。
2. **错误处理**：示例代码中的错误处理相对简单，实际应用中应更详细地处理各种可能的错误情况。
3. **性能优化**：示例代码中的加密和解密逻辑可以进一步优化，以提高性能。
4. **安全性**：确保所有敏感信息（如密码）都经过适当的安全处理。
5. **SMB 客户端实现**：示例代码中的 SMB 客户端实现非常基础，实际应用中可能需要更复杂的逻辑来处理 SMB 协议的细节。

这个示例代码提供了一个基本的 `SMBTunnel` 工具框架，你可以在此基础上进一步扩展和优化，以满足更多的功能需求。请注意，`libsmbclient` 库主要用于文件系统级别的操作，而不是直接用于建立隧道。因此，你需要根据实际需求调整 SMB 客户端的实现。
