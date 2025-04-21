# WIO Terminal俄罗斯方块开发教程
项目目标
开发基于WIO Terminal的经典俄罗斯方块游戏，实现以下核心功能：
- 五向按键控制（移动、旋转、硬降）
- TFT液晶屏图形渲染
- 7种经典方块类型
- 行消除计分系统
- 游戏状态管理（开始/结束/重置）

技术特性
| 类别        | 说明                      |
|-----------|-------------------------|
| 硬件平台     | Seeed Studio WIO Terminal |
| 显示系统     | 2.4" LCD (320x240)       |
| 控制方式     | 集成五向按键+功能键         |
| 开发框架     | Arduino Core + FreeRTOS  |
| 渲染引擎     | TFT_eSPI图形库            |

[系统架构图](https://via.placeholder.com/600x300?text=Hardware+Architecture)

材料清单
硬件组件
1. WIO Terminal开发板 ×1
2. USB Type-C数据线 ×1
3. 扩展底座（可选） ×1
   
![image](https://github.com/user-attachments/assets/571a7e59-97ab-416f-b53f-d4328d0c458e)

软件依赖:
Arduino IDE 2.3.2;
Seeed SAMD Boards 1.8.5;
“Seeed_Arduino_LCD” library;
Seeed_Arduino_FreeRTOS 1.0.2

开发环境搭建：
# 安装Arduino IDE后执行

双击已下载的应用程序启动

语言设置：
若界面显示非目标语言，可通过 文件 > 首选项 修改（详细指南）


Step 2. 添加 Wio Terminal 开发板支持
添加开发板管理器链接

打开 文件 > 首选项

在 附加开发板管理器网址 中粘贴以下 URL：

复制
https://files.seeedstudio.com/arduino/package_seeeduino_boards_index.json
安装 Wio Terminal 支持包

打开 工具 > 开发板 > 开发板管理器

搜索关键词 Wio Terminal 并安装

Step 4. 选择开发板与端口
选择开发板

工具 > 开发板 → 选择 Wio Terminal

选择端口

连接 Wio Terminal 到电脑

打开 工具 > 端口，选择出现的串口（如 COM3 或更高）

确认方法：
断开开发板后重新打开菜单，消失的端口即为目标端口，重新连接后选择它。
下载ZIP库文件在 ARDUINO IDE 加载Seeed Ardunio LCD库
https://codeload.github.com/Seeed-Studio/Seeed_Arduino_LCD/zip/refs/tags/V1.0
实现讲解：

# 输入检测函数（带防抖）

```cpp
void handleInput() {
  static uint32_t lastInput = 0;
  if(millis() - lastInput < 100) return;

  if(digitalRead(WIO_5S_LEFT))  tryMove(-1, 0);
  if(digitalRead(WIO_5S_RIGHT)) tryMove(1, 0);
  // 其他按键处理...
  
  lastInput = millis();
}
```
游戏核心逻辑：
graph TD
  A[新方块生成] --> B[玩家输入]
  B --> C{碰撞检测}
  C -->|允许移动| D[更新位置]
  C -->|触底| E[锁定方块]
  E --> F[行消除检查]
  F --> G[更新分数]
  G --> A

故障排查
| 现象                | 解决方案                     |
|--------------------|--------------------------|
| 屏幕无显示            | 检查TFT配置引脚定义           |
| 按键响应延迟           | 调整handleInput()防抖时间     |
| 方块渲染异常           | 验证颜色编码格式              |
| 内存不足导致崩溃         | 优化grid数组存储方式          |

## 参考资源
1. [WIO Terminal官方文档](https://wiki.seeedstudio.com/WIO-Terminal-Getting-Started/)
2. [俄罗斯方块实现规范](https://tetris.wiki/SRS)


完整项目代码：

```cpp
#include <TFT_eSPI.h>
#include <Seeed_Arduino_FreeRTOS.h>

TFT_eSPI tft;

// Game area definitions
#define GRID_WIDTH 10
#define GRID_HEIGHT 20
#define BLOCK_SIZE 15

// Color definitions
#define COLOR_I    0x07FF  // Cyan
#define COLOR_O    0xFFE0  // Yellow
#define COLOR_T    0xF81F  // Magenta
#define COLOR_S    0x07E0  // Green
#define COLOR_Z    0xF800  // Red
#define COLOR_J    0x001F  // Blue
#define COLOR_L    0xFC00  // Orange

// Block structures
struct Point { int x, y; };
struct Tetromino {
    Point blocks[4];
    int color;
    int x;
    int y;
};

// Game state
Tetromino current;
int grid[GRID_HEIGHT][GRID_WIDTH] = {0};
int score = 0;
bool gameOver = false;
uint32_t lastFall = 0;

// Tetromino shapes
const Tetromino SHAPES[7] = {
    {{{0,0}, {1,0}, {2,0}, {3,0}}, COLOR_I, 0, 0},  // I
    {{{0,0}, {1,0}, {0,1}, {1,1}}, COLOR_O, 0, 0},  // O
    {{{0,0}, {1,0}, {2,0}, {1,1}}, COLOR_T, 0, 0},  // T
    {{{1,0}, {2,0}, {0,1}, {1,1}}, COLOR_S, 0, 0},  // S
    {{{0,0}, {1,0}, {1,1}, {2,1}}, COLOR_Z, 0, 0},  // Z
    {{{0,0}, {0,1}, {0,2}, {1,2}}, COLOR_J, 0, 0},  // J
    {{{1,0}, {1,1}, {1,2}, {0,2}}, COLOR_L, 0, 0}   // L
};

void setup() {
    Serial.begin(115200);
    tft.begin();
    tft.setRotation(3);
    tft.fillScreen(TFT_BLACK);

    // Initialize buttons
    const uint8_t pins[] = {WIO_5S_UP, WIO_5S_DOWN, WIO_5S_LEFT, 
                           WIO_5S_RIGHT, WIO_5S_PRESS, 
                           WIO_KEY_A, WIO_KEY_B, WIO_KEY_C};
    for(auto pin : pins) pinMode(pin, INPUT_PULLUP);

    randomSeed(analogRead(0));
    newPiece();
}

void loop() {
    if(gameOver) {
        showGameOver();
        return;
    }

    handleInput();
    autoFall();
    drawGame();
    delay(50);
}

// Input handling with debounce
void handleInput() {
    static uint32_t lastInput = 0;
    if(millis() - lastInput < 100) return;

    if(digitalRead(WIO_5S_LEFT) == LOW)  tryMove(-1, 0);
    if(digitalRead(WIO_5S_RIGHT) == LOW) tryMove(1, 0);
    if(digitalRead(WIO_5S_DOWN) == LOW)  tryMove(0, 1);
    if(digitalRead(WIO_5S_UP) == LOW)    rotatePiece();
    if(digitalRead(WIO_5S_PRESS) == LOW) hardDrop();

    lastInput = millis();
}

// Automatic falling
void autoFall() {
    if(millis() - lastFall > 1000) {
        moveDown();
        lastFall = millis();
    }
}

// Create new tetromino
void newPiece() {
    current = SHAPES[random(7)];
    current.x = GRID_WIDTH/2 - 2;
    current.y = 0;

    if(!isValidPosition(0, 0)) {
        gameOver = true;
    }
}

// Collision detection
bool isValidPosition(int dx, int dy) {
    for(int i=0; i<4; i++) {
        int newX = current.x + current.blocks[i].x + dx;
        int newY = current.y + current.blocks[i].y + dy;

        if(newX < 0 || newX >= GRID_WIDTH) return false;
        if(newY >= GRID_HEIGHT) return false;
        if(newY >=0 && grid[newY][newX]) return false;
    }
    return true;
}

// Movement handling
bool tryMove(int dx, int dy) {
    if(!isValidPosition(dx, dy)) return false;
    
    current.x += dx;
    current.y += dy;
    return true;
}

void moveDown() {
    if(!tryMove(0, 1)) {
        lockPiece();
        checkLines();
        newPiece();
    }
}

// Lock piece to grid
void lockPiece() {
    for(int i=0; i<4; i++) {
        int x = current.x + current.blocks[i].x;
        int y = current.y + current.blocks[i].y;
        if(y >=0) grid[y][x] = current.color;
    }
}

// Line clearing
void checkLines() {
    for(int y=GRID_HEIGHT-1; y>=0; y--) {
        bool full = true;
        for(int x=0; x<GRID_WIDTH; x++) {
            if(grid[y][x] == 0) {
                full = false;
                break;
            }
        }

        if(full) {
            score += 100;
            for(int ny=y; ny>0; ny--) {
                memcpy(grid[ny], grid[ny-1], sizeof(grid[0]));
            }
            memset(grid[0], 0, sizeof(grid[0]));
            y++; // Check the new row that moved down
        }
    }
}

// Rotation handling
void rotatePiece() {
    Tetromino temp = current;
    
    for(int i=0; i<4; i++) {
        int newX = current.blocks[i].y;
        int newY = -current.blocks[i].x;
        temp.blocks[i].x = newX;
        temp.blocks[i].y = newY;
    }

    if(isValidRotation(temp)) {
        current = temp;
    }
}

bool isValidRotation(Tetromino &t) {
    for(int i=0; i<4; i++) {
        int newX = current.x + t.blocks[i].x;
        int newY = current.y + t.blocks[i].y;
        
        if(newX <0 || newX >=GRID_WIDTH) return false;
        if(newY >=GRID_HEIGHT) return false;
        if(grid[newY][newX]) return false;
    }
    return true;
}

// Hard drop (instant fall)
void hardDrop() {
    while(tryMove(0, 1)) {}
    moveDown();
}

// Render game
void drawGame() {
    tft.fillScreen(TFT_BLACK);
    
    // Draw game border
    tft.drawRect(0, 0, GRID_WIDTH*BLOCK_SIZE, 
                GRID_HEIGHT*BLOCK_SIZE, TFT_WHITE);

    // Draw locked pieces
    for(int y=0; y<GRID_HEIGHT; y++) {
        for(int x=0; x<GRID_WIDTH; x++) {
            if(grid[y][x]) {
                drawBlock(x, y, grid[y][x]);
            }
        }
    }

    // Draw current piece
    for(int i=0; i<4; i++) {
        int x = current.x + current.blocks[i].x;
        int y = current.y + current.blocks[i].y;
        if(y >=0) drawBlock(x, y, current.color);
    }

    // Draw score
    tft.setCursor(GRID_WIDTH*BLOCK_SIZE +10, 10);
    tft.setTextColor(TFT_WHITE);
    tft.print("Score:");
    tft.println(score);
}

// Draw individual block
void drawBlock(int x, int y, uint16_t color) {
    tft.fillRect(x*BLOCK_SIZE+1, y*BLOCK_SIZE+1,
                BLOCK_SIZE-2, BLOCK_SIZE-2, color);
}

// Game over screen
void showGameOver() {
    tft.fillScreen(TFT_RED);
    tft.setTextColor(TFT_WHITE);
    tft.setTextSize(2);
    tft.setCursor(30, 50);
    tft.print("GAME OVER");
    tft.setCursor(20, 80);
    tft.print("Score: ");
    tft.print(score);
    
    while(true) {
        if(digitalRead(WIO_KEY_C) == LOW) {
            resetGame();
            break;
        }
    }
}

// Reset game state
void resetGame() {
    memset(grid, 0, sizeof(grid));
    score = 0;
    gameOver = false;
    newPiece();
}
```

完整效果:

![image](https://github.com/user-attachments/assets/8cf8e3a9-7ffd-4627-92f4-44fb1c4b4514)

问题与解决:
如果屏幕不能被点亮: 尝试删除 “TFT_eSPI” library安装 “Seeed_Arduino_LCD” libraryhttps://codeload.github.com/Seeed-Studio/Seeed_Arduino_LCD/zip/refs/tags/V1.0

Wio Set up 指南：
https://wiki.seeedstudio.com/Wio-Terminal-Getting-Started/


