# git_bisect_homework

Для корректной работы нужно, чтобы исполняемый файл лежал рядом с репозиторием (.git).  
Пробовал сделать через --git-dir, но по какой-то причине не срабатывал корректно git reset --hard, запущенный из программы (оставлял файлы, как будто reset не hard). Хотя в strace вызов был с правильными аргументами.  
Из консоли reset --hard с --git-dir работает корректно.

Не очень понял по заданию, как команды git checkout и test -f 'Lib/bisect.py' могут работать вместе в данном случае.  
git checkout не удаляет файлы, а всего лишь переносит head на другой коммит. Поэтому test будет всегда говорить, что файл существует (либо всегда не существует), на каком бы коммите head не находился. По этой причине заменил git checkout на git reset --hard.

Еще не понял почему нужно искать саммый ранний коммит, в котором команда возвращает 1, когда test возвращает 0 при успехе. Т. е. для test всегда нужно искать самый ранний коммит, который возвращает 0.  
А еще тест возвращает не 1, а 256, если файл не найден.

Возможно, я где-то ошибся.

```c++
#include <iostream>
#include <string>
#include <sstream>
#include <vector>
#include <algorithm>
#include <iterator>
#include <unistd.h>
#include <sys/wait.h>
#include <cstring>

using namespace std;

#define COMMIT_FROM_POS 1
#define COMMIT_TO_POS   2
#define COMMAND_POS     3


std::vector<std::string> read_hashes(int a_fd)
{
    std::ostringstream current_hash;
    std::vector<std::string> commit_hashes;

    char buf[10];
    size_t bytes_read = read(a_fd, buf, sizeof(buf));
    while (bytes_read) {

        for (size_t i = 0; i < bytes_read; ++i) {
            char symbol = buf[i];

            if (symbol != '\n') {
                current_hash << symbol;
            } else {
                commit_hashes.push_back(current_hash.str());
                current_hash.str("");
            }
        }
        bytes_read = read(a_fd, buf, sizeof(buf));
    }
    return commit_hashes;
}

int execute(std::string cmd)
{
    vector<string> tokens;
    istringstream iss(cmd);
    copy(istream_iterator<string>(iss), istream_iterator<string>(), back_inserter(tokens));

    std::vector<const char*> cmd_tokens;
    cmd_tokens.reserve(tokens.size() + 1);

    for(const auto& sp: tokens) {
        cmd_tokens.push_back(sp.c_str());
    }
    cmd_tokens.push_back(nullptr);

    if (fork() == 0) {
        execvp(cmd_tokens[0], (char**)cmd_tokens.data());
    }

    int status;
    wait(&status);

    return status;
}

int main(int argc, char* argv[])
{
    if (argc < 4) {
        cout << "This app need 3 input arguments" << endl;
        return 0;
    }
    const char* commit_from = argv[COMMIT_FROM_POS];
    const char* commit_to = argv[COMMIT_TO_POS];
    const char* cmd = argv[COMMAND_POS];

//    const char* commit_from = "a43111118ff4e5ccc88e3307b998cb9e983207ed";
//    const char* commit_to = "3d1e146086ec308cb14416ed606bf6160e569878";
//    const char* cmd = "test -f 'Lib/bisect.py'";
//    Must match 4e16098ce74c645cf1d69566b6f8bc96031554b7

    // Получаем список коммитов
    int fd_to_parent[2];
    pipe(fd_to_parent);

    if (fork() == 0) {
        close(fd_to_parent[0]);
        dup2(fd_to_parent[1], 1);

        string commits_range = string(commit_from) + ".." + string(commit_to);
        return execlp("git", "git", "rev-list", "--ancestry-path", commits_range.c_str(), nullptr);
    }
    close(fd_to_parent[1]);

    std::vector<std::string> commit_hashes = read_hashes(fd_to_parent[0]);
    std::reverse(commit_hashes.begin(), commit_hashes.end());

    wait(NULL);

    int left = -1;
    int right = commit_hashes.size() - 1;

    while (left + 1 != right) {
        int mid = (right + left) / 2;

        execute("git reset --hard --quiet " + commit_hashes[mid]);

        if (execute(cmd) == 0) {
            // Успешное выполнение
            right = mid;
        } else {
            left = mid;
        }
    }

    execute("git reset --hard --quiet " + commit_hashes[right]);
    if (execute(cmd) == 0) {
        cout << "Commit " << commit_hashes[right] << " matched" << endl;
    } else {
        cout << "No commits matched" << endl;
    }

    return 0;
}
```

Пример запуска  
```console
./git_bisect a43111118ff4e5ccc88e3307b998cb9e983207ed 3d1e146086ec308cb14416ed606bf6160e569878 "test -f ./Lib/bisect.py"
Commit 4e16098ce74c645cf1d69566b6f8bc96031554b7 matched
```


