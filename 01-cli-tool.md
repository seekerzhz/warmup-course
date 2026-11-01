# 轮子 1 · 命令行笔记工具

## 一个真实的失败

你在 `main.cpp` 里写完了 `note` 的全部逻辑：加笔记、列笔记、按关键字搜、按 id 删。一共 600 多行，一个 `switch` 管四个子命令，函数名从 `add` 到 `add2` 到 `add_real`。编译通过，跑起来也对。

然后你发现搜索有个 bug：搜 `abc` 的时候会把 `abcdef` 也算命中，本该是子串匹配，结果变成了前缀匹配。

你去改。改完之后想验证，得先 `./note add "abcdef"`、再 `./note add "abc"`、再 `./note search abc`，看两行输出对不对。改一次逻辑，重跑四遍手工命令。第三次改的时候你手滑把 `add` 改成写入了两条，于是你以为修好了。

这就是这一轮真正要解决的东西。不是「怎么把一个字符串塞进 vector」，那个你会。是**怎么把程序拆到能改得动、能重跑、构建一条命令就完事**。

## 竞赛生映射

你已经会的（占比比你想的高）：

- 字符串处理、`std::string` 的 `find` / `substr`，用于搜索
- 数组 / `vector` 的遍历、删除元素（注意 `erase` 之后迭代器失效）
- 从 `cin` 读一行、按空格切分 —— 这和命令行参数解析几乎是同一件事

真正的新东西只有两件：

1. **一个程序 = 多个源文件 + 一个构建脚本**。你现在写的每个程序都是单文件、`g++ a.cpp -o a`。从这一轮开始，文件会变多，编译命令会变长到记不住，所以要交给 CMake 管。
2. **程序的输入来自命令行参数，不是标准输入**。`argv` 是别人（shell）递给你的字符串数组，你得自己校验它合不合法、不合法就报错退出。

## 你要做的东西

一个叫 `note` 的可执行文件，四个子命令，数据先放内存（这一轮不做持久化，那是轮子 2）：

```
./note add "买牛奶"
./note list
./note search 牛奶
./note delete 3
```

要求：`add` 打印新笔记的 id；`list` 每行一条；`search` 输出命中的条目；`delete` 删掉对应 id，删不存在的 id 要报错。参数不够、子命令不认识、id 不是数字，都要返回非 0 退出码并往 stderr 打一行提示。

不要一开始就拆文件。先在一个 `main.cpp` 里写通，跑起来，再拆。拆的过程本身就是这一轮的训练内容 —— 你会第一次遇到 `undefined reference`，也会第一次意识到「头文件里放什么」是个需要判断的问题。

### 建议的拆分（拆完再对照，别提前照抄）

- `note.hpp` / `note.cpp` —— 笔记的数据结构和增删查（纯逻辑，不碰 `argv`，不打印）
- `main.cpp` —— 解析 `argv`，分发子命令，负责打印和退出码

这个边界的意义：`note.cpp` 里不该出现 `std::cout`，`main.cpp` 里不该出现 `vector`。分得干净，下一轮加持久化的时候你只需要动 `note.cpp`。

## 真正的新东西

### 1. 声明和定义是两回事

头文件里写**声明**（长什么样），`.cpp` 里写**定义**（怎么做的）。链接器的工作是把「谁调用了什么」和「谁实现了什么」对上。对不上就报 `undefined reference to ...`。

一个常见误解：`#include "note.hpp"` 不等于把 `note.cpp` 编进来了。`#include` 是文本粘贴，只粘贴声明。实现必须靠构建脚本把 `note.cpp` 也编译进去，否则链接器找不到它。

### 2. CMakeLists 是最小构建脚本

你只需要记住它做的三件事：告诉 CMake 需要什么 C++ 标准、有哪些源文件、生成什么名字的可执行文件。

```cmake
cmake_minimum_required(VERSION 3.16)
project(note CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -g")

add_executable(note
    main.cpp
    note.cpp
)
```

用法固定三步：

```bash
cmake -S . -B build      # 配置，只在改了 CMakeLists 之后需要重跑
cmake --build build -j   # 编译，改了代码只需要这一步
./build/note list
```

`add_executable` 里漏了 `note.cpp`，症状就是编译期一声不响、链接期 `undefined reference`。这不是玄学，是名单没报全。

### 3. `argc` 和 `argv`

```cpp
int main(int argc, char** argv) {
    // argv[0] 是程序自己的名字，参数从 argv[1] 开始
    // argc 最小是 1：光敲 ./note 不加参数，argc 就是 1
}
```

两个必须记住的边界：`argc == 1` 时**不能**碰 `argv[1]`，那是越界；`argv[argc]` 保证是 `nullptr`，可以拿来当循环终止条件。参数里的中文是 UTF-8 字节串，`std::string` 直接接住就行，不用特殊处理。

### 4. 子命令分发

不要写一长串 `if/else` 套 `strcmp`。先把子命令名和参数列表取出来，再做分派 —— 分派表可以是 `if` 链，也可以是 `map<string, function>`，但这个决定要在你写完第一版之后再考虑，别为了「优雅」提前上抽象。

## 完成标志

做完这四件，这个轮子才算过：

1. 删掉 `build/` 目录，从零跑 `cmake -S . -B build && cmake --build build` 能成功，无警告
2. 四个子命令都能跑：`add` 两次之后 `list` 有两行，`search` 能命中子串，`delete` 之后 `list` 少一行
3. 错误输入返回非 0：`./note`（无参数）、`./note foo`（未知子命令）、`./note delete abc`（id 非数字）都要 `echo $?` 得到非 0
4. 你手动改 `note.cpp` 里的一个函数签名，重新只跑 `cmake --build build`，能看出错误来自哪个文件哪一行（这是 `-g` 和分文件带来的好处）

## 常见卡点

**`undefined reference to 'xxx'`，编译过了链接挂**
→ 声明了没实现，或者实现了但没进 `add_executable` 的名单
→ 先 `grep` 一下这个函数在哪个 `.cpp` 里有没有定义；有定义就检查 CMakeLists 名单，改完重跑 `cmake -S . -B build`（只改 `.cpp` 不用重跑，改 CMakeLists 必须重跑）

**`fatal error: note.hpp: No such file or directory`**
→ 头文件不在编译器找的路径里
→ 如果头文件和 `main.cpp` 同目录，用 `#include "note.hpp"`（双引号，先找当前目录）；用了尖括号就会去系统路径找

**程序不报错但什么都不输出，或者 `list` 输出乱码**
→ `argc` 边界没判，读到了 `argv[1]` 之外的内存
→ 在 `main` 第一行把 `argc` 和每个 `argv[i]` 打出来看一眼；顺便确认 `list` 里的 `printf("%s")` 没有配 `std::string`（要用 `.c_str()`）

**`delete` 之后 `list` 少了两条，或者直接崩**
→ `vector::erase` 之后继续用旧迭代器，或者边遍历边删
→ 改成「先记下要删的下标，遍历结束再 `erase`」，或者用 `std::remove_if` + `erase`

**改了代码但行为没变**
→ 编的是 `build/note`，跑的是 `./note`（旧的可执行文件），或者反过来
→ 统一用 `./build/note`，并且把构建命令写进一个 `run.sh` 里

## 对应 OJ 练习

这一轮的业务逻辑大部分不适合抽成纯函数，但有两块可以：

1. **子串搜索**：给定 N 个字符串和一个模式串，输出所有包含该模式的字符串的下标。这是 `std::string::find` 的直接练习，注意重叠匹配（`aaa` 里搜 `aa` 应该命中两次）。
2. **命令行参数解析**：给定一行 shell 命令（含引号），切分出 argv 数组并输出每个参数。这是分词，用状态机写，是 `argc/argv` 的真正内核。

HydroOJ 上如果有对应题库，直接搜「字符串匹配」「分词」；没有题库就自测 —— 自己准备 3 组输入输出，跑通算过，跑不通就说明边界没考虑全。
