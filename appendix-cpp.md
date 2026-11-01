# 附录：竞赛生视角的 C++ 工程速查

只放你真正会用到的四段。每段都能直接抄进项目，抄完按说明改一行就能跑。

## 1. CMake 最小模板

```cmake
cmake_minimum_required(VERSION 3.16)
project(note CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -g")

add_executable(note
    main.cpp
    note.cpp
    storage.cpp
)
```

`add_executable` 后面那个列表就是「本次编译的源文件名单」，漏一个就在链接期炸 `undefined reference`。改这个文件之后必须重跑配置（`cmake -S . -B build`），只改 `.cpp` 只需要 `cmake --build build`。`-Wall -Wextra` 别关，它抓出来的问题比你想的多；`-g` 让崩溃时能看到行号。

发布版再加一个选项，别直接改上面的 flags：

```cmake
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")
endif()
```

## 2. fstream 二进制读写

```cpp
#include <fstream>
#include <cstdint>
#include <string>

void write_record(std::ofstream& out, const std::string& s) {
    uint32_t len = static_cast<uint32_t>(s.size());
    out.write(reinterpret_cast<const char*>(&len), sizeof(len));
    out.write(s.data(), len);
}

bool read_record(std::ifstream& in, std::string& s) {
    uint32_t len = 0;
    in.read(reinterpret_cast<char*>(&len), sizeof(len));
    if (!in || len > (64u << 20)) return false;   // 读失败，或长度不合理
    s.resize(len);
    in.read(&s[0], len);
    return static_cast<bool>(in);
}
```

要点：**先存长度再存内容**，读的时候先 `resize` 再 `read` 到 `&s[0]`。`read/write` 的单位是字节，`sizeof(std::string)` 是对象大小（通常 32），跟正文长度毫无关系 —— 这是最高频的翻车点。每次 `read` 之后都要检查流状态，否则「读到一半文件结束」会被当成成功。

## 3. RAII 是什么

RAII = 把资源的生命周期绑在对象的作用域上：构造函数获取，析构函数释放。作用域一结束（包括抛异常），释放一定发生。

```cpp
class FileHandle {
    int fd_ = -1;
public:
    explicit FileHandle(const char* path) : fd_(::open(path, O_RDWR | O_CREAT, 0644)) {}
    ~FileHandle() { if (fd_ >= 0) ::close(fd_); }
    FileHandle(const FileHandle&) = delete;             // 不许拷贝，否则会 double close
    FileHandle& operator=(const FileHandle&) = delete;
    int get() const { return fd_; }
};
```

你已经在用 RAII 了，只是没意识到：`std::vector` 析构时自动 `free`、`std::lock_guard` 析构时自动解锁、`std::ifstream` 析构时自动 `close`。自己写资源类的时候照这个模式来，就永远不会忘 `close`。记住两条：**禁止拷贝**（拷贝会导致两次释放），**要么禁用移动要么正确实现移动**。

## 4. mutex 五行 + 优雅关闭骨架

```cpp
#include <mutex>
#include <shared_mutex>
#include <atomic>
#include <thread>
#include <condition_variable>

std::shared_mutex g_rw;              // 多读单写
std::atomic<bool> g_stop{false};

void reader() {
    std::shared_lock<std::shared_mutex> lk(g_rw);   // 读锁，可并发
    // 读数据
}

void writer() {
    std::unique_lock<std::shared_mutex> lk(g_rw);   // 写锁，独占
    // 改数据
}

void background() {
    std::mutex m; std::condition_variable cv;
    std::unique_lock<std::mutex> lk(m);
    while (!g_stop.load()) {
        cv.wait_for(lk, std::chrono::seconds(5), []{ return g_stop.load(); });
        if (g_stop.load()) break;
        do_compact();                               // 注意：锁外做重活
    }
}

int main() {
    std::signal(SIGTERM, [](int){ g_stop.store(true); });   // 信号里只置标志
    std::thread bg(background);
    // ... 主循环，检查 g_stop
    bg.join();                                              // 必须 join，否则 terminate
}
```

三个必须记住的：`std::thread` 析构前必须 `join()` 或 `detach()`，否则 `std::terminate`；`cv.wait` 一定要带谓词（内部是循环，防虚假唤醒和丢失唤醒）；信号处理函数里只碰 `std::atomic`，`printf` / `malloc` / 加锁一律禁止。

## 5. 编译期就能抓到的错误

```bash
# 运行时检查：越界、悬垂指针、内存泄漏（调试期必开）
cmake -S . -B build-asan -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -g -O1"

# 并发检查：数据竞争（轮子 5 必开）
cmake -S . -B build-tsan -DCMAKE_CXX_FLAGS="-fsanitize=thread -g -O1"
```

ASan 抓内存错误，UBSan 抓未定义行为（有符号溢出、空指针解引用），TSan 抓数据竞争。它们只在**实际执行到**的代码路径上报告，所以要把边界和并发路径都真的跑一遍。这三个工具是你的自动裁判 —— 比「我看了一遍觉得没问题」可靠一万倍。
