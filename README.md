#include <iostream>      

#include <vector>        

#include <string>        

#include <cmath>         

#include <utility>       

#include <thread>       

#include <chrono>        

#ifdef _WIN32

    #include <conio.h>  

#else

    #include <termios.h> 

    #include <unistd.h>  

    #include <fcntl.h>  

#endif

using namespace std;

const int WIDTH = 40;

const int HEIGHT = 20;

// ============ Platform-Agnostic Input ============

bool kbhit() {

#ifdef _WIN32

    return _kbhit();

#else

    termios oldt, newt;

    int ch;

    int oldf;

    tcgetattr(STDIN_FILENO, &oldt);

    newt = oldt;

    newt.c_lflag &= ~(ICANON | ECHO);

    tcsetattr(STDIN_FILENO, TCSANOW, &newt);

    oldf = fcntl(STDIN_FILENO, F_GETFL, 0);

    fcntl(STDIN_FILENO, F_SETFL, oldf | O_NONBLOCK);

    ch = getchar();

    tcsetattr(STDIN_FILENO, TCSANOW, &oldt);

    fcntl(STDIN_FILENO, F_SETFL, oldf);

    if (ch != EOF) {

        ungetc(ch, stdin);

        return true;

    }

    return false;

#endif

}

char getch_nonblock() {

#ifdef _WIN32

    return _getch();

#else

    return getchar();

#endif

}

// ============ Utility ============

pair<int, int> mapToScreen(float x, float y) {

    int sx = (int)((x + 1) * 0.5f * (WIDTH - 1));

    int sy = (int)((1 - (y + 1) * 0.5f) * (HEIGHT - 1));

    return {sx, sy};

}

void clearScreen() {

    cout << "\033[2J\033[1;1H";

}

// ============ ECS System ============

struct Transform {

    float angle = 0.0f;

};

class Entity {

public:

    Transform transform;

    virtual void update(char input) {}

    virtual void render(vector<string>& screen) {}

    virtual ~Entity() {}

};

class Rotator : public Entity {

public:

    Entity* target;

    float speed;

    Rotator(Entity* t, float s) : target(t), speed(s) {}

    void update(char input) override {

        target->transform.angle += speed;

    }

};

class Triangle : public Entity {

public:

    void update(char input) override {

        if (input == 'a' || input == 'A') transform.angle -= 0.1f;

        if (input == 'd' || input == 'D') transform.angle += 0.1f;

    }

    void render(vector<string>& screen) override {

        float a = transform.angle;

        float x1 = cos(a), y1 = sin(a);

        float x2 = cos(a + 2.094f), y2 = sin(a + 2.094f);

        float x3 = cos(a + 4.188f), y3 = sin(a + 4.188f);

        auto [sx1, sy1] = mapToScreen(x1, y1);

        auto [sx2, sy2] = mapToScreen(x2, y2);

        auto [sx3, sy3] = mapToScreen(x3, y3);

        if (sy1 >= 0 && sy1 < HEIGHT && sx1 >= 0 && sx1 < WIDTH) screen[sy1][sx1] = '*';

        if (sy2 >= 0 && sy2 < HEIGHT && sx2 >= 0 && sx2 < WIDTH) screen[sy2][sx2] = '*';

        if (sy3 >= 0 && sy3 < HEIGHT && sx3 >= 0 && sx3 < WIDTH) screen[sy3][sx3] = '*';

    }

};

// ============ Scene Management ============

class Scene {

public:

    virtual void update(char input) = 0;

    virtual void render() = 0;

    virtual ~Scene() {}

};

class MainScene : public Scene {

private:

    vector<Entity*> entities;

    int frameCount = 0;

public:

    MainScene() {

        Triangle* t = new Triangle();

        entities.push_back(t);

        entities.push_back(new Rotator(t, 0.02f));

    }

    void update(char input) override {

        frameCount++;

        for (auto e : entities) e->update(input);

    }

    void render() override {

        vector<string> screen(HEIGHT, string(WIDTH, ' '));

        for (auto e : entities) e->render(screen);

        clearScreen();

        cout << "Scene: Main Triangle World | Frame: " << frameCount << "\n";

        cout << "Controls: A/D to rotate, Q to quit\n";

        for (const auto& line : screen) cout << line << '\n';

    }

    ~MainScene() {

        for (auto e : entities) delete e;

    }

};

// ============ Game Loop ============

class Game {

private:

    Scene* currentScene = nullptr;

    bool running = true;

public:

    void setScene(Scene* scene) {

        if (currentScene) delete currentScene;

        currentScene = scene;

    }

    void run() {

        while (running) {

            char input = '\0';

            if (kbhit()) {

                input = getch_nonblock();

                if (input == 'q' || input == 'Q') running = false;

            }

            currentScene->update(input);

            currentScene->render();

            this_thread::sleep_for(chrono::milliseconds(100));

        }

    }

    ~Game() {

        if (currentScene) delete currentScene;

    }

};

// ============ Main ============

int main() {

    Game game;

    game.setScene(new MainScene());

    game.run();

    cout << "\nGame Over.\n";

    return 0;

}

//my game engine//
