1번문제

```
#include <SDL2/SDL.h>
#include <math.h>

#define W 640
#define H 480
#define ROWS 7
#define COLS 10
#define BW 58
#define BH 20
#define GAP 4
#define PW 90
#define PH 12
#define PY (H - 40)
#define BR 7
#define SPEED 380.0f
#define OFFX ((W - (COLS * (BW + GAP) - GAP)) / 2)
#define OFFY 50

typedef struct { Uint8 r, g, b; } Rgb;

static const Rgb rainbow[ROWS] = {
    {255, 0, 0}, {255, 127, 0}, {255, 235, 0}, {0, 200, 60},
    {0, 140, 255}, {75, 0, 200}, {170, 0, 220}
};

static int bricks[ROWS][COLS];
static float px, bx, by, vx, vy;
static int lives, remain, launched, state;

static void reset_ball(void)
{
    launched = 0;
    vx = vy = 0;
    bx = px + PW / 2.0f;
    by = PY - BR;
}

static void reset_game(void)
{
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            bricks[r][c] = 1;
    remain = ROWS * COLS;
    lives = 3;
    state = 0;
    px = (W - PW) / 2.0f;
    reset_ball();
}

static void launch(void)
{
    launched = 1;
    vx = 140;
    vy = -sqrtf(SPEED * SPEED - vx * vx);
}

static int hit(float x, float y, float rx, float ry, float rw, float rh)
{
    return x + BR > rx && x - BR < rx + rw && y + BR > ry && y - BR < ry + rh;
}

static void update(float dt, const Uint8 *k)
{
    if (state) return;

    if (k[SDL_SCANCODE_LEFT] || k[SDL_SCANCODE_A]) px -= 520 * dt;
    if (k[SDL_SCANCODE_RIGHT] || k[SDL_SCANCODE_D]) px += 520 * dt;
    if (px < 0) px = 0;
    if (px > W - PW) px = W - PW;

    if (!launched) {
        bx = px + PW / 2.0f;
        by = PY - BR;
        return;
    }

    bx += vx * dt;
    by += vy * dt;

    if (bx < BR) { bx = BR; vx = fabsf(vx); }
    if (bx > W - BR) { bx = W - BR; vx = -fabsf(vx); }
    if (by < BR) { by = BR; vy = fabsf(vy); }

    if (vy > 0 && hit(bx, by, px, PY, PW, PH)) {
        float t = (bx - (px + PW / 2.0f)) / (PW / 2.0f);
        if (t > 1) t = 1;
        if (t < -1) t = -1;
        vx = t * SPEED * 0.8f;
        vy = -sqrtf(SPEED * SPEED - vx * vx);
        by = PY - BR;
    }

    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            if (!bricks[r][c]) continue;
            float rx = OFFX + c * (BW + GAP);
            float ry = OFFY + r * (BH + GAP);
            if (hit(bx, by, rx, ry, BW, BH)) {
                bricks[r][c] = 0;
                remain--;
                float ox = fminf(bx + BR - rx, rx + BW - (bx - BR));
                float oy = fminf(by + BR - ry, ry + BH - (by - BR));
                if (ox < oy) vx = -vx;
                else vy = -vy;
                goto done;
            }
        }
    }
done:

    if (by > H + BR) {
        if (--lives <= 0) state = 2;
        else reset_ball();
    }
    if (remain == 0) state = 1;
}

static void circle(SDL_Renderer *rd, int cx, int cy, int r)
{
    for (int dy = -r; dy <= r; dy++) {
        int dx = (int)sqrtf((float)(r * r - dy * dy));
        SDL_RenderDrawLine(rd, cx - dx, cy + dy, cx + dx, cy + dy);
    }
}

static void render(SDL_Renderer *rd)
{
    SDL_SetRenderDrawColor(rd, 15, 15, 30, 255);
    SDL_RenderClear(rd);

    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            if (!bricks[r][c]) continue;
            SDL_Rect b = { OFFX + c * (BW + GAP), OFFY + r * (BH + GAP), BW, BH };
            SDL_SetRenderDrawColor(rd, rainbow[r].r, rainbow[r].g, rainbow[r].b, 255);
            SDL_RenderFillRect(rd, &b);
        }
    }

    SDL_Rect p = { (int)px, PY, PW, PH };
    SDL_SetRenderDrawColor(rd, 235, 235, 235, 255);
    SDL_RenderFillRect(rd, &p);
    circle(rd, (int)bx, (int)by, BR);

    for (int i = 0; i < lives; i++) {
        SDL_Rect l = { 10 + i * 18, 12, 12, 12 };
        SDL_SetRenderDrawColor(rd, 255, 70, 90, 255);
        SDL_RenderFillRect(rd, &l);
    }

    if (state) {
        SDL_SetRenderDrawBlendMode(rd, SDL_BLENDMODE_BLEND);
        if (state == 1) {
            int h = H / ROWS + 1;
            for (int i = 0; i < ROWS; i++) {
                SDL_Rect s = { 0, i * h, W, h };
                SDL_SetRenderDrawColor(rd, rainbow[i].r, rainbow[i].g, rainbow[i].b, 200);
                SDL_RenderFillRect(rd, &s);
            }
        } else {
            SDL_Rect s = { 0, 0, W, H };
            SDL_SetRenderDrawColor(rd, 0, 0, 0, 190);
            SDL_RenderFillRect(rd, &s);
        }
        SDL_SetRenderDrawBlendMode(rd, SDL_BLENDMODE_NONE);
    }

    SDL_RenderPresent(rd);
}

int main(int argc, char *argv[])
{
    (void)argc;
    (void)argv;

    if (SDL_Init(SDL_INIT_VIDEO) != 0) return 1;

    SDL_Window *win = SDL_CreateWindow("Rainbow Breakout", SDL_WINDOWPOS_CENTERED,
                                       SDL_WINDOWPOS_CENTERED, W, H, 0);
    SDL_Renderer *rd = SDL_CreateRenderer(win, -1,
                                          SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC);
    if (!win || !rd) return 1;

    reset_game();

    int running = 1;
    Uint32 last = SDL_GetTicks();

    while (running) {
        SDL_Event e;
        while (SDL_PollEvent(&e)) {
            if (e.type == SDL_QUIT) running = 0;
            if (e.type == SDL_KEYDOWN) {
                if (e.key.keysym.sym == SDLK_ESCAPE) running = 0;
                if (e.key.keysym.sym == SDLK_SPACE) {
                    if (state) reset_game();
                    else if (!launched) launch();
                }
            }
        }

        Uint32 now = SDL_GetTicks();
        float dt = (now - last) / 1000.0f;
        last = now;
        if (dt > 0.05f) dt = 0.05f;

        update(dt, SDL_GetKeyboardState(NULL));
        render(rd);
    }

    SDL_DestroyRenderer(rd);
    SDL_DestroyWindow(win);
    SDL_Quit();
    return 0;
}
```
