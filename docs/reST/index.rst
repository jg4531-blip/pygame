import math
import sys
import pygame

# --- GAME SETTINGS ---
SCREEN_WIDTH = 800
SCREEN_HEIGHT = 600
FPS = 60

# Player variables
player_x = 3.5
player_y = 3.5
player_angle = 0.0  # In radians
FOV = math.pi / 3  # 60-degree field of view
HALF_FOV = FOV / 2
NUM_RAYS = 120  # Number of rays to cast (higher = better quality, lower = faster)
MAX_DEPTH = 16  # Maximum visibility distance
PLAYER_SPEED = 0.05
ROTATION_SPEED = 0.04

# Map layout: 1s represent walls, 0s represent empty space
MAP = [
    [1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 0, 1, 0, 1],
    [1, 0, 1, 0, 0, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 1, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1],
]
MAP_SIZE = len(MAP)

# Initialize Pygame
pygame.init()
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Python Pseudo-3D Engine")
clock = pygame.time.Clock()

# Calculate dynamic projection scalars
DELTA_ANGLE = FOV / NUM_RAYS
SCALE = SCREEN_WIDTH // NUM_RAYS
DIST = NUM_RAYS / (2 * math.tan(HALF_FOV))
PROJ_COEFF = DIST * 40  # Constant to scale wall height adjustments

# Main Game Loop
while True:
    # 1. Event Handling
    for event in pygame.event.get():
        if event.type == pygame.QUIT or (
            event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE
        ):
            pygame.quit()
            sys.exit()

    # 2. Player Controls & Collision Detection
    keys = pygame.key.get_pressed()
    sin_a = math.sin(player_angle)
    cos_a = math.cos(player_angle)

    # Rotation
    if keys[pygame.K_LEFT]:
        player_angle -= ROTATION_SPEED
    if keys[pygame.K_RIGHT]:
        player_angle += ROTATION_SPEED

    # Forward / Backward Movement (with basic grid collision check)
    new_x, new_y = player_x, player_y
    if keys[pygame.K_w]:
        new_x += PLAYER_SPEED * cos_a
        new_y += PLAYER_SPEED * sin_a
    if keys[pygame.K_s]:
        new_x -= PLAYER_SPEED * cos_a
        new_y -= PLAYER_SPEED * sin_a

    if MAP[int(new_y)][int(new_x)] == 0:
        player_x, player_y = new_x, new_y

    # 3. Rendering (Drawing Sky and Floor)
    screen.fill((30, 30, 30))  # Dark gray floor
    pygame.draw.rect(
        screen, (50, 70, 90), (0, 0, SCREEN_WIDTH, SCREEN_HEIGHT // 2)
    )  # Blue-gray sky

    # 4. Raycasting Engine Logic
    start_angle = player_angle - HALF_FOV

    for ray in range(NUM_RAYS):
        cur_angle = start_angle + ray * DELTA_ANGLE
        sin_r = math.sin(cur_angle)
        cos_r = math.cos(cur_angle)

        # Cast a single ray forward until it hits a wall cell
        for depth in range(1, int(MAX_DEPTH * 10)):
            d = depth / 10
            target_x = player_x + d * cos_r
            target_y = player_y + d * sin_r

            # Map check boundaries
            if (
                0 <= target_x < MAP_SIZE
                and 0 <= target_y < MAP_SIZE
                and MAP[int(target_y)][int(target_x)] == 1
            ):
                # Fix fish-eye lens distortion
                d *= math.cos(player_angle - cur_angle)

                # Calculate the exact perceived 3D wall slice height
                wall_height = min(int(PROJ_COEFF / (d + 0.0001)), SCREEN_HEIGHT)

                # Shading: Walls farther away get deeper shades of grey/black
                color_val = int(255 / (1 + d * d * 0.2))
                wall_color = (color_val, color_val // 2, color_val // 3)

                # Draw the column slice representing this ray
                pygame.draw.rect(
                    screen,
                    wall_color,
                    (
                        ray * SCALE,
                        (SCREEN_HEIGHT // 2) - (wall_height // 2),
                        SCALE,
                        wall_height,
                    ),
                )
                break

    # 5. Core Screen Refresh
    pygame.display.flip()
    clock.tick(FPS)
