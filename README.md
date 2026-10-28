import random
import pygame
import mysql.connector

# ---------- DATABASE ----------

con = mysql.connector.connect(
    host="localhost",
    user="root",
    password="",
    database="snake_db"
)

cur = con.cursor(buffered=True)
cur.callproc("initGame")
con.commit()


def get_snake():
    cur.execute("SELECT x, y FROM queue ORDER BY id")
    return cur.fetchall()


def get_food():
    cur.execute("SELECT x, y FROM bucket ORDER BY id LIMIT 1")
    return cur.fetchone()


def set_direction(d):
    cur.execute(
        "UPDATE direction SET direction = %s",
        (d,)
    )
    con.commit()


def place_food(snake):
    empty = [
        (x, y)
        for x in range(10)
        for y in range(10)
        if (x, y) not in snake
    ]

    if empty:
        x, y = random.choice(empty)
        cur.execute(
            "UPDATE bucket SET x = %s, y = %s WHERE id = 1",
            (x, y)
        )
        con.commit()


# ---------- PYGAME ----------

pygame.init()

W, H = 900, 610
CELL = 42
BOARD = 420

screen = pygame.display.set_mode((W, H))
pygame.display.set_caption("Snake - MySQL Edition")
clock = pygame.time.Clock()

# Windows 95 colours
TEAL = (0, 128, 128)
GREY = (192, 192, 192)
DARK = (64, 64, 64)
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
BLUE = (0, 0, 128)
GREEN = (0, 180, 0)
DARK_GREEN = (0, 100, 0)
RED = (220, 0, 0)
YELLOW = (255, 255, 0)

font = pygame.font.SysFont("Courier New", 18, bold=True)
small = pygame.font.SysFont("Courier New", 14)
title_font = pygame.font.SysFont("Arial", 20, bold=True)

# Game settings
score = 0
speed = 180
paused = False
running = True
game_over = False
direction = "UP"
last_move = pygame.time.get_ticks()

snake = get_snake()
food = get_food()


# ---------- DRAWING HELPERS ----------

def text(msg, x, y, colour=BLACK, f=font):
    image = f.render(str(msg), True, colour)
    screen.blit(image, (x, y))


def panel(rect):
    pygame.draw.rect(screen, GREY, rect)

    x, y, w, h = rect

    pygame.draw.line(screen, WHITE, (x, y), (x + w - 1, y), 3)
    pygame.draw.line(screen, WHITE, (x, y), (x, y + h - 1), 3)

    pygame.draw.line(
        screen, DARK, (x, y + h - 1), (x + w - 1, y + h - 1), 3
    )
    pygame.draw.line(
        screen, DARK, (x + w - 1, y), (x + w - 1, y + h - 1), 3
    )


def button(label, rect):
    panel(rect)
    image = small.render(label, True, BLACK)
    screen.blit(
        image,
        image.get_rect(center=(
            rect[0] + rect[2] // 2,
            rect[1] + rect[3] // 2
        ))
    )


def draw():
    screen.fill(TEAL)

    # Main application window
    panel((20, 15, 860, 580))

    # Title bar
    pygame.draw.rect(screen, BLUE, (25, 20, 850, 35))
    text("Snake - MySQL Edition", 35, 25, WHITE, title_font)

    # Window controls
    panel((814, 23, 23, 23))
    text("_", 819, 19, BLACK, small)

    panel((840, 23, 23, 23))
    text("X", 845, 24, BLACK, small)

    # Menu bar
    pygame.draw.rect(screen, GREY, (25, 55, 850, 30))
    text("Game", 38, 60, BLACK, small)
    text("Options", 100, 60, BLACK, small)
    text("Help", 190, 60, BLACK, small)

    # Game board
    board_x, board_y = 35, 105

    pygame.draw.rect(
        screen, BLACK,
        (board_x, board_y, BOARD, BOARD)
    )

    # Grid
    for i in range(11):
        pygame.draw.line(
            screen, (35, 35, 35),
            (board_x + i * CELL, board_y),
            (board_x + i * CELL, board_y + BOARD)
        )
        pygame.draw.line(
            screen, (35, 35, 35),
            (board_x, board_y + i * CELL),
            (board_x + BOARD, board_y + i * CELL)
        )

    # Food
    if food is not None:
        fx, fy = food
        pygame.draw.rect(
            screen, RED,
            (board_x + fx * CELL + 7,
             board_y + fy * CELL + 7,
             CELL - 14, CELL - 14)
        )

    # Snake
    for i, (sx, sy) in enumerate(snake):
        colour = GREEN if i == 0 else DARK_GREEN

        pygame.draw.rect(
            screen, colour,
            (board_x + sx * CELL + 3,
             board_y + sy * CELL + 3,
             CELL - 6, CELL - 6)
        )

        pygame.draw.rect(
            screen, (0, 220, 0),
            (board_x + sx * CELL + 5,
             board_y + sy * CELL + 5,
             CELL - 10, CELL - 10),
            1
        )

    # Right-side information
    panel((480, 105, 365, 100))
    text("SCORE", 500, 120)
    text(score, 500, 150, BLUE, title_font)

    text("LENGTH", 650, 120)
    text(len(snake), 650, 150, BLUE, title_font)

    panel((480, 225, 365, 115))
    text("CONTROLS", 495, 240)
    text("Arrow keys - Move", 495, 270, BLACK, small)
    text("Space      - Pause", 495, 292, BLACK, small)
    text("Esc        - Exit", 495, 314, BLACK, small)

    # Buttons
    button("New Game", (490, 365, 105, 35))
    button("Pause", (605, 365, 95, 35))
    button("Speed", (710, 365, 95, 35))

    # Status bar
    panel((35, 545, 810, 30))

    if game_over:
        status = "GAME OVER - Click New Game"
    elif paused:
        status = "PAUSED"
    else:
        status = "Playing"

    text(status, 45, 550, BLACK, small)
    text("MySQL Connected", 650, 550, BLACK, small)

    pygame.display.flip()


# ---------- GAME LOOP ----------

while running:
    now = pygame.time.get_ticks()

    for event in pygame.event.get():

        if event.type == pygame.QUIT:
            running = False

        elif event.type == pygame.KEYDOWN:

            if event.key == pygame.K_ESCAPE:
                running = False

            elif event.key == pygame.K_SPACE and not game_over:
                paused = not paused

            elif not game_over:
                # Prevent reversing directly into the snake
                if event.key == pygame.K_UP and direction != "DOWN":
                    direction = "UP"

                elif event.key == pygame.K_DOWN and direction != "UP":
                    direction = "DOWN"

                elif event.key == pygame.K_LEFT and direction != "RIGHT":
                    direction = "LEFT"

                elif event.key == pygame.K_RIGHT and direction != "LEFT":
                    direction = "RIGHT"

        elif event.type == pygame.MOUSEBUTTONDOWN:
            mx, my = event.pos

            # New Game
            if 490 <= mx <= 595 and 365 <= my <= 400:
                cur.callproc("initGame")
                con.commit()

                score = 0
                direction = "UP"
                paused = False
                game_over = False

                snake = get_snake()
                place_food(snake)
                food = get_food()
                last_move = now

            # Pause
            elif 605 <= mx <= 700 and 365 <= my <= 400:
                if not game_over:
                    paused = not paused

            # Speed
            elif 710 <= mx <= 805 and 365 <= my <= 400:
                speed = 100 if speed == 180 else 180

    # Movement
    if not paused and not game_over and now - last_move >= speed:

        old_snake = get_snake()
        old_tail = old_snake[-1] if old_snake else None

        set_direction(direction)

        cur.callproc("moveQueue")
        con.commit()

        snake = get_snake()
        food = get_food()

        # Check whether the head ate the food
        if snake and food is not None and snake[0] == food:

            score += 1

            # Add a segment at the old tail position
            if old_tail is not None:
                cur.execute(
                    "INSERT INTO queue (x, y) VALUES (%s, %s)",
                    old_tail
                )
                con.commit()

            snake = get_snake()
            place_food(snake)
            food = get_food()

        # Detect collision with itself
        if snake and snake[0] in snake[1:]:
            game_over = True

        last_move = now

    draw()
    clock.tick(60)


# ---------- CLEANUP ----------

cur.close()
con.close()
pygame.quit()
