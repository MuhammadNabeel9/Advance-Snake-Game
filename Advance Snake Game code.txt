import pygame
import random
import sys
import math
import json
import os
import cv2
import numpy as np
from datetime import datetime

pygame.init()
user_type = None
username = None

# Global variable to track if video should be playing
video_playing = True

# Get screen dimensions
info = pygame.display.Info()
WIDTH, HEIGHT = info.current_w, info.current_h
SCALE_FACTOR = min(WIDTH / 600, HEIGHT / 600)  # Scale factor based on original 600x600

# Scale constants
BLOCK_SIZE = int(20 * SCALE_FACTOR)
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (200, 0, 0)
PINK = (255, 105, 180)
RAT_COLOR = (30, 30, 30)
BIG_RAT_COLOR = (30, 30, 30)  # Same color as regular rats
BUTTON_COLOR = (50, 120, 200)
BUTTON_HOVER = (70, 150, 230)
OBSTACLE_COLOR = (200, 200, 0)  # Distinct yellow color for obstacles
INTRO_FPS = 55  # Adjust this to your video's actual FPS if needed
PROFILE_COLOR = (70, 130, 180)
PROFILE_HOVER = (90, 150, 200)

CELL_SIZE = 20                # width/height of each grid cell in pixels
GRID_COLS = WIDTH // CELL_SIZE
GRID_ROWS = HEIGHT // CELL_SIZE
CENTER_COL = GRID_COLS // 2   # vertical border column index

applied_skin_index = 0
applied_head_index = 0
applied_body_index = 0
applied_tail_index = 0

# Add these constants near other button definitions
MULTIPLAYER_BUTTON_COLOR = (100, 100, 200)
MULTIPLAYER_BUTTON_HOVER = (120, 120, 220)

# Setup display
screen = pygame.display.set_mode((WIDTH, HEIGHT), pygame.FULLSCREEN)
pygame.display.set_caption("Snake Game")

# Scale fonts based on screen size
font = pygame.font.SysFont("Arial", int(25 * SCALE_FACTOR), bold=True)
big_font = pygame.font.SysFont("Arial", int(50 * SCALE_FACTOR))
small_font = pygame.font.SysFont("Arial", int(18 * SCALE_FACTOR), bold=True)
tiny_font = pygame.font.SysFont("Arial", int(14 * SCALE_FACTOR))

# Load assets (if you decide to use them)
video_path = "menu_bg_video.mp4"  # Make sure this video file is in the same folder
cap = cv2.VideoCapture(video_path)

# Scale icons
play_icon = pygame.image.load("play_icon.jpg").convert_alpha()
play_icon = pygame.transform.scale(play_icon, (int(90 * SCALE_FACTOR), int(90 * SCALE_FACTOR)))

settings_icon = pygame.image.load("settings_icon.png").convert_alpha()
settings_icon = pygame.transform.scale(settings_icon, (int(65 * SCALE_FACTOR), int(65 * SCALE_FACTOR)))

# After pygame.init() and other image loads, add:
try:
    high_score_img = pygame.image.load("high_score_icon.png").convert_alpha()
    high_score_img = pygame.transform.scale(high_score_img, (int(60 * SCALE_FACTOR), int(60 * SCALE_FACTOR)))
except:
    print("High score image not found, using text instead")
    high_score_img = None

intro_path = "intro_bg_video.mp4"  # Use your actual video filename here
cap_intro = cv2.VideoCapture(intro_path)

if not cap_intro.isOpened():
    print("Intro video failed to load.")
    cap_intro = None

exit_icon = pygame.image.load("exit_icon.png").convert_alpha()
exit_icon = pygame.transform.scale(exit_icon, (int(65 * SCALE_FACTOR), int(65 * SCALE_FACTOR)))

high_score_icon = pygame.Surface((int(65 * SCALE_FACTOR), int(65 * SCALE_FACTOR)), pygame.SRCALPHA)
pygame.draw.circle(high_score_icon, BUTTON_COLOR, (int(32 * SCALE_FACTOR), int(32 * SCALE_FACTOR)), int(30 * SCALE_FACTOR))
score_text = pygame.font.SysFont("Arial", int(20 * SCALE_FACTOR), bold=True).render("HS", True, WHITE)
high_score_icon.blit(score_text, (int(32 * SCALE_FACTOR) - score_text.get_width() // 2, 
                     int(32 * SCALE_FACTOR) - score_text.get_height() // 2))

reset_icon = pygame.Surface((int(65 * SCALE_FACTOR), int(65 * SCALE_FACTOR)), pygame.SRCALPHA)
pygame.draw.circle(reset_icon, BUTTON_COLOR, (int(32 * SCALE_FACTOR), int(32 * SCALE_FACTOR)), int(30 * SCALE_FACTOR))
reset_text = pygame.font.SysFont("Arial", int(20 * SCALE_FACTOR), bold=True).render("R", True, WHITE)
reset_icon.blit(reset_text, (int(32 * SCALE_FACTOR) - reset_text.get_width() // 2, 
                int(32 * SCALE_FACTOR) - reset_text.get_height() // 2))

pygame.mixer.pre_init(44100, -16, 2, 512)
pygame.init() # Pygame already initialized above. This is redundant.

# Game settings
clock = pygame.time.Clock()
speed_levels = [300, 275, 250, 225, 200, 175, 150, 125, 100, 50]  # Delays in milliseconds
current_speed_index = 4  # Default to level 5 (200ms)
difficulty_levels = ["Easy", "Medium", "Hard"]
current_difficulty_index = 0

# File to store Facebook user data
facebook_users_file = "facebook_users.json"

# File to store profile data
profile_data_file = "user_profiles.json"

# Load existing users or create empty dict
if os.path.exists(facebook_users_file):
    with open(facebook_users_file, "r") as f:
        try:
            facebook_users = json.load(f)
        except json.JSONDecodeError:
            facebook_users = {}
else:
    facebook_users = {}

def save_facebook_users():
    with open(facebook_users_file, "w") as f:
        json.dump(facebook_users, f)

# Load profile data or create empty dict
if os.path.exists(profile_data_file):
    with open(profile_data_file, "r") as f:
        try:
            profile_data = json.load(f)
        except json.JSONDecodeError:
            profile_data = {}
else:
    profile_data = {}

def save_profile_data():
    with open(profile_data_file, "w") as f:
        json.dump(profile_data, f)

# Default avatars (10 simple colored circles with faces)
DEFAULT_AVATARS = [
    {"color": (255, 0, 0), "eyes": "happy"},    # Red
    {"color": (0, 255, 0), "eyes": "normal"},   # Green
    {"color": (0, 0, 255), "eyes": "surprised"},# Blue
    {"color": (255, 255, 0), "eyes": "angry"},  # Yellow
    {"color": (255, 0, 255), "eyes": "happy"},  # Magenta
    {"color": (0, 255, 255), "eyes": "normal"}, # Cyan
    {"color": (255, 128, 0), "eyes": "happy"},  # Orange
    {"color": (128, 0, 128), "eyes": "normal"}, # Purple
    {"color": (0, 128, 0), "eyes": "happy"},    # Dark Green
    {"color": (128, 128, 128), "eyes": "angry"} # Gray
]

# Snake Skin Designs - Only colors
SNAKE_SKINS = [
    # Original single-color skins (15)
    {"name": "Gold", "color": (255, 215, 0)},
    {"name": "Silver", "color": (200, 200, 200)},
    {"name": "Emerald", "color": (80, 200, 120)},
    {"name": "Ruby", "color": (220, 60, 60)},
    {"name": "Sapphire", "color": (60, 100, 220)},
    {"name": "Amethyst", "color": (180, 80, 220)},
    {"name": "Pearl", "color": (240, 240, 240)},
    {"name": "Obsidian", "color": (40, 40, 40)},
    {"name": "Lava", "color": (255, 80, 0)},
    {"name": "Ocean", "color": (0, 120, 200)},
    {"name": "Sunset", "color": (255, 100, 0)},
    {"name": "Midnight", "color": (0, 50, 100)},
    {"name": "Lemon", "color": (255, 255, 0)},
    {"name": "Lavender", "color": (200, 150, 255)},
    {"name": "Charcoal", "color": (60, 60, 60)},
    
    # New multi-colored skins (15)
    {"name": "Rainbow", "colors": [
        (255, 0, 0),    # Red
        (255, 165, 0),  # Orange
        (255, 255, 0),  # Yellow
        (0, 255, 0),    # Green
        (0, 0, 255),    # Blue
        (75, 0, 130),   # Indigo
        (148, 0, 211)   # Violet
    ]},
    {"name": "Tropical", "colors": [
        (255, 105, 180),  # Hot pink
        (255, 215, 0),    # Gold
        (0, 191, 255),    # Deep sky blue
        (50, 205, 50)     # Lime green
    ]},
    {"name": "Galaxy", "colors": [
        (25, 25, 112),    # Midnight blue
        (75, 0, 130),     # Indigo
        (138, 43, 226),   # Blue violet
        (147, 112, 219),  # Medium purple
        (230, 230, 250)   # Lavender
    ]},
    {"name": "Fire", "colors": [
        (255, 0, 0),      # Red
        (255, 69, 0),     # Red-orange
        (255, 140, 0),    # Dark orange
        (255, 165, 0),    # Orange
        (255, 215, 0)     # Gold
    ]},
    {"name": "Ice", "colors": [
        (173, 216, 230),  # Light blue
        (135, 206, 235),  # Sky blue
        (176, 224, 230),  # Powder blue
        (224, 255, 255),  # Light cyan
        (240, 248, 255)   # Alice blue
    ]},
    {"name": "Desert", "colors": [
        (210, 180, 140),  # Tan
        (205, 133, 63),   # Peru
        (139, 69, 19),    # Saddle brown
        (222, 184, 135),  # Burlywood
        (245, 222, 179)   # Wheat
    ]},
    {"name": "Jungle", "colors": [
        (34, 139, 34),    # Forest green
        (0, 100, 0),      # Dark green
        (107, 142, 35),   # Olive drab
        (85, 107, 47),    # Dark olive green
        (154, 205, 50)    # Yellow green
    ]},
    {"name": "Neon", "colors": [
        (255, 0, 255),    # Fuchsia
        (0, 255, 255),    # Cyan
        (255, 255, 0),    # Yellow
        (50, 255, 50),    # Bright green
        (255, 0, 128)     # Deep pink
    ]},
    {"name": "Sunrise", "colors": [
        (255, 20, 147),   # Deep pink
        (255, 69, 0),     # Red-orange
        (255, 140, 0),    # Dark orange
        (255, 215, 0),    # Gold
        (255, 255, 224)   # Light yellow
    ]},
    {"name": "Ocean Depth", "colors": [
        (0, 0, 128),      # Navy
        (0, 0, 205),      # Medium blue
        (65, 105, 225),   # Royal blue
        (0, 191, 255),    # Deep sky blue
        (64, 224, 208)    # Turquoise
    ]},
    {"name": "Candy", "colors": [
        (255, 105, 180),  # Hot pink
        (255, 182, 193),  # Light pink
        (173, 216, 230),  # Light blue
        (152, 251, 152),  # Pale green
        (255, 228, 196)   # Bisque
    ]},
    {"name": "Autumn", "colors": [
        (139, 69, 19),    # Saddle brown
        (160, 82, 45),    # Sienna
        (205, 133, 63),   # Peru
        (210, 105, 30),   # Chocolate
        (218, 165, 32)    # Goldenrod
    ]},
    {"name": "Cyberpunk", "colors": [
        (0, 255, 255),    # Cyan
        (255, 0, 255),    # Magenta
        (0, 0, 128),      # Navy
        (75, 0, 130),     # Indigo
        (138, 43, 226)    # Blue violet
    ]},
    {"name": "Pastel", "colors": [
        (255, 182, 193),  # Light pink
        (221, 160, 221),  # Plum
        (152, 251, 152),  # Pale green
        (175, 238, 238),  # Pale turquoise
        (255, 228, 196)   # Bisque
    ]},
    {"name": "Metallic", "colors": [
        (192, 192, 192),  # Silver
        (169, 169, 169),  # Dark gray
        (218, 165, 32),   # Goldenrod
        (184, 134, 11),   # Dark goldenrod
        (139, 137, 137)   # Light slate gray
    ]}
]

current_skin_index = 0  # Track currently selected skin       

# Snake Part Database (10 heads, 10 bodies, 5 tails) - Shapes only
SNAKE_PARTS = {
    "heads": [
        {"name": "Classic", "shape": "circle"},
        {"name": "Viper", "shape": "triangle"},
        {"name": "Cobra", "shape": "diamond"},
        {"name": "Dragon", "shape": "square"},
        {"name": "Alien", "shape": "hexagon"},
        {"name": "Robot", "shape": "gear"},
        {"name": "Demon", "shape": "spade"},
        {"name": "Angel", "shape": "heart"},
        {"name": "Shark", "shape": "shark"},
        {"name": "Skull", "shape": "skull"}
    ],
    "bodies": [
        {"name": "Classic", "pattern": "solid"},
        {"name": "Striped", "pattern": "stripes"},
        {"name": "Spotted", "pattern": "spots"},
        {"name": "Gradient", "pattern": "gradient"},
        {"name": "Metallic", "pattern": "metallic"},
        {"name": "Camo", "pattern": "camo"},
        {"name": "Glass", "pattern": "glass"},
        {"name": "Scales", "pattern": "scales"},
        {"name": "Banded", "pattern": "bands"},
        {"name": "Neon", "pattern": "glow"}
    ],
    "tails": [
        {"name": "Classic", "shape": "point"},
        {"name": "Rattle", "shape": "rattle"},
        {"name": "Forked", "shape": "fork"},
        {"name": "Spade", "shape": "spade"},
        {"name": "Arrow", "shape": "arrow"}
    ]
}

# Initialize current part indices AFTER defining SNAKE_PARTS
current_head_index = 0
current_body_index = 0
current_tail_index = 0
current_tab = 0  # 0=Head, 1=Body, 2=Tail

class UserProfile:
    def __init__(self, username, user_type):
        self.username = username
        self.user_type = user_type
        self.profile_key = f"{user_type}_{username}"
        
        # Initialize profile if it doesn't exist
        if self.profile_key not in profile_data:
            profile_data[self.profile_key] = {
                "avatar_index": random.randint(0, len(DEFAULT_AVATARS)-1),
                "level": 1,
                "exp": 0,
                "total_exp": 0,
                "high_score": 0,
                "games_played": 0,
                "last_played": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            }
            save_profile_data()
        
        self.data = profile_data[self.profile_key]
    
    def add_exp(self, amount):
        """Add experience points and level up if needed"""
        self.data["exp"] += amount
        self.data["total_exp"] += amount
        exp_needed = self.get_exp_needed()
        
        if self.data["exp"] >= exp_needed:
            self.data["level"] += 1
            self.data["exp"] -= exp_needed
            return True  # Level up occurred
        return False
    
    def get_exp_needed(self):
        """Calculate experience needed for next level"""
        return 100 * (2 ** (self.data["level"] - 1))
    
    def update_high_score(self, score):
        """Update high score if new score is higher"""
        if score > self.data["high_score"]:
            self.data["high_score"] = score
            return True
        return False
    
    def increment_games_played(self):
        """Increment games played counter"""
        self.data["games_played"] += 1
        self.data["last_played"] = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    def set_avatar(self, index):
        """Set avatar index (0-9 for default avatars)"""
        if 0 <= index < len(DEFAULT_AVATARS):
            self.data["avatar_index"] = index
            return True
        return False
    
    def get_avatar_surface(self, size):
        """Generate avatar surface with given size"""
        avatar = DEFAULT_AVATARS[self.data["avatar_index"]]
        surface = pygame.Surface((size, size), pygame.SRCALPHA)
        
        # Draw face circle
        pygame.draw.circle(surface, avatar["color"], (size//2, size//2), size//2)
        
        # Draw eyes based on type
        eye_type = avatar["eyes"]
        eye_size = size // 5
        left_eye_pos = (size//2 - size//4, size//2 - size//8)
        right_eye_pos = (size//2 + size//4, size//2 - size//8)
        
        if eye_type == "happy":
            # Smiling eyes
            pygame.draw.ellipse(surface, BLACK, 
                               (left_eye_pos[0] - eye_size//2, left_eye_pos[1] - eye_size//4, 
                                eye_size, eye_size//2))
            pygame.draw.ellipse(surface, BLACK, 
                               (right_eye_pos[0] - eye_size//2, right_eye_pos[1] - eye_size//4, 
                                eye_size, eye_size//2))
        elif eye_type == "angry":
            # Angry eyes (diagonal lines)
            pygame.draw.line(surface, BLACK, 
                            (left_eye_pos[0] - eye_size//2, left_eye_pos[1] + eye_size//4),
                            (left_eye_pos[0] + eye_size//2, left_eye_pos[1] - eye_size//4), 3)
            pygame.draw.line(surface, BLACK, 
                            (right_eye_pos[0] - eye_size//2, right_eye_pos[1] - eye_size//4),
                            (right_eye_pos[0] + eye_size//2, right_eye_pos[1] + eye_size//4), 3)
        elif eye_type == "surprised":
            # Circular eyes
            pygame.draw.circle(surface, BLACK, left_eye_pos, eye_size//2)
            pygame.draw.circle(surface, BLACK, right_eye_pos, eye_size//2)
        else:  # normal
            # Neutral eyes (horizontal lines)
            pygame.draw.line(surface, BLACK, 
                            (left_eye_pos[0] - eye_size//2, left_eye_pos[1]),
                            (left_eye_pos[0] + eye_size//2, left_eye_pos[1]), 3)
            pygame.draw.line(surface, BLACK, 
                            (right_eye_pos[0] - eye_size//2, right_eye_pos[1]),
                            (right_eye_pos[0] + eye_size//2, right_eye_pos[1]), 3)
        
        # Draw mouth
        mouth_y = size//2 + size//4
        if eye_type == "happy":
            # Smile
            pygame.draw.arc(surface, BLACK, 
                           (size//4, mouth_y - size//8, size//2, size//4),
                           0, math.pi, 2)
        elif eye_type == "angry":
            # Frown
            pygame.draw.arc(surface, BLACK, 
                           (size//4, mouth_y, size//2, size//4),
                           math.pi, 2*math.pi, 2)
        else:
            # Straight line
            pygame.draw.line(surface, BLACK, 
                            (size//4, mouth_y), 
                            (3*size//4, mouth_y), 2)
        
        return surface
    
    def save(self):
        """Save profile data"""
        profile_data[self.profile_key] = self.data
        save_profile_data()

def show_profile_popup(profile):
    """Display the profile popup with user information"""
    popup_width = int(WIDTH * 0.6)
    popup_height = int(HEIGHT * 0.6)
    popup_x = (WIDTH - popup_width) // 2
    popup_y = (HEIGHT - popup_height) // 2
    
    while True:  # Main loop for profile popup
        # Draw video background continuously
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Shadow effect
        shadow = pygame.Surface((popup_width + 10, popup_height + 10), pygame.SRCALPHA)
        shadow.fill((0, 0, 0, 100))
        screen.blit(shadow, (popup_x - 5, popup_y - 5))
        
        # Main popup background
        pygame.draw.rect(screen, (240, 240, 250), (popup_x, popup_y, popup_width, popup_height), border_radius=20)
        
        # Header
        header_rect = pygame.Rect(popup_x, popup_y, popup_width, int(popup_height * 0.15))
        pygame.draw.rect(screen, (70, 130, 180), header_rect, border_top_left_radius=20, border_top_right_radius=20)
        
        title = font.render("Player Profile", True, WHITE)
        screen.blit(title, (popup_x + (popup_width - title.get_width()) // 2, popup_y + 20))
        
        # Close button
        close_button = pygame.Rect(popup_x + popup_width - 50, popup_y + 15, 30, 30)
        pygame.draw.rect(screen, (200, 50, 50), close_button, border_radius=15)
        close_text = font.render("X", True, WHITE)
        screen.blit(close_text, (close_button.centerx - close_text.get_width() // 2, 
                                close_button.centery - close_text.get_height() // 2))
        
        # Avatar section (left side) - reduced size
        avatar_size = int(popup_height * 0.3)
        avatar_x = popup_x + int(popup_width * 0.15)
        avatar_y = popup_y + int(popup_height * 0.25)
        
        avatar_surface = profile.get_avatar_surface(avatar_size)
        screen.blit(avatar_surface, (avatar_x - avatar_size//2, avatar_y - avatar_size//2))
        
        # Only show change avatar button for non-guest users
        if profile.user_type != "guest":
            change_avatar_button = pygame.Rect(avatar_x - 80, avatar_y + avatar_size//2 + 20, 160, 40)
            pygame.draw.rect(screen, BUTTON_COLOR, change_avatar_button, border_radius=10)
            change_text = small_font.render("Change Avatar", True, WHITE)
            screen.blit(change_text, (change_avatar_button.centerx - change_text.get_width() // 2, 
                                    change_avatar_button.centery - change_text.get_height() // 2))
        else:
            change_avatar_button = None
        
        # Info section (right side)
        info_x = popup_x + int(popup_width * 0.55)
        info_y = popup_y + int(popup_height * 0.25)
        
        # Username
        username_text = font.render(f"Name: {profile.username}", True, BLACK)
        screen.blit(username_text, (info_x - username_text.get_width() // 2, info_y))
        
        # Level
        level_text = font.render(f"Level: {profile.data['level']}", True, BLACK)
        screen.blit(level_text, (info_x - level_text.get_width() // 2, info_y + 50))
        
        # EXP progress bar
        exp_needed = profile.get_exp_needed()
        exp_percent = (profile.data["exp"] / exp_needed) * 100
        exp_text = small_font.render(f"EXP: {profile.data['exp']}/{exp_needed} ({exp_percent:.1f}%)", True, BLACK)
        screen.blit(exp_text, (info_x - exp_text.get_width() // 2, info_y + 90))
        
        # Progress bar background
        progress_width = int(popup_width * 0.3)
        progress_rect = pygame.Rect(info_x - progress_width // 2, info_y + 120, progress_width, 20)
        pygame.draw.rect(screen, (200, 200, 200), progress_rect, border_radius=10)
        
        # Progress bar fill
        fill_width = int(progress_width * (exp_percent / 100))
        fill_rect = pygame.Rect(info_x - progress_width // 2, info_y + 120, fill_width, 20)
        pygame.draw.rect(screen, (50, 200, 50), fill_rect, border_radius=10)
        
        # High score
        high_score_text = font.render(f"High Score: {profile.data['high_score']}", True, BLACK)
        screen.blit(high_score_text, (info_x - high_score_text.get_width() // 2, info_y + 160))
        
        # Games played
        games_text = font.render(f"Games Played: {profile.data['games_played']}", True, BLACK)
        screen.blit(games_text, (info_x - games_text.get_width() // 2, info_y + 210))
        
        # Footer buttons
        footer_y = popup_y + popup_height - 70
        logout_button = pygame.Rect(popup_x + popup_width - 150, footer_y, 120, 50)
        pygame.draw.rect(screen, (200, 50, 50), logout_button, border_radius=10)
        logout_text = font.render("Logout", True, WHITE)
        screen.blit(logout_text, (logout_button.centerx - logout_text.get_width() // 2, 
                                logout_button.centery - logout_text.get_height() // 2))
        
        back_button = pygame.Rect(popup_x + 30, footer_y, 120, 50)
        pygame.draw.rect(screen, BUTTON_COLOR, back_button, border_radius=10)
        back_text = font.render("Back", True, WHITE)
        screen.blit(back_text, (back_button.centerx - back_text.get_width() // 2, 
                               back_button.centery - back_text.get_height() // 2))
        
        pygame.display.flip()
        
        # Handle events
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if close_button.collidepoint(pos):
                    return "close"
                if back_button.collidepoint(pos):
                    return "back"
                if logout_button.collidepoint(pos):
                    return "logout"
                if change_avatar_button and change_avatar_button.collidepoint(pos):
                    # Call avatar selection without pausing video
                    show_avatar_selection(profile)
                    # Redraw immediately after returning
                    break
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                return "close"

def show_avatar_selection(profile):
    """Display avatar selection screen with continuous video"""
    popup_width = int(WIDTH * 0.8)
    popup_height = int(HEIGHT * 0.7)
    popup_x = (WIDTH - popup_width) // 2
    popup_y = (HEIGHT - popup_height) // 2
    
    # Pre-render all avatars
    avatar_size = int(popup_height * 0.25)
    avatar_surfaces = []
    for avatar_data in DEFAULT_AVATARS:
        surface = pygame.Surface((avatar_size, avatar_size), pygame.SRCALPHA)
        
        # Draw face circle
        pygame.draw.circle(surface, avatar_data["color"], (avatar_size//2, avatar_size//2), avatar_size//2)
        
        # Draw eyes based on type
        eye_type = avatar_data["eyes"]
        eye_size = avatar_size // 5
        left_eye_pos = (avatar_size//2 - avatar_size//4, avatar_size//2 - avatar_size//8)
        right_eye_pos = (avatar_size//2 + avatar_size//4, avatar_size//2 - avatar_size//8)
        
        if eye_type == "happy":
            # Smiling eyes
            pygame.draw.ellipse(surface, BLACK, 
                               (left_eye_pos[0] - eye_size//2, left_eye_pos[1] - eye_size//4, 
                                eye_size, eye_size//2))
            pygame.draw.ellipse(surface, BLACK, 
                               (right_eye_pos[0] - eye_size//2, right_eye_pos[1] - eye_size//4, 
                                eye_size, eye_size//2))
        elif eye_type == "angry":
            # Angry eyes (diagonal lines)
            pygame.draw.line(surface, BLACK, 
                            (left_eye_pos[0] - eye_size//2, left_eye_pos[1] + eye_size//4),
                            (left_eye_pos[0] + eye_size//2, left_eye_pos[1] - eye_size//4), 3)
            pygame.draw.line(surface, BLACK, 
                            (right_eye_pos[0] - eye_size//2, right_eye_pos[1] - eye_size//4),
                            (right_eye_pos[0] + eye_size//2, right_eye_pos[1] + eye_size//4), 3)
        elif eye_type == "surprised":
            # Circular eyes
            pygame.draw.circle(surface, BLACK, left_eye_pos, eye_size//2)
            pygame.draw.circle(surface, BLACK, right_eye_pos, eye_size//2)
        else:  # normal
            # Neutral eyes (horizontal lines)
            pygame.draw.line(surface, BLACK, 
                            (left_eye_pos[0] - eye_size//2, left_eye_pos[1]),
                            (left_eye_pos[0] + eye_size//2, left_eye_pos[1]), 3)
            pygame.draw.line(surface, BLACK, 
                            (right_eye_pos[0] - eye_size//2, right_eye_pos[1]),
                            (right_eye_pos[0] + eye_size//2, right_eye_pos[1]), 3)
        
        # Draw mouth
        mouth_y = avatar_size//2 + avatar_size//4
        if eye_type == "happy":
            # Smile
            pygame.draw.arc(surface, BLACK, 
                           (avatar_size//4, mouth_y - avatar_size//8, 
                            avatar_size//2, avatar_size//4),
                           0, math.pi, 2)
        elif eye_type == "angry":
            # Frown
            pygame.draw.arc(surface, BLACK, 
                           (avatar_size//4, mouth_y, 
                            avatar_size//2, avatar_size//4),
                           math.pi, 2*math.pi, 2)
        else:
            # Straight line
            pygame.draw.line(surface, BLACK, 
                            (avatar_size//4, mouth_y), 
                            (3*avatar_size//4, mouth_y), 2)
        
        avatar_surfaces.append(surface)
    
    # Initial setup
    avatar_buttons = [None] * len(DEFAULT_AVATARS)
    apply_button = pygame.Rect(popup_x + popup_width - 220, popup_y + popup_height - 80, 180, 50)
    apply_text = font.render("Apply", True, WHITE)
    cancel_button = pygame.Rect(popup_x + 40, popup_y + popup_height - 80, 180, 50)
    cancel_text = font.render("Cancel", True, WHITE)
    
    # Track selected avatar
    selected_index = profile.data["avatar_index"]
    
    # Draw the full popup
    close_button = pygame.Rect(popup_x + popup_width - 50, popup_y + 15, 30, 30)
    
    # Handle events
    while True:
        # Draw video background continuously
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Draw shadow and background
        shadow = pygame.Surface((popup_width + 10, popup_height + 10), pygame.SRCALPHA)
        shadow.fill((0, 0, 0, 100))
        screen.blit(shadow, (popup_x - 5, popup_y - 5))
        
        pygame.draw.rect(screen, (240, 240, 250), (popup_x, popup_y, popup_width, popup_height), border_radius=20)
        
        # Draw header
        header_rect = pygame.Rect(popup_x, popup_y, popup_width, int(popup_height * 0.15))
        pygame.draw.rect(screen, (70, 130, 180), header_rect, border_top_left_radius=20, border_top_right_radius=20)
        
        # Draw title
        title = font.render("Select Avatar", True, WHITE)
        screen.blit(title, (popup_x + (popup_width - title.get_width()) // 2, popup_y + 20))
        
        # Draw close button
        pygame.draw.rect(screen, (200, 50, 50), close_button, border_radius=15)
        close_text = font.render("X", True, WHITE)
        screen.blit(close_text, (close_button.centerx - close_text.get_width() // 2, 
                                close_button.centery - close_text.get_height() // 2))
        
        # Draw avatar grid
        avatar_spacing = int(popup_width * 0.1)
        start_x = popup_x + (popup_width - (5 * avatar_size + 4 * avatar_spacing)) // 2
        start_y = popup_y + int(popup_height * 0.25)
        
        for i in range(len(DEFAULT_AVATARS)):
            row = i // 5
            col = i % 5
            x = start_x + col * (avatar_size + avatar_spacing)
            y = start_y + row * (avatar_size + avatar_spacing)
            
            # Draw avatar
            screen.blit(avatar_surfaces[i], (x, y))
            
            # Highlight selected avatar
            if i == selected_index:
                pygame.draw.rect(screen, (50, 200, 50), (x - 5, y - 5, avatar_size + 10, avatar_size + 10), 3, border_radius=avatar_size//2)
            
            # Store button rect
            avatar_buttons[i] = pygame.Rect(x, y, avatar_size, avatar_size)
        
        # Draw buttons
        pygame.draw.rect(screen, (50, 150, 50), apply_button, border_radius=10)
        screen.blit(apply_text, (apply_button.centerx - apply_text.get_width() // 2, 
                                apply_button.centery - apply_text.get_height() // 2))
        pygame.draw.rect(screen, (200, 50, 50), cancel_button, border_radius=10)
        screen.blit(cancel_text, (cancel_button.centerx - cancel_text.get_width() // 2, 
                                 cancel_button.centery - cancel_text.get_height() // 2))
        
        pygame.display.flip()
        
        # Handle events
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if close_button.collidepoint(pos) or cancel_button.collidepoint(pos):
                    return  # Return to profile popup
                if apply_button.collidepoint(pos):
                    profile.set_avatar(selected_index)
                    profile.save()
                    return  # Return to profile popup with updated avatar
                
                # Check avatar clicks
                for i, button in enumerate(avatar_buttons):
                    if button and button.collidepoint(pos):
                        selected_index = i
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                return  # Return to profile popup
    
    def draw_full_popup():
        """Helper function to draw the entire popup"""
        # Draw video background continuously
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Draw shadow and background
        shadow = pygame.Surface((popup_width + 10, popup_height + 10), pygame.SRCALPHA)
        shadow.fill((0, 0, 0, 100))
        screen.blit(shadow, (popup_x - 5, popup_y - 5))
        
        pygame.draw.rect(screen, (240, 240, 250), (popup_x, popup_y, popup_width, popup_height), border_radius=20)
        
        # Draw header
        header_rect = pygame.Rect(popup_x, popup_y, popup_width, int(popup_height * 0.15))
        pygame.draw.rect(screen, (70, 130, 180), header_rect, border_top_left_radius=20, border_top_right_radius=20)
        
        # Draw title
        title = font.render("Select Avatar", True, WHITE)
        screen.blit(title, (popup_x + (popup_width - title.get_width()) // 2, popup_y + 20))
        
        # Draw close button
        close_button = pygame.Rect(popup_x + popup_width - 50, popup_y + 15, 30, 30)
        pygame.draw.rect(screen, (200, 50, 50), close_button, border_radius=15)
        close_text = font.render("X", True, WHITE)
        screen.blit(close_text, (close_button.centerx - close_text.get_width() // 2, 
                                close_button.centery - close_text.get_height() // 2))
        
        # Draw avatar grid
        avatar_spacing = int(popup_width * 0.1)
        start_x = popup_x + (popup_width - (5 * avatar_size + 4 * avatar_spacing)) // 2
        start_y = popup_y + int(popup_height * 0.25)
        
        for i in range(len(DEFAULT_AVATARS)):
            row = i // 5
            col = i % 5
            x = start_x + col * (avatar_size + avatar_spacing)
            y = start_y + row * (avatar_size + avatar_spacing)
            
            # Draw avatar
            screen.blit(avatar_surfaces[i], (x, y))
            
            # Highlight selected avatar
            if i == selected_index:
                pygame.draw.rect(screen, (50, 200, 50), (x - 5, y - 5, avatar_size + 10, avatar_size + 10), 3, border_radius=avatar_size//2)
            
            # Store button rect
            avatar_buttons[i] = pygame.Rect(x, y, avatar_size, avatar_size)
        
        # Draw buttons
        pygame.draw.rect(screen, (50, 150, 50), apply_button, border_radius=10)
        screen.blit(apply_text, (apply_button.centerx - apply_text.get_width() // 2, 
                                apply_button.centery - apply_text.get_height() // 2))
        pygame.draw.rect(screen, (200, 50, 50), cancel_button, border_radius=10)
        screen.blit(cancel_text, (cancel_button.centerx - cancel_text.get_width() // 2, 
                                 cancel_button.centery - cancel_text.get_height() // 2))
        
        return close_button, apply_button, cancel_button
    
    # Initial setup
    avatar_buttons = [None] * len(DEFAULT_AVATARS)
    apply_button = pygame.Rect(popup_x + popup_width - 220, popup_y + popup_height - 80, 180, 50)
    apply_text = font.render("Apply", True, WHITE)
    cancel_button = pygame.Rect(popup_x + 40, popup_y + popup_height - 80, 180, 50)
    cancel_text = font.render("Cancel", True, WHITE)
    
    # Track selected avatar
    selected_index = profile.data["avatar_index"]
    
    # Draw the full popup
    close_button, apply_button, cancel_button = draw_full_popup()
    pygame.display.flip()
    
    # Handle events
    running = True
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if close_button.collidepoint(pos) or cancel_button.collidepoint(pos):
                    running = False
                if apply_button.collidepoint(pos):
                    profile.set_avatar(selected_index)
                    profile.save()
                    running = False
                
                # Check avatar clicks
                for i, button in enumerate(avatar_buttons):
                    if button and button.collidepoint(pos):
                        selected_index = i
                        # Redraw with new selection
                        close_button, apply_button, cancel_button = draw_full_popup()
                        pygame.display.flip()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                running = False

def show_logout_confirmation():
    global video_playing
    # Pause video playback
    video_playing = False
    
    """Display logout confirmation dialog"""
    popup_width = int(WIDTH * 0.4)
    popup_height = int(HEIGHT * 0.3)
    popup_x = (WIDTH - popup_width) // 2
    popup_y = (HEIGHT - popup_height) // 2
    
    # Shadow effect
    shadow = pygame.Surface((popup_width + 10, popup_height + 10), pygame.SRCALPHA)
    shadow.fill((0, 0, 0, 100))
    screen.blit(shadow, (popup_x - 5, popup_y - 5))
    
    # Main popup background
    pygame.draw.rect(screen, (240, 240, 250), (popup_x, popup_y, popup_width, popup_height), border_radius=20)
    
    # Message
    message = font.render("Do you want to log out?", True, BLACK)
    screen.blit(message, (popup_x + (popup_width - message.get_width()) // 2, popup_y + 50))
    
    # Buttons
    yes_button = pygame.Rect(popup_x + popup_width // 2 - 120, popup_y + popup_height - 80, 100, 50)
    pygame.draw.rect(screen, (200, 50, 50), yes_button, border_radius=10)
    yes_text = font.render("Yes", True, WHITE)
    screen.blit(yes_text, (yes_button.centerx - yes_text.get_width() // 2, 
                          yes_button.centery - yes_text.get_height() // 2))
    
    no_button = pygame.Rect(popup_x + popup_width // 2 + 20, popup_y + popup_height - 80, 100, 50)
    pygame.draw.rect(screen, BUTTON_COLOR, no_button, border_radius=10)
    no_text = font.render("No", True, WHITE)
    screen.blit(no_text, (no_button.centerx - no_text.get_width() // 2, 
                         no_button.centery - no_text.get_height() // 2))
    
    pygame.display.flip()
    
    # Handle events
    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if yes_button.collidepoint(pos):
                    # Resume video playback
                    video_playing = True
                    return True
                if no_button.collidepoint(pos):
                    # Resume video playback
                    video_playing = True
                    return False
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_y or event.key == pygame.K_RETURN:
                    # Resume video playback
                    video_playing = True
                    return True
                if event.key == pygame.K_n or event.key == pygame.K_ESCAPE:
                    # Resume video playback
                    video_playing = True
                    return False

def draw_reference_points():
    """Draw small reference points at key grid intersections"""
    point_color = (150, 150, 150)  # Light gray color for reference points
    point_radius = 2  # Small dot size
    
    # Draw points at every 5th grid intersection
    for x in range(0, WIDTH, BLOCK_SIZE * 5):
        for y in range(0, HEIGHT, BLOCK_SIZE * 5):
            pygame.draw.circle(screen, point_color, (x, y), point_radius)
    
    # Draw points along horizontal center line
    for x in range(0, WIDTH, BLOCK_SIZE):
        pygame.draw.circle(screen, point_color, (x, HEIGHT // 2), point_radius)
    
    # Draw points along vertical center line
    for y in range(0, HEIGHT, BLOCK_SIZE):
        pygame.draw.circle(screen, point_color, (WIDTH // 2, y), point_radius)

def draw_head_part(surface, pos, size, head_data, direction="RIGHT", color=None):
    """Draw a snake head based on its shape type with consistent coloring"""
    # Use provided color or default to red
    head_color = color or (255, 50, 50)
    eye_color = WHITE
    
    center_x, center_y = pos[0] + size // 2, pos[1] + size // 2
    eye_size = size // 5
    offset = size // 4
    
    # Determine eye positions based on direction
    if direction == "RIGHT":
        eye1 = (center_x + offset, center_y - offset)
        eye2 = (center_x + offset, center_y + offset)
        angle = 0
    elif direction == "LEFT":
        eye1 = (center_x - offset, center_y - offset)
        eye2 = (center_x - offset, center_y + offset)
        angle = 180
    elif direction == "UP":
        eye1 = (center_x - offset, center_y - offset)
        eye2 = (center_x + offset, center_y - offset)
        angle = 90
    else:  # DOWN
        eye1 = (center_x - offset, center_y + offset)
        eye2 = (center_x + offset, center_y + offset)
        angle = 270
    
    # Draw the head shape with the selected color
    if head_data["shape"] == "circle":
        pygame.draw.circle(surface, head_color, (center_x, center_y), size // 2)
    elif head_data["shape"] == "triangle":
        points = [
            (center_x + size//2 * math.cos(math.radians(angle)), 
             center_y - size//2 * math.sin(math.radians(angle))),
            (center_x + size//2 * math.cos(math.radians(angle+120)), 
             center_y - size//2 * math.sin(math.radians(angle+120))),
            (center_x + size//2 * math.cos(math.radians(angle+240)), 
             center_y - size//2 * math.sin(math.radians(angle+240)))
        ]
        pygame.draw.polygon(surface, head_color, points)
    elif head_data["shape"] == "diamond":
        # Fixed polygon points to be proper (x,y) tuples
        points = [
            (center_x, center_y - size//2),
            (center_x + size//2, center_y),
            (center_x, center_y + size//2),
            (center_x - size//2, center_y)
        ]
        pygame.draw.polygon(surface, head_color, points)
    elif head_data["shape"] == "square":
        pygame.draw.rect(surface, head_color, (pos[0], pos[1], size, size))
    elif head_data["shape"] == "hexagon":
        points = []
        for i in range(6):
            # Create proper (x,y) tuples
            x = center_x + size//2 * math.cos(math.radians(angle + i*60))
            y = center_y + size//2 * math.sin(math.radians(angle + i*60))
            points.append((x, y))
        pygame.draw.polygon(surface, head_color, points)
    elif head_data["shape"] == "gear":
        # Draw gear shape
        pygame.draw.circle(surface, head_color, (center_x, center_y), size // 2)
        for i in range(8):
            angle_rad = math.radians(i * 45)
            start_x = center_x + (size//3) * math.cos(angle_rad)
            start_y = center_y + (size//3) * math.sin(angle_rad)
            end_x = center_x + (size//2) * math.cos(angle_rad)
            end_y = center_y + (size//2) * math.sin(angle_rad)
            pygame.draw.line(surface, BLACK, (start_x, start_y), (end_x, end_y), 3)
    elif head_data["shape"] == "spade":
        # Fixed polygon points to be proper (x,y) tuples
        points = [
            (center_x, center_y - size//2),
            (center_x + size//2, center_y),
            (center_x, center_y + size//4),
            (center_x - size//2, center_y)
        ]
        pygame.draw.polygon(surface, head_color, points)
    elif head_data["shape"] == "heart":
        # Draw heart shape
        pygame.draw.circle(surface, head_color, (center_x - size//4, center_y - size//4), size//4)
        pygame.draw.circle(surface, head_color, (center_x + size//4, center_y - size//4), size//4)
        # Fixed polygon points to be proper (x,y) tuples
        points = [
            (center_x - size//4, center_y - size//8),
            (center_x + size//4, center_y - size//8),
            (center_x + size//2, center_y + size//4),
            (center_x, center_y + size//2),
            (center_x - size//2, center_y + size//4)
        ]
        pygame.draw.polygon(surface, head_color, points)
    elif head_data["shape"] == "shark":
        # Draw shark head
        pygame.draw.ellipse(surface, head_color, (pos[0], pos[1], size, size))
        # Draw fin
        # Fixed polygon points to be proper (x,y) tuples
        pygame.draw.polygon(surface, head_color, [
            (center_x - size//4, center_y - size//2),
            (center_x, center_y - size//3),
            (center_x + size//4, center_y - size//2)
        ])
    elif head_data["shape"] == "skull":
        # Draw skull
        pygame.draw.circle(surface, head_color, (center_x, center_y), size//2)
        # Draw eye sockets
        pygame.draw.circle(surface, BLACK, (center_x - size//4, center_y), size//5)
        pygame.draw.circle(surface, BLACK, (center_x + size//4, center_y), size//5)
        # Draw nose - fixed polygon points
        pygame.draw.polygon(surface, BLACK, [
            (center_x - size//8, center_y + size//8),
            (center_x + size//8, center_y + size//8),
            (center_x, center_y + size//4)
        ])
        # Draw teeth
        pygame.draw.rect(surface, WHITE, (center_x - size//3, center_y + size//4, size//6, size//5))
        pygame.draw.rect(surface, WHITE, (center_x - size//12, center_y + size//4, size//6, size//5))
        pygame.draw.rect(surface, WHITE, (center_x + size//12, center_y + size//4, size//6, size//5))
    
    # Draw eyes on all head types
    pygame.draw.circle(surface, eye_color, eye1, eye_size)
    pygame.draw.circle(surface, eye_color, eye2, eye_size)
    pygame.draw.circle(surface, (0, 0, 0), eye1, eye_size // 2)
    pygame.draw.circle(surface, (0, 0, 0), eye2, eye_size // 2)

def show_skin_selection():
    """Display the skin selection screen with proper coloring"""
    screen.fill((0, 30, 60))
    title = big_font.render("Select Snake Skin", True, WHITE)
    screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.08))
    
    # Display current skin
    current_skin = SNAKE_SKINS[int(current_skin_index)]
    skin_name = font.render(current_skin["name"], True, WHITE)
    screen.blit(skin_name, (WIDTH // 2 - skin_name.get_width() // 2, HEIGHT * 0.15))
    
    # Draw preview snake
    preview_size = int(200 * SCALE_FACTOR)
    preview_start_x = WIDTH // 2 - preview_size // 2
    preview_y = HEIGHT * 0.2
    block_size = int(30 * SCALE_FACTOR)
    
    # Get current shapes
    head_part = SNAKE_PARTS["heads"][current_head_index]
    body_part = SNAKE_PARTS["bodies"][current_body_index]
    tail_part = SNAKE_PARTS["tails"][current_tail_index]
    
    # Create colors for preview
    if "color" in current_skin:
        # Single color skin
        primary_color = current_skin["color"]
        secondary_color = tuple(max(0, c - 40) for c in primary_color)
    else:
        # Multi-color skin - use first color as primary
        primary_color = current_skin["colors"][0]
        secondary_color = tuple(max(0, c - 40) for c in primary_color)
    
    # Draw a mini snake (5 segments)
    for i in range(5):
        x = preview_start_x + i * block_size
        y = preview_y
        
        # For multi-color skins, select color for this segment
        if "colors" in current_skin and len(current_skin["colors"]) > 1:
            segment_color = current_skin["colors"][i % len(current_skin["colors"])]
            body_primary = segment_color
            body_secondary = tuple(max(0, c - 40) for c in body_primary)
        else:
            segment_color = primary_color
            body_primary = primary_color
            body_secondary = secondary_color
        
        if i == 0:  # Head
            draw_head_part(screen, (x, y), block_size, head_part, "RIGHT", segment_color)
        elif i == 4:  # Tail
            draw_tail_part(screen, (x, y), block_size, tail_part, "RIGHT", segment_color)
        else:  # Body
            draw_body_part(screen, (x, y), block_size, body_part, body_primary, body_secondary)

def draw_tail_part(surface, pos, size, tail_data, direction="RIGHT", color=None):
    """Draw the tail segment based on its type with consistent size"""
    # Use provided color or default to red
    tail_color = color or (255, 50, 50)
    
    x, y = pos
    center_x, center_y = x + size // 2, y + size // 2
    
    if tail_data["shape"] == "point":
        # Pointing away from the body segment
        if direction == "RIGHT":
            points = [
                (x + size, y + size//2),   # Right point
                (x, y),                     # Top-left
                (x, y + size)               # Bottom-left
            ]
        elif direction == "LEFT":
            points = [
                (x, y + size//2),           # Left point
                (x + size, y),              # Top-right
                (x + size, y + size)         # Bottom-right
            ]
        elif direction == "UP":
            points = [
                (x + size//2, y),           # Top point
                (x, y + size),              # Bottom-left
                (x + size, y + size)         # Bottom-right
            ]
        else:  # DOWN
            points = [
                (x + size//2, y + size),    # Bottom point
                (x, y),                     # Top-left
                (x + size, y)               # Top-right
            ]
        pygame.draw.polygon(surface, tail_color, points)
    
    elif tail_data["shape"] == "spade":
        # Spade shape pointing away from body
        if direction == "RIGHT":
            points = [
                (x + size, y + size//2),    # Right point
                (x, y),                     # Top-left
                (x, y + size),              # Bottom-left
                (x + size//2, y + size),     # Bottom center
                (x + size, y + size//2)      # Right center
            ]
        elif direction == "LEFT":
            points = [
                (x, y + size//2),           # Left point
                (x + size, y),              # Top-right
                (x + size, y + size),       # Bottom-right
                (x + size//2, y + size),     # Bottom center
                (x, y + size//2)            # Left center
            ]
        elif direction == "UP":
            points = [
                (x + size//2, y),           # Top point
                (x, y + size),              # Bottom-left
                (x + size, y + size),       # Bottom-right
                (x + size, y),              # Top-right
                (x, y)                      # Top-left
            ]
        else:  # DOWN
            points = [
                (x + size//2, y + size),    # Bottom point
                (x, y),                     # Top-left
                (x + size, y),              # Top-right
                (x + size, y + size),       # Bottom-right
                (x, y + size)               # Bottom-left
            ]
        pygame.draw.polygon(surface, tail_color, points)
    
    elif tail_data["shape"] == "arrow":
        # Arrow shape pointing away from body
        if direction == "RIGHT":
            points = [
                (x + size, y + size//2),     # Right point
                (x, y + size//4),            # Top-left
                (x + size//2, y + size//4),  # Top-center
                (x + size//2, y),            # Top-right
                (x + size, y + size//2),     # Right point
                (x + size//2, y + size),     # Bottom-right
                (x + size//2, y + size*3//4),# Bottom-center
                (x, y + size*3//4)           # Bottom-left
            ]
        elif direction == "LEFT":
            points = [
                (x, y + size//2),            # Left point
                (x + size, y + size//4),     # Top-right
                (x + size//2, y + size//4),  # Top-center
                (x + size//2, y),            # Top-left
                (x, y + size//2),            # Left point
                (x + size//2, y + size),     # Bottom-left
                (x + size//2, y + size*3//4),# Bottom-center
                (x + size, y + size*3//4)    # Bottom-right
            ]
        elif direction == "UP":
            points = [
                (x + size//2, y),            # Top point
                (x + size//4, y + size),     # Bottom-left
                (x + size//4, y + size//2),  # Center-left
                (x, y + size//2),            # Left-center
                (x + size//2, y),            # Top point
                (x + size, y + size//2),     # Right-center
                (x + size*3//4, y + size//2),# Center-right
                (x + size*3//4, y + size)    # Bottom-right
            ]
        else:  # DOWN
            points = [
                (x + size//2, y + size),     # Bottom point
                (x + size//4, y),            # Top-left
                (x + size//4, y + size//2),  # Center-left
                (x, y + size//2),            # Left-center
                (x + size//2, y + size),     # Bottom point
                (x + size, y + size//2),     # Right-center
                (x + size*3//4, y + size//2),# Center-right
                (x + size*3//4, y)           # Top-right
            ]
        pygame.draw.polygon(surface, tail_color, points)
    
    elif tail_data["shape"] == "rattle":
        # Circular rattle
        pygame.draw.circle(surface, tail_color, (center_x, center_y), size//2)
        # Draw rattle segments
        for i in range(3):
            seg_y = y + size//4 + i * (size//4)
            pygame.draw.rect(surface, (200, 200, 200), 
                           (x + size//4, seg_y, size//2, size//8))
    
    elif tail_data["shape"] == "fork":
        # Forked tail
        pygame.draw.rect(surface, tail_color, (x, y, size, size))
        # Draw fork lines
        pygame.draw.line(surface, BLACK, (x, y + size//3), (x + size, y + size//3), 3)
        pygame.draw.line(surface, BLACK, (x, y + size*2//3), (x + size, y + size*2//3), 3)
    
    else:  # Default to circle
        pygame.draw.circle(surface, tail_color, (center_x, center_y), size//2)

def show_shape_selection():
    global current_head_index, current_body_index, current_tail_index, current_tab
    global applied_head_index, applied_body_index, applied_tail_index

    def is_applied():
        return (current_head_index == applied_head_index and
                current_body_index == applied_body_index and
                current_tail_index == applied_tail_index)

    apply_text = "Applied" if is_applied() else "Apply"

    while True:
        screen.fill((0, 30, 60))
        title = big_font.render("Customize Snake Shape", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.08))

        tabs = ["Head", "Body", "Tail"]
        tab_buttons = []
        tab_width = WIDTH // len(tabs)
        for i, tab in enumerate(tabs):
            rect = pygame.Rect(i * tab_width, HEIGHT * 0.15, tab_width, 50 * SCALE_FACTOR)
            tab_buttons.append(rect)
            color = BUTTON_HOVER if i == current_tab else BUTTON_COLOR
            pygame.draw.rect(screen, color, rect)
            label = font.render(tab, True, WHITE)
            screen.blit(label, (rect.centerx - label.get_width() // 2, rect.centery - label.get_height() // 2))

        parts = SNAKE_PARTS["heads"] if current_tab == 0 else (
            SNAKE_PARTS["bodies"] if current_tab == 1 else SNAKE_PARTS["tails"]
        )
        current_index = [current_head_index, current_body_index, current_tail_index][current_tab]
        part_name = parts[current_index]["name"]
        name_label = font.render(part_name, True, WHITE)
        screen.blit(name_label, (WIDTH // 2 - name_label.get_width() // 2, HEIGHT * 0.2))

        cols, rows = (5, 2) if current_tab in [0, 1] else (5, 1)
        cell_width = WIDTH // cols
        grid_height = int(HEIGHT * 0.6)
        cell_height = grid_height // rows
        grid_surface = pygame.Surface((WIDTH, grid_height))
        grid_surface.fill((30, 50, 80))

        current_skin = SNAKE_SKINS[current_skin_index]
        primary_color = current_skin["color"] if "color" in current_skin else current_skin["colors"][0]
        secondary_color = tuple(max(0, c - 40) for c in primary_color)

        for i, part in enumerate(parts):
            row, col = i // cols, i % cols
            x, y = col * cell_width, row * cell_height
            part_size = min(cell_width, cell_height) * 0.8
            part_x = x + (cell_width - part_size) // 2
            part_y = y + (cell_height - part_size) // 2
            if i == current_index:
                pygame.draw.rect(grid_surface, (255, 215, 0), (x, y, cell_width, cell_height), 4)
                cell_color = BUTTON_HOVER
            else:
                cell_color = BUTTON_COLOR
            pygame.draw.rect(grid_surface, cell_color, (x, y, cell_width, cell_height))

            if current_tab == 0:
                draw_head_part(grid_surface, (part_x, part_y), part_size, part, "RIGHT", primary_color)
            elif current_tab == 1:
                draw_body_part(grid_surface, (part_x, part_y), part_size, part, primary_color, secondary_color)
            else:
                draw_tail_part(grid_surface, (part_x, part_y), part_size, part, "RIGHT", primary_color)

        screen.blit(grid_surface, (0, HEIGHT * 0.25))

        apply_button = Button((WIDTH // 2, HEIGHT - int(50 * SCALE_FACTOR)), int(40 * SCALE_FACTOR), text=apply_text)
        back_button = Button((int(50 * SCALE_FACTOR), int(50 * SCALE_FACTOR)), int(30 * SCALE_FACTOR), text="X")
        mouse_pos = pygame.mouse.get_pos()

        for btn in [apply_button, back_button]:
            btn.draw(screen, mouse_pos)

        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                return "back"
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if apply_button.is_clicked(pos):
                    applied_head_index = current_head_index
                    applied_body_index = current_body_index
                    applied_tail_index = current_tail_index
                    apply_text = "Applied"
                elif back_button.is_clicked(pos):
                    return "back"
                for i, tab in enumerate(tab_buttons):
                    if tab.collidepoint(pos):
                        current_tab = i
                        break
                if pos[1] > HEIGHT * 0.25:
                    col = int(pos[0] // cell_width)
                    row = int((pos[1] - HEIGHT * 0.25) // cell_height)
                    index = row * cols + col
                    if index < len(parts):
                        if current_tab == 0:
                            current_head_index = index
                        elif current_tab == 1:
                            current_body_index = index
                        else:
                            current_tail_index = index
                        apply_text = "Applied" if is_applied() else "Apply"

def set_parts(self, head_idx, body_idx, tail_idx):
    """Set the snake parts using the specified indices"""
    # Ensure indices are integers and within range
    head_idx = int(head_idx) % len(SNAKE_PARTS["heads"])
    body_idx = int(body_idx) % len(SNAKE_PARTS["bodies"])
    tail_idx = int(tail_idx) % len(SNAKE_PARTS["tails"])
    
    # Only set shapes, not colors
    self.head_part = {"name": SNAKE_PARTS["heads"][head_idx]["name"],
                     "shape": SNAKE_PARTS["heads"][head_idx]["shape"]}
    self.body_part = {"name": SNAKE_PARTS["bodies"][body_idx]["name"],
                     "pattern": SNAKE_PARTS["bodies"][body_idx]["pattern"]}
    self.tail_part = {"name": SNAKE_PARTS["tails"][tail_idx]["name"],
                     "shape": SNAKE_PARTS["tails"][tail_idx]["shape"]}



def facebook_auth_screen(screen, WIDTH, HEIGHT):
    # User data storage
    users_file = "facebook_users.json"
    
    def load_users():
        try:
            with open(users_file, 'r') as f:
                return json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            return {}

    def save_users(users):
        with open(users_file, 'w') as f:
            json.dump(users, f)

    # Screen states
    STATE_LOGIN = 0
    STATE_SIGNUP = 1
    STATE_FORGOT_PW = 2
    STATE_RESET_PW = 3
    
    current_state = STATE_LOGIN
    username = ""
    password = ""
    pin = ""
    temp_password = ""
    error_msg = ""
    attempts_left = 3
    cursor_visible = True
    cursor_timer = 0
    cursor_pos = 0
    current_field = "username"
    
    # Scale positions
    username_rect = pygame.Rect(WIDTH//2 - 150 * SCALE_FACTOR, 200 * SCALE_FACTOR, 
                               300 * SCALE_FACTOR, 40 * SCALE_FACTOR)
    password_rect = pygame.Rect(WIDTH//2 - 150 * SCALE_FACTOR, 260 * SCALE_FACTOR, 
                               300 * SCALE_FACTOR, 40 * SCALE_FACTOR)
    pin_rect = pygame.Rect(WIDTH//2 - 150 * SCALE_FACTOR, 320 * SCALE_FACTOR, 
                          300 * SCALE_FACTOR, 40 * SCALE_FACTOR)
    forgot_rect = pygame.Rect(WIDTH//2 - 100 * SCALE_FACTOR, 380 * SCALE_FACTOR, 
                             200 * SCALE_FACTOR, 30 * SCALE_FACTOR)
    submit_rect = pygame.Rect(WIDTH//2 - 75 * SCALE_FACTOR, 420 * SCALE_FACTOR, 
                             150 * SCALE_FACTOR, 40 * SCALE_FACTOR)
    switch_rect = pygame.Rect(WIDTH//2 - 100 * SCALE_FACTOR, 470 * SCALE_FACTOR, 
                             200 * SCALE_FACTOR, 30 * SCALE_FACTOR)
    back_rect = pygame.Rect(20 * SCALE_FACTOR, HEIGHT - 50 * SCALE_FACTOR, 
                           80 * SCALE_FACTOR, 30 * SCALE_FACTOR)
    
    font = pygame.font.SysFont("Arial", int(25 * SCALE_FACTOR))
    small_font = pygame.font.SysFont("Arial", int(18 * SCALE_FACTOR))
    clock = pygame.time.Clock()

    def handle_submit():
        nonlocal current_state, error_msg, attempts_left, temp_password
        users = load_users()
        
        if current_state == STATE_LOGIN:
            if username not in users:
                error_msg = "Username not found"
            elif users[username]['password'] != password:
                attempts_left -= 1
                if attempts_left <= 0:
                    return None
                error_msg = f"Wrong password ({attempts_left} attempts left)"
            else:
                return username
                
        elif current_state == STATE_SIGNUP:
            if not username or not password or not pin:
                error_msg = "All fields are required"
            elif username in users:
                error_msg = "Username already exists"
            elif len(pin) != 4 or not pin.isdigit():
                error_msg = "PIN must be 4 digits"
            else:
                users[username] = {
                    'password': password,
                    'pin': pin
                }
                save_users(users)
                return username
                
        elif current_state == STATE_FORGOT_PW:
            if username not in users:
                error_msg = "Username not found"
            elif users[username]['pin'] != pin:
                error_msg = "Invalid PIN"
            else:
                current_state = STATE_RESET_PW
                temp_password = ""
                error_msg = ""
                
        elif current_state == STATE_RESET_PW:
            if len(temp_password) < 6:
                error_msg = "Password must be 6+ characters"
            else:
                users[username]['password'] = temp_password
                save_users(users)
                return username
        return None

    while True:
        # Handle events
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
                
            elif event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                
                # Handle back button
                if back_rect.collidepoint(pos):
                    return None
                
                # Handle field focus
                if username_rect.collidepoint(pos):
                    current_field = "username"
                    cursor_pos = len(username)
                elif password_rect.collidepoint(pos):
                    if current_state == STATE_RESET_PW:
                        current_field = "temp_password"
                    else:
                        current_field = "password"
                    cursor_pos = len(temp_password if current_state == STATE_RESET_PW else password)
                elif pin_rect.collidepoint(pos) and current_state in (STATE_SIGNUP, STATE_FORGOT_PW, STATE_RESET_PW):
                    current_field = "pin"
                    cursor_pos = len(pin)
                
                # Handle submit button
                elif submit_rect.collidepoint(pos):
                    result = handle_submit()
                    if result:
                        return result
                    
                # Handle forgot password link
                elif forgot_rect.collidepoint(pos) and current_state == STATE_LOGIN:
                    current_state = STATE_FORGOT_PW
                    error_msg = ""
                    
                # Handle switch between login/signup
                elif switch_rect.collidepoint(pos):
                    if current_state == STATE_LOGIN:
                        current_state = STATE_SIGNUP
                    else:
                        current_state = STATE_LOGIN
                    error_msg = ""
                    username = ""
                    password = ""
                    pin = ""
                    temp_password = ""
                    
            elif event.type == pygame.KEYDOWN:
                # Handle Enter key - move to next field or submit
                if event.key == pygame.K_RETURN:
                    if current_field == "username":
                        current_field = "password"
                        cursor_pos = len(password)
                    elif current_field == "password" and current_state in (STATE_SIGNUP, STATE_FORGOT_PW):
                        current_field = "pin"
                        cursor_pos = len(pin)
                    else:
                        # Last field - perform submit action
                        result = handle_submit()
                        if result:
                            return result
                    continue
                
                # Handle text input for the current field
                if current_field == "username":
                    text = username
                elif current_field == "password":
                    text = password
                elif current_field == "pin":
                    text = pin
                elif current_field == "temp_password":
                    text = temp_password
                
                if event.key == pygame.K_TAB:
                    fields = ["username", "password"]
                    if current_state in (STATE_SIGNUP, STATE_FORGOT_PW, STATE_RESET_PW):
                        fields.append("pin")
                    if current_state == STATE_RESET_PW:
                        fields[fields.index("password")] = "temp_password"
                    current_idx = fields.index(current_field) if current_field in fields else 0
                    current_field = fields[(current_idx + 1) % len(fields)]
                    cursor_pos = len({
                        "username": username,
                        "password": password,
                        "pin": pin,
                        "temp_password": temp_password
                    }[current_field])
                elif event.key == pygame.K_BACKSPACE:
                    if cursor_pos > 0:
                        if current_field == "username":
                            username = username[:cursor_pos-1] + username[cursor_pos:]
                        elif current_field == "password":
                            password = password[:cursor_pos-1] + password[cursor_pos:]
                        elif current_field == "pin":
                            pin = pin[:cursor_pos-1] + pin[cursor_pos:]
                        elif current_field == "temp_password":
                            temp_password = temp_password[:cursor_pos-1] + temp_password[cursor_pos:]
                        cursor_pos -= 1
                elif event.key == pygame.K_LEFT:
                    cursor_pos = max(0, cursor_pos - 1)
                elif event.key == pygame.K_RIGHT:
                    cursor_pos = min(len(text), cursor_pos + 1)
                elif event.unicode.isprintable():
                    if current_field == "username":
                        username = username[:cursor_pos] + event.unicode + username[cursor_pos:]
                        cursor_pos += 1
                    elif current_field == "password":
                        password = password[:cursor_pos] + event.unicode + password[cursor_pos:]
                        cursor_pos += 1
                    elif current_field == "pin":
                        if event.unicode.isdigit() and len(pin) < 4:
                            pin = pin[:cursor_pos] + event.unicode + pin[cursor_pos:]
                            cursor_pos += 1
                    elif current_field == "temp_password":
                        temp_password = temp_password[:cursor_pos] + event.unicode + temp_password[cursor_pos:]
                        cursor_pos += 1
        
        # Drawing
        screen.fill((20, 20, 50))
        
        # Title based on state
        titles = {
            STATE_LOGIN: "Facebook Login",
            STATE_SIGNUP: "Facebook Sign Up",
            STATE_FORGOT_PW: "Forgot Password",
            STATE_RESET_PW: "Reset Password"
        }
        title = font.render(titles[current_state], True, WHITE)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 140 * SCALE_FACTOR))
        
        # Draw input fields
        def draw_input_box(rect, text, active, is_password=False, placeholder=""):
            pygame.draw.rect(screen, (50, 50, 80), rect)
            border_color = (100, 100, 200) if active else (70, 70, 70)
            pygame.draw.rect(screen, border_color, rect, 2)
            
            display_text = "*" * len(text) if is_password else text
            if not text and not active:
                text_surface = font.render(placeholder, True, (150, 150, 150))
            else:
                text_surface = font.render(display_text, True, WHITE)
            screen.blit(text_surface, (rect.x + 10, rect.y + 10))
            
            if active and pygame.time.get_ticks() % 1000 < 500:  # Blinking cursor
                cursor_x = rect.x + 10 + font.size(display_text[:cursor_pos])[0]
                pygame.draw.line(screen, WHITE, (cursor_x, rect.y + 5), (cursor_x, rect.y + 35), 2)
        
        # Username field (always visible)
        draw_input_box(username_rect, username, current_field == "username", False, "Enter username")
        
        # Password field (visible in login, signup, and reset)
        if current_state in (STATE_LOGIN, STATE_SIGNUP, STATE_RESET_PW):
            if current_state == STATE_RESET_PW:
                # Use temp_password for reset password field
                draw_input_box(
                    password_rect, 
                    temp_password,
                    current_field == "temp_password", 
                    True, 
                    "Enter new password"
                )
            else:
                draw_input_box(
                    password_rect, 
                    password,
                    current_field == "password", 
                    True, 
                    "Enter password"
                )
        
        # PIN field (visible in signup and forgot password)
        if current_state in (STATE_SIGNUP, STATE_FORGOT_PW):
            draw_input_box(
                pin_rect, 
                pin, 
                current_field == "pin", 
                False, 
                "Enter your 4-digit PIN" if current_state == STATE_FORGOT_PW else "Create 4-digit PIN"
            )
        
        # Forgot password link (only in login)
        if current_state == STATE_LOGIN:
            forgot_label = small_font.render("Forgot Password?", True, (150, 150, 255))
            screen.blit(forgot_label, (forgot_rect.centerx - forgot_label.get_width()//2, 
                                      forgot_rect.centery - forgot_label.get_height()//2))
        
        # Submit button
        submit_texts = {
            STATE_LOGIN: "Login",
            STATE_SIGNUP: "Sign Up",
            STATE_FORGOT_PW: "Verify PIN",
            STATE_RESET_PW: "Change Password"
        }
        submit_text = submit_texts[current_state]
        pygame.draw.rect(screen, BUTTON_COLOR, submit_rect, 0, int(10 * SCALE_FACTOR))
        submit_label = font.render(submit_text, True, WHITE)
        screen.blit(submit_label, (submit_rect.centerx - submit_label.get_width()//2, 
                                  submit_rect.centery - submit_label.get_height()//2))
        
        # Switch between login/signup
        switch_text = "Switch to Sign Up" if current_state == STATE_LOGIN else "Switch to Login"
        switch_label = small_font.render(switch_text, True, (150, 150, 255))
        screen.blit(switch_label, (switch_rect.centerx - switch_label.get_width()//2, 
                                 switch_rect.centery - switch_label.get_height()//2))
        
        # Back button
        pygame.draw.rect(screen, (70, 70, 120), back_rect, 0, int(5 * SCALE_FACTOR))
        back_label = small_font.render("← Back", True, WHITE)
        screen.blit(back_label, (back_rect.x + 10, back_rect.y + 8))
        
        # Error message
        if error_msg:
            error_surface = small_font.render(error_msg, True, (255, 50, 50))
            screen.blit(error_surface, (WIDTH//2 - error_surface.get_width()//2, 520 * SCALE_FACTOR))
        
        pygame.display.flip()
        clock.tick(30)

class Button:
    def __init__(self, center, radius, text=None, icon=None, color=None, hover_color=None):  # Fix: __init__ instead of _init_
        self.center = center
        self.radius = radius
        self.text = text
        self.icon = icon
        self.color = color or BUTTON_COLOR
        self.hover_color = hover_color or BUTTON_HOVER
    
    def is_clicked(self, pos):
        # Calculate distance between pos and center
        distance = math.sqrt((pos[0] - self.center[0])**2 + (pos[1] - self.center[1])**2)
        return distance <= self.radius
    
    def draw(self, surface, mouse_pos):
        color = self.hover_color if self.is_clicked(mouse_pos) else self.color
        pygame.draw.circle(surface, color, self.center, self.radius)

        if self.icon:
            icon_rect = self.icon.get_rect(center=self.center)
            surface.blit(self.icon, icon_rect)
        elif self.text:
            label = font.render(self.text, True, WHITE)
            surface.blit(label, label.get_rect(center=self.center))

def show_auth_screen():
    global user_type, username
    running = True

    google_btn = Button((WIDTH // 2, HEIGHT * 0.35), int(40 * SCALE_FACTOR), text="Google")
    facebook_btn = Button((WIDTH // 2, HEIGHT * 0.45), int(40 * SCALE_FACTOR), text="Facebook")
    guest_btn = Button((WIDTH // 2, HEIGHT * 0.6), int(40 * SCALE_FACTOR), text="Guest")

    while running:
        screen.fill((0, 30, 60))
        title = big_font.render("Login to Play", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.15))

        mouse_pos = pygame.mouse.get_pos()
        google_btn.draw(screen, mouse_pos)
        facebook_btn.draw(screen, mouse_pos)
        guest_btn.draw(screen, mouse_pos)
        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            elif event.type == pygame.MOUSEBUTTONDOWN:
                if google_btn.is_clicked(event.pos):
                    user_type = "google"
                    username = prompt_username("Google")
                    running = False
                elif facebook_btn.is_clicked(event.pos):
                    user_type = "facebook"
                    username = prompt_username("Facebook")
                    running = False
                elif guest_btn.is_clicked(event.pos):
                    user_type = "guest"
                    username = f"guest_{random.randint(1000, 9999)}"
                    running = False

def prompt_username(provider):
    input_active = True
    username = ""
    password = ""
    entering_password = False

    while input_active:
        screen.fill((20, 20, 40))
        prompt_title = font.render(f"{provider} Login", True, WHITE)
        user_prompt = font.render("Username:", True, WHITE)
        pass_prompt = font.render("Password:", True, WHITE)

        username_display = font.render(username + ("|" if not entering_password else ""), True, WHITE)
        password_display = font.render("*" * len(password) + ("|" if entering_password else ""), True, WHITE)

        # Position elements using relative scaling
        screen.blit(prompt_title, (WIDTH // 2 - prompt_title.get_width() // 2, HEIGHT * 0.15))
        screen.blit(user_prompt, (WIDTH * 0.25, HEIGHT * 0.35))
        screen.blit(username_display, (WIDTH * 0.5, HEIGHT * 0.35))
        screen.blit(pass_prompt, (WIDTH * 0.25, HEIGHT * 0.45))
        screen.blit(password_display, (WIDTH * 0.5, HEIGHT * 0.45))
        
        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_RETURN:
                    if entering_password:
                        if username and password:
                            input_active = False
                    else:
                        entering_password = True
                elif event.key == pygame.K_BACKSPACE:
                    if entering_password:
                        password = password[:-1]
                    else:
                        username = username[:-1]
                elif event.unicode.isprintable():
                    if entering_password:
                        password += event.unicode
                    else:
                        username += event.unicode
                elif event.key == pygame.K_ESCAPE:
                    return None  # Allow escaping the prompt

    return username  # Return only username, store password if needed for simulation


def get_high_score_filename():
    global username, user_type
    if user_type == "guest" or not username:
        return None
    safe_username = username.replace(" ", "_").lower()
    return f"snake_high_scores_{user_type}_{safe_username}.json"

class HighScoreManager:
    def __init__(self):
        self.high_scores = [0] * 5  # Store top 5 scores
        self.filename = get_high_score_filename()
        self.load_high_scores()

    def load_high_scores(self):
        """Load high scores from file if it exists"""
        if not self.filename:
            return
            
        try:
            if os.path.exists(self.filename):
                with open(self.filename, 'r') as f:
                    self.high_scores = json.load(f)
                    # Ensure we always have exactly 5 scores
                    while len(self.high_scores) < 5:
                        self.high_scores.append(0)
                    self.high_scores = self.high_scores[:5]
        except (json.JSONDecodeError, IOError):
            self.high_scores = [0] * 5

    def save_high_scores(self):
        """Save high scores to file"""
        if not self.filename:
            return
            
        try:
            with open(self.filename, 'w') as f:
                json.dump(self.high_scores, f)
        except IOError:
            print("Failed to save high scores")

    def add_score(self, score):
        """Add a new score to the high scores list if it qualifies"""
        if score > 0 and (len(self.high_scores) < 5 or score > min(self.high_scores)):
            self.high_scores.append(score)
            self.high_scores.sort(reverse=True)
            self.high_scores = self.high_scores[:5]  # Keep only top 5
            self.save_high_scores()
            return True
        return False

    def reset_scores(self):
        """Reset all high scores"""
        self.high_scores = [0] * 5
        self.save_high_scores()

    def get_high_scores(self):
        """Return the current high scores"""
        return self.high_scores

class GameState:
    def __init__(self):
        self.regular_rat_position = None
        self.big_rat_position = None
        self.next_direction = None
        self.reset()
    
    def get_random_position(self, size=1):
        """Returns a random grid-aligned position that doesn't collide with obstacles or snake"""
        while True:
            # Calculate maximum grid positions (subtract size to stay within bounds)
            max_x = (WIDTH // BLOCK_SIZE) - size
            max_y = (HEIGHT // BLOCK_SIZE) - size

            # Get random grid coordinates (integers)
            grid_x = random.randint(0, max_x)
            grid_y = random.randint(0, max_y)
            
            if grid_x == 0 or grid_y == 0 or grid_x == max_x or grid_y == max_y:
                continue  # Retry random position

            # Convert to pixel coordinates (guaranteed grid alignment)
            pos = (grid_x * BLOCK_SIZE, grid_y * BLOCK_SIZE)
        
            # Check collision with snake and obstacles
            position_occupied = False
            # Prevent rats from spawning on the border edges

            # Check each block in this position
            for dx in range(size):
                for dy in range(size):
                    check_pos = (pos[0] + dx * BLOCK_SIZE, 
                                pos[1] + dy * BLOCK_SIZE)

                    # Check against snake
                    if check_pos in self.snake:
                        position_occupied = True
                        break
                
                    # Check against obstacles
                    if check_pos in self.obstacles:
                        position_occupied = True
                        break
                        
                    # Check against existing rats
                    if (self.regular_rat_position and 
                        check_pos == self.regular_rat_position):
                        position_occupied = True
                        break
                    
                    if self.big_rat_position:
                        big_rat_x, big_rat_y = self.big_rat_position
                        if (big_rat_x <= check_pos[0] < big_rat_x + 2*BLOCK_SIZE and
                            big_rat_y <= check_pos[1] < big_rat_y + 2*BLOCK_SIZE):
                            position_occupied = True
                            break
                        
            if position_occupied:
                break
            
            if not position_occupied:
                return pos

    def check_regular_rat_collision(self, new_head):
        """Check if snake head collides with regular rat using grid-aligned positions"""
        if not self.regular_rat_position:
            return False
        
        # Convert to grid coordinates for reliable comparison
        head_grid = (new_head[0] // BLOCK_SIZE, 
                     new_head[1] // BLOCK_SIZE)
        rat_grid = (self.regular_rat_position[0] // BLOCK_SIZE,
                    self.regular_rat_position[1] // BLOCK_SIZE)
        
        return head_grid == rat_grid

    def check_big_rat_collision(self, new_head):
        """Check if snake head collides with big rat using precise rectangle collision"""
        if not self.big_rat_position or not self.big_rat_active:
            return False
        
        # Create rectangles for collision detection
        head_rect = pygame.Rect(new_head[0], new_head[1], 
                              BLOCK_SIZE, BLOCK_SIZE)
        big_rat_rect = pygame.Rect(self.big_rat_position[0],
                                  self.big_rat_position[1],
                                  BLOCK_SIZE*2, BLOCK_SIZE*2)
        
        return head_rect.colliderect(big_rat_rect) 

    def reset(self):
        # Start with head and tail only, aligned to grid
        # Position slightly left of center
        head_x = (WIDTH // 2) - (BLOCK_SIZE * 4)
        head_y = HEIGHT // 2
    
        # Ensure positions are aligned to grid
        head_x = (head_x // BLOCK_SIZE) * BLOCK_SIZE
        head_y = (head_y // BLOCK_SIZE) * BLOCK_SIZE

        tail_x = head_x - BLOCK_SIZE
        tail_y = head_y

        self.snake = [(head_x, head_y), (tail_x, tail_y)]  # Head at [0], tail at [1]
        self.direction = "RIGHT"
        self.next_direction = None
        self.base_word = "USMAN MAZHAR 101 - "
        self.current_word = self.base_word
        self.score = 0
        self.tongue_out = False
        self.tongue_timer = 0
        self.tongue_length = 0
        self.obstacles = set()
        self.setup_obstacles()  # Setup obstacles BEFORE getting rat position
        self.regular_rat_position = self.get_random_position()
        self.big_rat_position = None
        self.rats_eaten = 0
        self.big_rat_timer = 0.0
        self.big_rat_active = False
        self.set_parts(current_head_index, current_body_index, current_tail_index)
    
    def setup_obstacles(self):
        self.obstacles = set()
        if current_difficulty_index == 1:  # Medium
            num_blocks = random.randint(10, 15)
        elif current_difficulty_index == 2:  # Hard
            num_blocks = random.randint(30, 35)
        else:  # Easy
            num_blocks = 0
        
        for _ in range(num_blocks):
            pos = self.get_random_position()
            if pos not in self.snake:
                self.obstacles.add(pos)
    
    def update_word(self):
        repeat_count = (len(self.snake) // len(self.base_word)) + 1
        self.current_word = self.base_word * repeat_count
    
    def set_skin(self, index):
        global current_skin_index
        current_skin_index = int(index) % len(SNAKE_SKINS)
    
    def set_parts(self, head_idx, body_idx, tail_idx):
        """Set the snake parts using the specified indices"""
        # Safely get the parts lists
        heads = SNAKE_PARTS.get("heads", [{"name": "Classic", "shape": "circle"}])
        bodies = SNAKE_PARTS.get("bodies", [{"name": "Classic", "pattern": "solid"}])
        tails = SNAKE_PARTS.get("tails", [{"name": "Classic", "shape": "point"}])
        
        # Ensure indices are integers and within range
        head_idx = int(head_idx) % len(heads)
        body_idx = int(body_idx) % len(bodies)
        tail_idx = int(tail_idx) % len(tails)
        
        # Only set shapes, not colors
        self.head_part = {"name": heads[head_idx]["name"],
                         "shape": heads[head_idx]["shape"]}
        self.body_part = {"name": bodies[body_idx]["name"],
                         "pattern": bodies[body_idx]["pattern"]}
        self.tail_part = {"name": tails[tail_idx]["name"],
                         "shape": tails[tail_idx]["shape"]}
    
    def head_in_grid_center(self):
        head_x, head_y = self.snake[0]
        return (head_x % BLOCK_SIZE == 0) and (head_y % BLOCK_SIZE == 0)

class MultiplayerGameState:
    """Two‑player classic Snake with split‑arena rats (P1‑left, P2‑right)."""

    RAT_SHIFT = int(BLOCK_SIZE * 0.365)   # aligns centre wall & rat grid

    # ───────────────────────────────────────────────────────────── #
    #  INITIALISATION                                              #
    # ───────────────────────────────────────────────────────────── #
    def __init__(self, p1_name, p1_skin, p2_name, p2_skin, speed_level):
        self.speed_level = speed_level
        self.move_delay  = speed_levels[speed_level] / 1000.0   # ms → s
        self.move_timer  = 0.0

        self.player1 = self._new_player_dict(p1_name, p1_skin)
        self.player2 = self._new_player_dict(p2_name, p2_skin)

        # column of blocks that form the centre wall (handy for drawing)
        half    = BLOCK_SIZE // 2
        left_x  = (WIDTH // 2 - half) + self.RAT_SHIFT
        right_x = (WIDTH // 2 + half) + self.RAT_SHIFT
        self.center_border = (
            [(left_x,  y) for y in range(0, HEIGHT, BLOCK_SIZE)] +
            [(right_x, y) for y in range(0, HEIGHT, BLOCK_SIZE)]
        )

        self.reset()

    # ───────────────────────────────────────────────────────────── #
    #  RAT‑PLACEMENT HELPERS                                       #
    # ───────────────────────────────────────────────────────────── #
    def _is_safe_rat_square(self, x: int, y: int) -> bool:
        """True if (x,y) is not a wall, centre wall, or snake body square."""
        # outer frame (one‑block thick)
        if (x < BLOCK_SIZE or x >= WIDTH  - BLOCK_SIZE or
            y < BLOCK_SIZE or y >= HEIGHT - BLOCK_SIZE):
            return False
        # centre vertical wall
        cl = WIDTH // 2 - BLOCK_SIZE + self.RAT_SHIFT
        cr = WIDTH // 2 + BLOCK_SIZE + self.RAT_SHIFT
        if cl <= x < cr:
            return False
        # any snake segment
        if (x, y) in self.player1["snake"] or (x, y) in self.player2["snake"]:
            return False
        return True

    def _rand_pos(self, side: str) -> tuple[int, int]:
        """Return a legal 1 × 1 rat square on the requested side (full height)."""
        while True:
            if side == "right":
                x_min = WIDTH // 2 + BLOCK_SIZE + self.RAT_SHIFT
                x_max = WIDTH - BLOCK_SIZE
            else:                       # left
                x_min = BLOCK_SIZE
                x_max = WIDTH // 2 - BLOCK_SIZE
            y_min = BLOCK_SIZE
            y_max = HEIGHT - BLOCK_SIZE

            x = random.randrange(x_min, x_max, BLOCK_SIZE)
            y = random.randrange(y_min, y_max, BLOCK_SIZE)
            if self._is_safe_rat_square(x, y):
                return x, y

    def _rand_big_pos(self, side: str) -> tuple[int, int]:
        """Return top‑left block for a 2 × 2 big rat on the requested side."""
        while True:
            if side == "right":
                x_min = WIDTH // 2 + BLOCK_SIZE + self.RAT_SHIFT
                x_max = WIDTH - BLOCK_SIZE * 2
            else:                       # left
                x_min = BLOCK_SIZE
                x_max = WIDTH // 2 - BLOCK_SIZE * 2
            y_min = BLOCK_SIZE
            y_max = HEIGHT - BLOCK_SIZE * 2

            x = random.randrange(x_min, x_max, BLOCK_SIZE)
            y = random.randrange(y_min, y_max, BLOCK_SIZE)

            # make sure all four blocks are safe
            if all(self._is_safe_rat_square(x + dx, y + dy)
                   for dx in (0, BLOCK_SIZE)
                   for dy in (0, BLOCK_SIZE)):
                return x, y

    # simple wrapper used by reset()
    def _spawn_rat(self, side: str) -> tuple[int, int]:
        return self._rand_pos("left" if side.lower().startswith("l") else "right")

    # ───────────────────────────────────────────────────────────── #
    #  PLAYER DICT TEMPLATE                                        #
    # ───────────────────────────────────────────────────────────── #
    def _new_player_dict(self, name, skin):
        return dict(
            name=name, skin=skin, score=0, snake=[],
            direction="RIGHT", next_direction=None,
            regular_rat=None, big_rat=None, rats_eaten=0,
            big_timer=0.0, big_active=False, alive=True
        )

    # ───────────────────────────────────────────────────────────── #
    #  UPDATE LOOP                                                 #
    # ───────────────────────────────────────────────────────────── #
    def update(self, dt):
        """Advance timers, move snakes, return (p1_result, p2_result)."""
        # big‑rat countdown (every frame)
        for p in (self.player1, self.player2):
            if p["big_active"]:
                p["big_timer"] -= dt
                if p["big_timer"] <= 0:
                    p["big_active"] = False
                    p["big_rat"]    = None

        # snake movement cadence
        self.move_timer += dt
        if self.move_timer < self.move_delay:
            return None, None
        self.move_timer = 0.0

        r1 = self._move_snake(self.player1) if self.player1["alive"] else None
        r2 = self._move_snake(self.player2) if self.player2["alive"] else None

        # mark deaths
        if r1 == "DEAD": self.player1["alive"] = False
        if r2 == "DEAD": self.player2["alive"] = False

        # win / draw logic
        if not self.player1["alive"] and not self.player2["alive"]:
            return "DEAD", "DEAD"
        if self.player1["alive"] and not self.player2["alive"]:
            if self.player1["score"] > self.player2["score"]:
                return "WIN", "DEAD"
        if self.player2["alive"] and not self.player1["alive"]:
            if self.player2["score"] > self.player1["score"]:
                return "DEAD", "WIN"
        return r1, r2

    # ───────────────────────────────────────────────────────────── #
    #  RESET                                                       #
    # ───────────────────────────────────────────────────────────── #
    def reset(self):
        """Start a new round with fresh snakes and rats."""
        head_y = (HEIGHT // 2) // BLOCK_SIZE * BLOCK_SIZE

        # player 1 (left)
        head_x1 = (WIDTH // 4) // BLOCK_SIZE * BLOCK_SIZE
        self.player1["snake"] = [(head_x1, head_y),
                                 (head_x1 - BLOCK_SIZE, head_y)]
        self.player1.update(dict(
            direction="RIGHT", next_direction=None, score=0,
            alive=True, rats_eaten=0, big_active=False,
            big_rat=None, regular_rat=self._spawn_rat("left")
        ))

        # player 2 (right)
        head_x2 = (3 * WIDTH // 4) // BLOCK_SIZE * BLOCK_SIZE
        self.player2["snake"] = [(head_x2, head_y),
                                 (head_x2 + BLOCK_SIZE, head_y)]
        self.player2.update(dict(
            direction="LEFT", next_direction=None, score=0,
            alive=True, rats_eaten=0, big_active=False,
            big_rat=None, regular_rat=self._spawn_rat("right")
        ))

    # ───────────────────────────────────────────────────────────── #
    #  SNAKE MOVEMENT / EATING                                     #
    # ───────────────────────────────────────────────────────────── #
    def _move_snake(self, p):
        if p["next_direction"]:
            p["direction"] = p["next_direction"]
            p["next_direction"] = None

        x, y = p["snake"][0]
        if p["direction"] == "UP":    y -= BLOCK_SIZE
        if p["direction"] == "DOWN":  y += BLOCK_SIZE
        if p["direction"] == "LEFT":  x -= BLOCK_SIZE
        if p["direction"] == "RIGHT": x += BLOCK_SIZE
        head = (x, y)

        if self._collision(p, head):
            return "DEAD"

        p["snake"].insert(0, head)

        # food checks
        if self._eat_regular_rat(p, head):
            p["score"] += 10
            p["rats_eaten"] += 1
            self._spawn_regular_rat(
                p, side=("left" if p is self.player1 else "right"))
            if p["rats_eaten"] >= 5:
                p["rats_eaten"] = 0
                self._spawn_big_rat(p)

        elif self._eat_big_rat(p, head):
            pts = max(10, min(50, int(p["big_timer"] * 10)))
            p["score"] += pts
            p["big_active"] = False
            p["big_rat"] = None

        else:
            p["snake"].pop()

        return None

    # ───────────────────────────────────────────────────────────── #
    #  COLLISION DETECTION                                         #
    # ───────────────────────────────────────────────────────────── #
    def _collision(self, p, head):
        # arena frame
        if (head[0] < BLOCK_SIZE or head[0] >= WIDTH - BLOCK_SIZE or
            head[1] < BLOCK_SIZE or head[1] >= HEIGHT - BLOCK_SIZE):
            return True
        # centre wall
        le = WIDTH // 2 - BLOCK_SIZE + int(BLOCK_SIZE * 0.3)
        re = WIDTH // 2 + BLOCK_SIZE + int(BLOCK_SIZE * 0.3)
        if le <= head[0] < re:
            return True
        # self
        if head in p["snake"][1:]:
            return True
        # other snake
        other = self.player2 if p is self.player1 else self.player1
        if head in other["snake"]:
            return True
        # other player’s big rat
        if other["big_active"]:
            br = other["big_rat"]
            if pygame.Rect(br[0], br[1], BLOCK_SIZE * 2,
                           BLOCK_SIZE * 2).collidepoint(head):
                return True
        return False

    # ───────────────────────────────────────────────────────────── #
    #  RAT SPAWN & EAT                                             #
    # ───────────────────────────────────────────────────────────── #
    def _spawn_regular_rat(self, p, *, side):
        p["regular_rat"] = self._rand_pos(side)

    def _spawn_big_rat(self, p):
        side = "left" if p is self.player1 else "right"
        p["big_rat"]    = self._rand_big_pos(side)
        p["big_timer"]  = 5.0
        p["big_active"] = True

    def _eat_regular_rat(self, p, head):
        rat = p["regular_rat"]
        return rat and (head[0] // BLOCK_SIZE, head[1] // BLOCK_SIZE) == \
                       (rat[0] // BLOCK_SIZE,  rat[1] // BLOCK_SIZE)

    def _eat_big_rat(self, p, head):
        if not p["big_active"]:
            return False
        br = p["big_rat"]
        return pygame.Rect(*head, BLOCK_SIZE, BLOCK_SIZE).colliderect(
               pygame.Rect(br[0], br[1], BLOCK_SIZE * 2, BLOCK_SIZE * 2))

import random
import pygame

class MultiplayerWarGameState:
    """
    War‑mode game state.

    • One shared small rat (+10 pts) that can appear anywhere except the
      one‑block outer border and never on a snake.
    • After a combined total of 5 small rats are eaten by either or both
      snakes, a single shared big rat (2×2 cells) spawns.
    • The big rat lasts for exactly 5 seconds.  If neither snake eats it
      in that time it disappears automatically.
    • The big rat’s value starts at 50 pts and decays by 1 pt every 0.1 s
      (min 10 pts).
    • Round ends immediately on any collision; higher‑score logic is
      handled by your outer game‑loop.
    """

    # ──────────────────────────────────────────────────────────────── #
    #  Initialisation                                                 #
    # ──────────────────────────────────────────────────────────────── #
    def __init__(self, p1_name, p1_skin, p2_name, p2_skin, speed_level):
        # Timing
        self.speed_level  = speed_level
        self.move_delay   = speed_levels[speed_level] / 1000.0  # ms→s
        self.move_timer   = 0.0

        # Players
        self.player1 = self._new_player_dict(p1_name, p1_skin)
        self.player2 = self._new_player_dict(p2_name, p2_skin)

        # Shared food
        self.shared_regular_rat = None
        self.shared_big_rat     = None
        self.shared_big_active  = False
        self.shared_big_timer   = 0.0  # counts down from 5 s

        # Total small rats eaten (combined)
        self.small_rats_eaten_total = 0

        # War‑mode has no centre wall; keep list for future features
        self.center_border = []

        self.reset()

    # ──────────────────────────────────────────────────────────────── #
    #  Helper creators                                                #
    # ──────────────────────────────────────────────────────────────── #
    def _new_player_dict(self, name, skin):
        return {
            "name":           name,
            "skin":           skin,
            "score":          0,
            "snake":          [],
            "direction":      "RIGHT",
            "next_direction": None,
            "alive":          True,
        }

    @staticmethod
    def _is_opposite(d1, d2):
        return (d1, d2) in {("UP", "DOWN"), ("DOWN", "UP"),
                            ("LEFT", "RIGHT"), ("RIGHT", "LEFT")}

    # ──────────────────────────────────────────────────────────────── #
    #  Public interface                                               #
    # ──────────────────────────────────────────────────────────────── #
    def reset(self):
        """Reset snakes, scores and food for a fresh round."""

        # Player 1 – left side, facing right
        head_x1 = (WIDTH // 4) // BLOCK_SIZE * BLOCK_SIZE
        head_y  = (HEIGHT // 2) // BLOCK_SIZE * BLOCK_SIZE
        self.player1["snake"] = [(head_x1, head_y),
                                 (head_x1 - BLOCK_SIZE, head_y)]
        self.player1.update({"direction": "RIGHT",
                             "next_direction": None,
                             "score": 0,
                             "alive": True})

        # Player 2 – right side, facing left
        head_x2 = (3 * WIDTH // 4) // BLOCK_SIZE * BLOCK_SIZE
        self.player2["snake"] = [(head_x2, head_y),
                                 (head_x2 + BLOCK_SIZE, head_y)]
        self.player2.update({"direction": "LEFT",
                             "next_direction": None,
                             "score": 0,
                             "alive": True})

        # Initialise food
        self.shared_regular_rat = self._rand_pos()
        self.shared_big_rat     = None
        self.shared_big_active  = False
        self.shared_big_timer   = 0.0
        self.small_rats_eaten_total = 0

        self.move_timer = 0.0

    def update(self, dt):
        """Advance the simulation by *dt* seconds.

        Returns (result_p1, result_p2):
            None   → still alive
            "DEAD" → collided this step
            "WIN"  → other snake died this step
        """
        # ---------------------------------------------------------------
        # 1) Big‑rat countdown runs EVERY frame (real‑time seconds)
        # ---------------------------------------------------------------
        if self.shared_big_active:
            self.shared_big_timer -= dt
            if self.shared_big_timer <= 0:
                self.shared_big_active = False
                self.shared_big_rat    = None

        # ---------------------------------------------------------------
        # 2) Snake movement throttled by move_delay
        # ---------------------------------------------------------------
        self.move_timer += dt
        if self.move_timer < self.move_delay:
            return None, None          # ← early exit keeps countdown accurate
        self.move_timer = 0.0

        # Move snakes if alive
        r1 = self._move_snake(self.player1) if self.player1["alive"] else None
        r2 = self._move_snake(self.player2) if self.player2["alive"] else None

        # ---------------------------------------------------------------
        # 3) End round immediately on any death
        # ---------------------------------------------------------------
        if r1 == "DEAD" or r2 == "DEAD":
            if r1 == "DEAD": self.player1["alive"] = False
            if r2 == "DEAD": self.player2["alive"] = False

            if self.player1["alive"] and not self.player2["alive"]: return "WIN", "DEAD"
            if self.player2["alive"] and not self.player1["alive"]: return "DEAD", "WIN"
            return "DEAD", "DEAD"  # simultaneous crash

        return r1, r2

    # ──────────────────────────────────────────────────────────────── #
    #  Internal helpers                                               #
    # ──────────────────────────────────────────────────────────────── #
    def _move_snake(self, p):
        # Apply queued turn (ignore 180° reverse)
        if p["next_direction"] and not self._is_opposite(p["direction"],
                                                         p["next_direction"]):
            p["direction"] = p["next_direction"]
        p["next_direction"] = None

        # Compute new head
        x, y = p["snake"][0]
        if p["direction"] == "UP":    y -= BLOCK_SIZE
        if p["direction"] == "DOWN":  y += BLOCK_SIZE
        if p["direction"] == "LEFT":  x -= BLOCK_SIZE
        if p["direction"] == "RIGHT": x += BLOCK_SIZE
        head = (x, y)

        # Collision?
        if self._collision(p, head):
            return "DEAD"

        # Add new head
        p["snake"].insert(0, head)

        # Eat small rat?
        if self._eat_regular_rat(head):
            p["score"] += 10
            self.small_rats_eaten_total += 1
            self.shared_regular_rat = self._rand_pos()
            if self.small_rats_eaten_total >= 5:
                self.small_rats_eaten_total = 0
                self._spawn_big_rat()

        # Eat big rat?
        elif self._eat_big_rat(head):
            points = max(10, min(50, int(self.shared_big_timer * 10)))
            p["score"] += points
            self.shared_big_active = False
            self.shared_big_rat    = None

        else:
            # No food → drop tail
            p["snake"].pop()

        return None

    # --------------------- collision helpers ------------------------ #
    def _collision(self, p, head):
        # Border
        if (head[0] < BLOCK_SIZE or head[0] >= WIDTH  - BLOCK_SIZE or
            head[1] < BLOCK_SIZE or head[1] >= HEIGHT - BLOCK_SIZE):
            return True
        # Self
        if head in p["snake"][1:]:
            return True
        # Other snake body
        other = self.player2 if p is self.player1 else self.player1
        if head in other["snake"][1:]:
            return True
        # Head‑on
        return head == other["snake"][0]

    # ---------------------- food helpers ---------------------------- #
    def _eat_regular_rat(self, head):
        if not self.shared_regular_rat:
            return False
        return (head[0] // BLOCK_SIZE == self.shared_regular_rat[0] // BLOCK_SIZE and
                head[1] // BLOCK_SIZE == self.shared_regular_rat[1] // BLOCK_SIZE)

    def _eat_big_rat(self, head):
        if not self.shared_big_active:
            return False
        head_rect = pygame.Rect(*head, BLOCK_SIZE, BLOCK_SIZE)
        big_rect  = pygame.Rect(*self.shared_big_rat,
                                BLOCK_SIZE * 2, BLOCK_SIZE * 2)
        return head_rect.colliderect(big_rect)

    # ---------------------- spawn helpers --------------------------- #
    def _spawn_big_rat(self):
        self.shared_big_rat    = self._rand_pos()
        self.shared_big_active = True
        self.shared_big_timer  = 5.0  # 5 seconds lifetime

    def _rand_pos(self):
        """Return a random BLOCK‑aligned position:
           • Not on the one‑block border
           • Not inside either snake
        """
        while True:
            x = random.randint(2 * BLOCK_SIZE, WIDTH  - 3 * BLOCK_SIZE)
            y = random.randint(2 * BLOCK_SIZE, HEIGHT - 3 * BLOCK_SIZE)
            pos = (x - x % BLOCK_SIZE, y - y % BLOCK_SIZE)
            if pos in self.player1["snake"] or pos in self.player2["snake"]:
                continue
            return pos



def show_high_scores(high_score_manager):
    global video_playing
    # Pause video playback
    video_playing = False
    
    """Display the high scores screen"""
    screen.fill((0, 0, 50))
    title = big_font.render("High Scores", True, WHITE)
    screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.1))
    
    scores = high_score_manager.get_high_scores()
    for i, score in enumerate(scores):
        score_text = font.render(f"{i+1}. {score}", True, WHITE)
        screen.blit(score_text, (WIDTH // 2 - score_text.get_width() // 2, 
                                HEIGHT * 0.2 + i * 50 * SCALE_FACTOR))
    
    # Create buttons
    back_button = Button((WIDTH // 2 - int(100 * SCALE_FACTOR), HEIGHT - int(100 * SCALE_FACTOR)), 
                         int(30 * SCALE_FACTOR), text="Back")
    reset_button = Button((WIDTH // 2 + int(100 * SCALE_FACTOR), HEIGHT - int(100 * SCALE_FACTOR)), 
                         int(30 * SCALE_FACTOR), text="Reset")
    
    mouse_pos = pygame.mouse.get_pos()
    back_button.draw(screen, mouse_pos)
    reset_button.draw(screen, mouse_pos)
    
    pygame.display.flip()
    
    waiting = True
    while waiting:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                if back_button.is_clicked(event.pos):
                    # Resume video playback
                    video_playing = True
                    waiting = False
                elif reset_button.is_clicked(event.pos):
                    high_score_manager.reset_scores()
                    # Resume video playback
                    video_playing = True
                    waiting = False
                    return True  # Return True to indicate we need to refresh
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                # Resume video playback
                video_playing = True
                waiting = False
    
    return False  # Return False if we just went back without resetting

def draw_body_part(surface, pos, size, body_data, primary_color, secondary_color):
    """Draw a snake body segment with pattern using provided colors"""
    center_x, center_y = pos[0] + size // 2, pos[1] + size // 2
    
    # Draw based on pattern type
    if body_data["pattern"] == "solid":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
    
    elif body_data["pattern"] == "stripes":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        for i in range(3):  # Draw 3 stripes
            angle = i * 60
            start_x = center_x + (size//2) * math.cos(math.radians(angle))
            start_y = center_y + (size//2) * math.sin(math.radians(angle))
            end_x = center_x + (size//2) * math.cos(math.radians(angle + 180))
            end_y = center_y + (size//2) * math.sin(math.radians(angle + 180))
            pygame.draw.line(surface, secondary_color, (start_x, start_y), (end_x, end_y), 3)
    
    elif body_data["pattern"] == "spots":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        for _ in range(5):  # Draw 5 spots
            spot_x = center_x + random.randint(-size//3, size//3)
            spot_y = center_y + random.randint(-size//3, size//3)
            pygame.draw.circle(surface, secondary_color, (spot_x, spot_y), size//6)
    
    elif body_data["pattern"] == "gradient":
        # Create gradient effect with concentric circles
        steps = 5
        for i in range(steps):
            ratio = i / steps
            r = int(primary_color[0] * (1 - ratio) + secondary_color[0] * ratio)
            g = int(primary_color[1] * (1 - ratio) + secondary_color[1] * ratio)
            b = int(primary_color[2] * (1 - ratio) + secondary_color[2] * ratio)
            radius = size//2 - (i * size//(steps*2))
            pygame.draw.circle(surface, (r, g, b), (center_x, center_y), radius)
    
    elif body_data["pattern"] == "metallic":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        # Add highlights
        highlight_color = (min(255, primary_color[0] + 50), 
                          min(255, primary_color[1] + 50), 
                          min(255, primary_color[2] + 50))
        pygame.draw.ellipse(surface, highlight_color, 
                          (center_x - size//3, center_y - size//3, size//1.5, size//3))
    
    elif body_data["pattern"] == "camo":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        # Draw random camo blobs
        for _ in range(5):
            blob_x = center_x + random.randint(-size//3, size//3)
            blob_y = center_y + random.randint(-size//3, size//3)
            blob_size = random.randint(size//8, size//4)
            pygame.draw.ellipse(surface, secondary_color, 
                              (blob_x - blob_size//2, blob_y - blob_size//2, blob_size, blob_size))
    
    elif body_data["pattern"] == "glass":
        # Create a transparent surface
        s = pygame.Surface((size, size), pygame.SRCALPHA)
        # Draw glass circle
        glass_color = (primary_color[0], primary_color[1], primary_color[2], 180)  # Semi-transparent
        pygame.draw.circle(s, glass_color, (size//2, size//2), size//2)
        # Add highlights
        highlight_color = (255, 255, 255, 120)
        pygame.draw.ellipse(s, highlight_color, (size//4, size//4, size//2, size//4))
        surface.blit(s, pos)
    
    elif body_data["pattern"] == "scales":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        # Draw scales
        for i in range(8):  # Draw 8 scales
            angle = i * 45
            scale_x = center_x + (size//3) * math.cos(math.radians(angle))
            scale_y = center_y + (size//3) * math.sin(math.radians(angle))
            scale_width = size//3
            scale_height = size//4
            pygame.draw.ellipse(surface, secondary_color, 
                              (scale_x - scale_width//2, scale_y - scale_height//2, 
                               scale_width, scale_height))
    
    elif body_data["pattern"] == "bands":
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        # Draw 3 horizontal bands
        band_height = size // 6
        pygame.draw.rect(surface, secondary_color, (pos[0], center_y - band_height*1.5, size, band_height))
        pygame.draw.rect(surface, secondary_color, (pos[0], center_y - band_height*0.5, size, band_height))
        pygame.draw.rect(surface, secondary_color, (pos[0], center_y + band_height*0.5, size, band_height))
    
    elif body_data["pattern"] == "glow":
        # Draw base circle
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)
        # Create glow effect with concentric semi-transparent circles
        s = pygame.Surface((size*2, size*2), pygame.SRCALPHA)
        for i in range(3):
            alpha = 150 - i * 50
            glow_color = (primary_color[0], primary_color[1], primary_color[2], alpha)
            radius = size//2 + i*5
            pygame.draw.circle(s, glow_color, (size, size), radius)
        surface.blit(s, (pos[0] - size//2, pos[1] - size//2))
    
    else:  # Default to solid if pattern not recognized
        pygame.draw.circle(surface, primary_color, (center_x, center_y), size // 2)

def show_skin_selection():
    global current_skin_index, applied_skin_index

    apply_text = "Applied" if current_skin_index == applied_skin_index else "Apply"

    while True:
        screen.fill((0, 30, 60))
        title = big_font.render("Select Snake Skin", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.08))

        current_skin = SNAKE_SKINS[current_skin_index]
        skin_name = font.render(current_skin["name"], True, WHITE)
        screen.blit(skin_name, (WIDTH // 2 - skin_name.get_width() // 2, HEIGHT * 0.15))

        preview_size = int(200 * SCALE_FACTOR)
        preview_start_x = WIDTH // 2 - preview_size // 2
        preview_y = HEIGHT * 0.2
        block_size = int(30 * SCALE_FACTOR)

        head_part = SNAKE_PARTS["heads"][current_head_index]
        body_part = SNAKE_PARTS["bodies"][current_body_index]
        tail_part = SNAKE_PARTS["tails"][current_tail_index]

        if "color" in current_skin:
            primary_color = current_skin["color"]
            secondary_color = tuple(max(0, c - 40) for c in primary_color)
        else:
            primary_color = current_skin["colors"][0]
            secondary_color = tuple(max(0, c - 40) for c in primary_color)

        for i in range(5):
            x = preview_start_x + i * block_size
            y = preview_y
            segment_color = current_skin["colors"][i % len(current_skin["colors"])] if "colors" in current_skin else primary_color
            body_primary = segment_color
            body_secondary = tuple(max(0, c - 40) for c in body_primary)

            if i == 0:
                draw_head_part(screen, (x, y), block_size, head_part, "RIGHT", segment_color)
            elif i == 4:
                draw_tail_part(screen, (x, y), block_size, tail_part, "RIGHT", segment_color)
            else:
                draw_body_part(screen, (x, y), block_size, body_part, body_primary, body_secondary)

        apply_button = Button((WIDTH // 2, HEIGHT - int(50 * SCALE_FACTOR)), int(40 * SCALE_FACTOR), text=apply_text)
        back_button = Button((int(50 * SCALE_FACTOR), int(50 * SCALE_FACTOR)), int(30 * SCALE_FACTOR), text="X")
        prev_button = Button((int(100 * SCALE_FACTOR), HEIGHT - int(100 * SCALE_FACTOR)), int(40 * SCALE_FACTOR), text="<")
        next_button = Button((WIDTH - int(100 * SCALE_FACTOR), HEIGHT - int(100 * SCALE_FACTOR)), int(40 * SCALE_FACTOR), text=">")

        mouse_pos = pygame.mouse.get_pos()
        for btn in [apply_button, back_button, prev_button, next_button]:
            btn.draw(screen, mouse_pos)

        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                return "back"
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if apply_button.is_clicked(pos):
                    applied_skin_index = current_skin_index
                    apply_text = "Applied"
                elif back_button.is_clicked(pos):
                    return "back"
                elif prev_button.is_clicked(pos):
                    current_skin_index = (current_skin_index - 1) % len(SNAKE_SKINS)
                    apply_text = "Applied" if current_skin_index == applied_skin_index else "Apply"
                elif next_button.is_clicked(pos):
                    current_skin_index = (current_skin_index + 1) % len(SNAKE_SKINS)
                    apply_text = "Applied" if current_skin_index == applied_skin_index else "Apply"


def draw_grid():
    GRID_WIDTH = WIDTH // BLOCK_SIZE
    GRID_HEIGHT = HEIGHT // BLOCK_SIZE
    grid_color = (50, 50, 50)  # Dark gray grid lines
    
    # Draw vertical lines (skip the last one)
    for x in range(0, WIDTH - BLOCK_SIZE, BLOCK_SIZE):  # Changed to WIDTH - BLOCK_SIZE
        pygame.draw.line(screen, grid_color, (x, 0), (x, HEIGHT), 1)
    
    # Draw horizontal lines
    for y in range(0, HEIGHT, BLOCK_SIZE):
        pygame.draw.line(screen, grid_color, (0, y), (WIDTH, y), 1)

def get_video_frame():
    global video_playing
    if not video_playing:
        return None
    
    ret, frame = cap.read()
    if not ret:
        cap.set(cv2.CAP_PROP_POS_FRAMES, 0)  # Loop the video
        ret, frame = cap.read()
    frame = cv2.resize(frame, (WIDTH, HEIGHT))
    frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    return pygame.surfarray.make_surface(np.rot90(frame))

def show_intro_logo():
    if not cap_intro or not cap_intro.isOpened():
        print("Intro video not available")
        return
    
    # Speed control for both audio and video (1.0 = normal, 2.0 = 2x faster, 0.5 = 2x slower)
    playback_speed = 2.0  # Change this to adjust both audio and video speed
    
    # Video setup
    fps = cap_intro.get(cv2.CAP_PROP_FPS)
    if fps <= 0:
        fps = 30  # Fallback FPS
    
    # Calculate frame skipping for video
    skip_frames = int(playback_speed - 1) if playback_speed > 1 else 0
    frame_delay = int(1000 / (fps * playback_speed))  # Adjusted frame delay
    
    # Audio setup
    try:
        # Create temp slowed/sped-up audio file
        import subprocess, os
        temp_audio = "temp_speed_adjusted.wav"
        
        if playback_speed != 1.0:
            if playback_speed < 0.5:
                # Need to chain atempo filters for very slow speeds
                atempo_chain = f'atempo={0.5},atempo={playback_speed/0.5}'
            elif playback_speed > 2.0:
                # Need to chain atempo filters for very fast speeds
                atempo_chain = f'atempo={2.0},atempo={playback_speed/2.0}'
            else:
                atempo_chain = f'atempo={playback_speed}'
            
            subprocess.run([
                'ffmpeg', '-i', 'intro_music.wav',
                '-filter:a', atempo_chain,
                '-y', temp_audio
            ], check=True, stderr=subprocess.DEVNULL)
        else:
            temp_audio = "intro_music.wav"  # Use original if speed is 1.0
    except Exception as e:
        print(f"Audio speed adjustment failed: {e}")
        temp_audio = "intro_music.wav"  # Fallback to original

    start_time = pygame.time.get_ticks()
    original_duration = 24300  # Original duration in ms
    adjusted_duration = original_duration / playback_speed  # Adjusted duration

    try:
        pygame.mixer.music.load(temp_audio)
        pygame.mixer.music.play()
    except Exception as e:
        print(f"Could not play intro music: {e}")

    while pygame.time.get_ticks() - start_time < adjusted_duration:
        # Skip frames for faster playback
        for _ in range(skip_frames):
            if not cap_intro.grab():  # Skip without decoding
                cap_intro.set(cv2.CAP_PROP_POS_FRAMES, 0)
                break
                
        ret, frame = cap_intro.read()
        if not ret:
            cap_intro.set(cv2.CAP_PROP_POS_FRAMES, 0)
            continue
            
        frame = cv2.resize(frame, (WIDTH, HEIGHT))
        frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        frame_surface = pygame.surfarray.make_surface(np.rot90(frame))
        screen.blit(frame_surface, (0, 0))
        pygame.display.flip()
        
        # Process events
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN or event.type == pygame.MOUSEBUTTONDOWN:
                cap_intro.set(cv2.CAP_PROP_POS_FRAMES, 0)
                pygame.mixer.music.stop()
                if temp_audio != "intro_music.wav":
                    try: os.remove(temp_audio)
                    except: pass
                return

        pygame.time.delay(frame_delay)

    cap_intro.set(cv2.CAP_PROP_POS_FRAMES, 0)
    pygame.mixer.music.stop()
    if temp_audio != "intro_music.wav":
        try: os.remove(temp_audio)
        except: pass

def draw_border():
    # Use BLOCK_SIZE for left, bottom, and top borders
    thick_left_top_bottom = BLOCK_SIZE  # 1 block thickness
    thick_right = BLOCK_SIZE // 2  # Half block thickness for right border
    color = (200, 0, 0)  # Red border
    
    # Draw borders
    pygame.draw.rect(screen, color, (0, 0, WIDTH, thick_left_top_bottom))  # Top border (1 block)
    pygame.draw.rect(screen, color, (0, HEIGHT - thick_left_top_bottom, WIDTH, thick_left_top_bottom))  # Bottom border (1 block)
    pygame.draw.rect(screen, color, (0, 0, thick_left_top_bottom, HEIGHT))  # Left border (1 block)
    pygame.draw.rect(screen, color, (WIDTH - thick_right, 0, thick_right, HEIGHT))  # Right border (half block)


def show_authentication_screen(screen, WIDTH, HEIGHT):
    running = True
    facebook_btn = Button((WIDTH // 2, HEIGHT * 0.35), int(40 * SCALE_FACTOR), text="Facebook")
    guest_btn = Button((WIDTH // 2, HEIGHT * 0.5), int(40 * SCALE_FACTOR), text="Guest")

    user_type = None
    username = None

    while running:
        screen.fill((0, 30, 60))
        title = big_font.render("Login to Play", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.15))

        mouse_pos = pygame.mouse.get_pos()
        facebook_btn.draw(screen, mouse_pos)
        guest_btn.draw(screen, mouse_pos)
        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            elif event.type == pygame.MOUSEBUTTONDOWN:
                if facebook_btn.is_clicked(event.pos):
                    result = facebook_auth_screen(screen, WIDTH, HEIGHT)
                    if result is not None:  # Only proceed if not going back
                        user_type = "facebook"
                        username = result
                        running = False
                elif guest_btn.is_clicked(event.pos):
                    user_type = "guest"
                    username = f"guest_{random.randint(1000, 9999)}"
                    running = False
            elif event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                running = False
                return None, None

    return user_type, username

def get_random_position():
    max_x = WIDTH // BLOCK_SIZE - 2
    max_y = HEIGHT // BLOCK_SIZE - 2
    while True:
        grid_x = random.randint(1, max_x)
        grid_y = random.randint(1, max_y)
        return (grid_x * BLOCK_SIZE, grid_y * BLOCK_SIZE)

def draw_menu(in_settings=False, profile=None):
    frame = get_video_frame()
    if frame:
        screen.blit(frame, (0, 0))  # Draw video background if available

    # Title text
    title = big_font.render("Snake Game", True, WHITE)
    subtitle = font.render("By Toon Squad (TS)", True, WHITE)
    screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.1))
    screen.blit(subtitle, (WIDTH // 2 - subtitle.get_width() // 2, HEIGHT * 0.18))

    mouse_pos = pygame.mouse.get_pos()

    # Draw profile button in top-left corner
    if profile:
        avatar_size = int(35 * SCALE_FACTOR)  # Reduced from 50 to 35
        avatar_surface = profile.get_avatar_surface(avatar_size)
        profile_rect = pygame.Rect(20, 20, avatar_size + 10 + font.size(f"{profile.username} (Lv. {profile.data['level']})")[0], avatar_size)
        
        # Draw button background
        color = PROFILE_HOVER if profile_rect.collidepoint(mouse_pos) else PROFILE_COLOR
        pygame.draw.rect(screen, color, profile_rect, border_radius=avatar_size//2)
        
        # Draw avatar
        screen.blit(avatar_surface, (25, 25))
        
        # Draw username and level
        profile_text = font.render(f"{profile.username} (Lv. {profile.data['level']})", True, WHITE)
        screen.blit(profile_text, (25 + avatar_size + 10, 25 + (avatar_size - profile_text.get_height()) // 2))

    # In the draw_menu function where the settings buttons are created:
    if in_settings:
        # Settings menu buttons
        buttons = [
            Button((WIDTH // 2, HEIGHT * 0.35), int(30 * SCALE_FACTOR), 
               text=f"Speed: Level {current_speed_index+1}"),
            Button((WIDTH // 2, HEIGHT * 0.45), int(30 * SCALE_FACTOR), 
               text=f"Difficulty: {difficulty_levels[current_difficulty_index]}"),
            Button((WIDTH // 2, HEIGHT * 0.55), int(30 * SCALE_FACTOR), text="Back")
    ]
    else:
        # Main menu buttons with proper positions
        buttons = [
            Button((WIDTH // 2, HEIGHT * 0.35), int(40 * SCALE_FACTOR), icon=play_icon),  # Play
            Button((WIDTH // 2 - int(120 * SCALE_FACTOR), HEIGHT * 0.5), int(30 * SCALE_FACTOR), text="Skin"),
            Button((WIDTH // 2 - int(40 * SCALE_FACTOR), HEIGHT * 0.5), int(30 * SCALE_FACTOR), text="Shape"),
            Button((WIDTH // 2 + int(40 * SCALE_FACTOR), HEIGHT * 0.5), int(30 * SCALE_FACTOR), 
                   icon=high_score_img or None, text="Scores" if high_score_img is None else None),
            Button((WIDTH // 2 + int(120 * SCALE_FACTOR), HEIGHT * 0.5), int(30 * SCALE_FACTOR), text="Multi"),
            Button((int(100 * SCALE_FACTOR), HEIGHT - int(100 * SCALE_FACTOR)), int(30 * SCALE_FACTOR), icon=settings_icon),
            Button((WIDTH - int(100 * SCALE_FACTOR), HEIGHT - int(100 * SCALE_FACTOR)), int(30 * SCALE_FACTOR), icon=exit_icon)
        ]

    for btn in buttons:
        btn.draw(screen, mouse_pos)

    pygame.display.update()
    return buttons, profile_rect if profile else None

def draw_sky_gradient_background():
    top_color = (173, 216, 230)
    bottom_color = (135, 206, 250)
    for y in range(HEIGHT):
        ratio = y / HEIGHT
        r = int(top_color[0] * (1 - ratio) + bottom_color[0] * ratio)
        g = int(top_color[1] * (1 - ratio) + bottom_color[1] * ratio)
        b = int(top_color[2] * (1 - ratio) + bottom_color[2] * ratio)
        pygame.draw.line(screen, (r, g, b), (0, y), (WIDTH, y))

def draw_snake():
    game.tongue_timer += 1
    if game.tongue_timer > 6:
        game.tongue_out = not game.tongue_out
        game.tongue_timer = 0
    game.tongue_length = BLOCK_SIZE if game.tongue_out else BLOCK_SIZE // 4
    game.update_word()
    
    # Get current skin
    current_skin = SNAKE_SKINS[current_skin_index]
    
    # Create colors
    if "color" in current_skin:
        # Single color skin
        primary_color = current_skin["color"]
        secondary_color = tuple(max(0, c - 40) for c in primary_color)
    else:
        # Multi-color skin
        primary_color = current_skin["colors"][0]  # Use first color as primary
        secondary_color = tuple(max(0, c - 40) for c in primary_color)
    
    # Draw each segment
    for i, pos in enumerate(game.snake):
        # For multi-color skins, select color for this segment
        if "colors" in current_skin and len(current_skin["colors"]) > 1:
            segment_color = current_skin["colors"][i % len(current_skin["colors"])]
        else:
            segment_color = primary_color
        
        if i == 0:  # Head
            draw_head_part(screen, pos, BLOCK_SIZE, game.head_part, game.direction, segment_color)
        elif i == len(game.snake) - 1:  # Tail
            # Determine tail direction based on previous segment
            if len(game.snake) > 1:
                prev_pos = game.snake[i-1]
                dx = prev_pos[0] - pos[0]
                dy = prev_pos[1] - pos[1]
                
                if dx > 0:  # Previous segment is to the right
                    tail_dir = "LEFT"
                elif dx < 0:  # Previous segment is to the left
                    tail_dir = "RIGHT"
                elif dy > 0:  # Previous segment is below
                    tail_dir = "UP"
                else:  # Previous segment is above
                    tail_dir = "DOWN"
            else:
                tail_dir = game.direction
                
            draw_tail_part(screen, pos, BLOCK_SIZE, game.tail_part, tail_dir, segment_color)
        else:  # Body
            # For multi-color skins, use segment-specific colors
            if "colors" in current_skin and len(current_skin["colors"]) > 1:
                body_primary = segment_color
                body_secondary = tuple(max(0, c - 40) for c in body_primary)
            else:
                body_primary = primary_color
                body_secondary = secondary_color
            
            draw_body_part(screen, pos, BLOCK_SIZE, game.body_part, body_primary, body_secondary)
        
        # Draw letter on body segments only
        if 0 < i < len(game.snake) - 1 and i - 1 < len(game.current_word):
            char = game.current_word[i - 1]
            label = small_font.render(char, True, WHITE)
            screen.blit(label, (pos[0] + BLOCK_SIZE//4, pos[1] + BLOCK_SIZE//4))

def draw_multiplayer_snake(player, skin_idx):
    """
    Draw one snake for multiplayer: custom head/body/tail plus the
    player's name spelled letter‑by‑letter along body segments.
    """
    skin = SNAKE_SKINS[skin_idx]
    primary = skin.get("color", skin.get("colors", [(0, 200, 0)])[0])

    letters = (player["name"] + "  ") * 50   # repeat plenty
    segs = player["snake"]

    for i, pos in enumerate(segs):
        if i == 0:  # head
            draw_head_part(screen, pos, BLOCK_SIZE,
                           SNAKE_PARTS["heads"][0],
                           player["direction"], primary)
        elif i == len(segs) - 1:  # tail
            draw_tail_part(screen, pos, BLOCK_SIZE,
                           SNAKE_PARTS["tails"][0],
                           player["direction"], primary)
        else:  # body with letters
            pygame.draw.rect(screen, primary,
                             (pos[0], pos[1], BLOCK_SIZE, BLOCK_SIZE))
            ch = letters[i - 1]
            lbl = small_font.render(ch, True, WHITE)
            screen.blit(lbl, (pos[0] + BLOCK_SIZE // 4,
                              pos[1] + BLOCK_SIZE // 4))


def draw_rat_at_position(pos):
    x, y = pos
    bs = int(BLOCK_SIZE * 0.6)
    pygame.draw.ellipse(screen, RAT_COLOR, (x, y + bs // 4, int(bs * 1.5), bs))
    pygame.draw.circle(screen, RAT_COLOR, (x + bs * 3 // 2, y + bs // 2), bs // 2)
    pygame.draw.circle(screen, WHITE, (x + bs * 14 // 10, y + bs // 2), bs // 10)
    pygame.draw.circle(screen, PINK, (x + bs * 17 // 10, y + bs // 2), bs // 12)

def draw_big_rat_at_position(pos):
    x, y = pos
    pygame.draw.ellipse(screen, BIG_RAT_COLOR, (x, y, BLOCK_SIZE * 2, BLOCK_SIZE * 2))
    pygame.draw.circle(screen, WHITE, (x + BLOCK_SIZE // 2, y + BLOCK_SIZE), BLOCK_SIZE // 4)
    pygame.draw.circle(screen, WHITE, (x + BLOCK_SIZE * 3 // 2, y + BLOCK_SIZE), BLOCK_SIZE // 4)
    pygame.draw.circle(screen, BLACK, (x + BLOCK_SIZE // 2, y + BLOCK_SIZE), BLOCK_SIZE // 8)
    pygame.draw.circle(screen, BLACK, (x + BLOCK_SIZE * 3 // 2, y + BLOCK_SIZE), BLOCK_SIZE // 8)
    pygame.draw.circle(screen, PINK, (x + BLOCK_SIZE, y + BLOCK_SIZE * 1.2), BLOCK_SIZE // 5)


def draw_multiplayer_ui(player1, player2):
    """Names & scores at the edges, controls hint centred."""
    # Player 1 (left)
    name1  = font.render(player1["name"], True, WHITE)
    score1 = font.render(f"Score: {player1['score']}", True, WHITE)
    screen.blit(name1,  (20, 20))
    screen.blit(score1, (20, 60))

    # Player 2 (right)
    name2  = font.render(player2["name"], True, WHITE)
    score2 = font.render(f"Score: {player2['score']}", True, WHITE)
    screen.blit(name2,  (WIDTH - name2.get_width()  - 20, 20))
    screen.blit(score2, (WIDTH - score2.get_width() - 20, 60))

    # Quick reminder
    hint = small_font.render("P1: WASD   |   P2: Arrow Keys", True, WHITE)
    screen.blit(hint, (WIDTH // 2 - hint.get_width() // 2, HEIGHT - 40))

def draw_multiplayer_game_over(winner, player1, player2):
    overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
    overlay.fill((0, 0, 0, 150))
    screen.blit(overlay, (0, 0))
    
    if winner == "Tie":
        title = big_font.render("Game Tied!", True, WHITE)
    else:
        title = big_font.render(f"{winner} Wins!", True, WHITE)
    
    score1 = font.render(f"{player1['name']}: {player1['score']}", True, WHITE)
    score2 = font.render(f"{player2['name']}: {player2['score']}", True, WHITE)
    
    replay_btn = pygame.Rect(WIDTH // 2 - 160, HEIGHT // 2 + 60, 140, 50)
    menu_btn = pygame.Rect(WIDTH // 2 + 20, HEIGHT // 2 + 60, 140, 50)
    
    pygame.draw.rect(screen, (50, 150, 50), replay_btn, border_radius=5)
    pygame.draw.rect(screen, (200, 50, 50), menu_btn, border_radius=5)
    
    replay_text = font.render("Replay", True, WHITE)
    menu_text = font.render("Menu", True, WHITE)
    
    screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 3))
    screen.blit(score1, (WIDTH // 2 - score1.get_width() // 2, HEIGHT // 3 + 80))
    screen.blit(score2, (WIDTH // 2 - score2.get_width() // 2, HEIGHT // 3 + 120))
    screen.blit(replay_text, (replay_btn.centerx - replay_text.get_width() // 2, replay_btn.centery - replay_text.get_height() // 2))
    screen.blit(menu_text, (menu_btn.centerx - menu_text.get_width() // 2, menu_btn.centery - menu_text.get_height() // 2))
    
    pygame.display.flip()
    
    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if replay_btn.collidepoint(pos):
                    return "replay"
                elif menu_btn.collidepoint(pos):
                    return "menu"
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_r:  # R for replay
                    return "replay"
                elif event.key == pygame.K_m or event.key == pygame.K_ESCAPE:  # M or ESC for menu
                    return "menu"

def draw_rat():
    # Draw regular rat (single block)
    if game.regular_rat_position:
        pos = game.regular_rat_position
        x, y = pos
        bs = int(BLOCK_SIZE * 0.6)
        pygame.draw.ellipse(screen, RAT_COLOR, (x, y + bs // 4, int(bs * 1.5), bs))
        pygame.draw.circle(screen, RAT_COLOR, (x + bs * 3 // 2, y + bs // 2), bs // 2)
        pygame.draw.circle(screen, WHITE, (x + bs * 14 // 10, y + bs // 2), bs // 10)
        pygame.draw.circle(screen, PINK, (x + bs * 17 // 10, y + bs // 2), bs // 12)

def draw_big_rat():
    # Draw big rat (2x2 blocks)
    if game.big_rat_position:
        pos = game.big_rat_position
        x, y = pos
        
        # Draw body (2x2 blocks)
        pygame.draw.ellipse(screen, BIG_RAT_COLOR, (x, y, BLOCK_SIZE * 2, BLOCK_SIZE * 2))
        
        # Draw ears
        pygame.draw.circle(screen, BIG_RAT_COLOR, (x + BLOCK_SIZE // 3, y + BLOCK_SIZE // 3), BLOCK_SIZE // 3)
        pygame.draw.circle(screen, BIG_RAT_COLOR, (x + BLOCK_SIZE * 5 // 3, y + BLOCK_SIZE // 3), BLOCK_SIZE // 3)
        
        # Draw eyes
        pygame.draw.circle(screen, WHITE, (x + BLOCK_SIZE // 2, y + BLOCK_SIZE), BLOCK_SIZE // 4)
        pygame.draw.circle(screen, WHITE, (x + BLOCK_SIZE * 3 // 2, y + BLOCK_SIZE), BLOCK_SIZE // 4)
        pygame.draw.circle(screen, BLACK, (x + BLOCK_SIZE // 2, y + BLOCK_SIZE), BLOCK_SIZE // 8)
        pygame.draw.circle(screen, BLACK, (x + BLOCK_SIZE * 3 // 2, y + BLOCK_SIZE), BLOCK_SIZE // 8)
        
        # Draw nose
        pygame.draw.circle(screen, PINK, (x + BLOCK_SIZE, y + BLOCK_SIZE * 1.2), BLOCK_SIZE // 5)

def draw_obstacles():
    for obs in game.obstacles:
        if obs is None:  # Skip None values
            continue
        pygame.draw.rect(screen, OBSTACLE_COLOR, (obs[0], obs[1], BLOCK_SIZE, BLOCK_SIZE))

def show_game_ui():
    # Score display
    score_label = font.render(f"Score: {game.score}", True, WHITE)
    screen.blit(score_label, (10, 10))
    
    # Rats eaten counter
    rats_label = font.render(f"Rats: {game.rats_eaten}/5", True, WHITE)
    screen.blit(rats_label, (10, 50))
    
    # Big rat timer if active
    if game.big_rat_active and game.big_rat_timer > 0:
        timer_label = font.render(f"Big Rat: {game.big_rat_timer:.1f}s", True, WHITE)
        screen.blit(timer_label, (10, 90))

def game_over(high_score_manager, profile):
    # Update profile with game results
    if profile:
        profile.update_high_score(game.score)
        profile.add_exp(game.score // 10)  # 1 EXP per 10 points
        profile.increment_games_played()
        profile.save()
    
    high_score_manager.add_score(game.score)
    # Draw semi-transparent overlay
    overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
    overlay.fill((0, 0, 0, 150))
    screen.blit(overlay, (0, 0))
    msg = big_font.render("GAME OVER", True, RED)
    score = font.render(f"Your Score: {game.score}", True, WHITE)
    restart = font.render("Press R to return to Menu or Q to quit", True, WHITE)
    screen.blit(msg, (WIDTH // 2 - msg.get_width() // 2, HEIGHT // 2 - 60 * SCALE_FACTOR))
    screen.blit(score, (WIDTH // 2 - score.get_width() // 2, HEIGHT // 2))
    screen.blit(restart, (WIDTH // 2 - restart.get_width() // 2, HEIGHT // 2 + 50 * SCALE_FACTOR))
    pygame.display.flip()
    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT or (event.type == pygame.KEYDOWN and event.key == pygame.K_q):
                return False
            if event.type == pygame.KEYDOWN and event.key == pygame.K_r:
                return True
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                return True


def check_regular_rat_collision(self, new_head):
    """Check if snake head collides with regular rat using grid-aligned positions"""
    if not self.regular_rat_position:
        return False
    
    # Convert to grid coordinates for reliable comparison
    head_grid = (new_head[0] // BLOCK_SIZE, 
                 new_head[1] // BLOCK_SIZE)
    rat_grid = (self.regular_rat_position[0] // BLOCK_SIZE,
                self.regular_rat_position[1] // BLOCK_SIZE)
    
    return head_grid == rat_grid

def check_big_rat_collision(self, new_head):
    """Check if snake head collides with big rat using precise rectangle collision"""
    if not self.big_rat_position or not self.big_rat_active:
        return False
    
    # Create rectangles for collision detection
    head_rect = pygame.Rect(new_head[0], new_head[1], 
                           BLOCK_SIZE, BLOCK_SIZE)
    big_rat_rect = pygame.Rect(self.big_rat_position[0],
                              self.big_rat_position[1],
                              BLOCK_SIZE*2, BLOCK_SIZE*2)
    
    return head_rect.colliderect(big_rat_rect)

def show_speed_selection():
    global current_speed_index, speed_levels, video_playing
    
    # Panel dimensions
    panel_width = int(WIDTH * 0.8)
    panel_height = int(HEIGHT * 0.5)
    panel_x = (WIDTH - panel_width) // 2
    panel_y = (HEIGHT - panel_height) // 2
    
    # Animation state
    panel_scale = 0.1
    growing = True
    applied_visible = False
    applied_timer = 0
    selected_index = current_speed_index
    
    # Main loop for speed selection
    running = True
    while running:
        # Draw video background continuously
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Handle animation
        if growing:
            panel_scale = min(panel_scale + 0.1, 1.0)
            if panel_scale >= 1.0:
                growing = False
        
        # Handle applied message timer
        if applied_visible:
            applied_timer += 1
            if applied_timer > 90:  # 1.5 seconds at 60 FPS
                applied_visible = False
        
        # Draw dimmed overlay
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 150))
        screen.blit(overlay, (0, 0))
        
        # Draw panel with scaling animation
        scaled_width = int(panel_width * panel_scale)
        scaled_height = int(panel_height * panel_scale)
        scaled_x = panel_x + (panel_width - scaled_width) // 2
        scaled_y = panel_y + (panel_height - scaled_height) // 2
        
        pygame.draw.rect(screen, (40, 40, 80), (scaled_x, scaled_y, scaled_width, scaled_height), border_radius=20)
        pygame.draw.rect(screen, (70, 70, 120), (scaled_x, scaled_y, scaled_width, scaled_height), 3, border_radius=20)
        
        # Only draw contents when fully scaled
        if panel_scale >= 1.0:
            # Title
            title = big_font.render("SELECT SNAKE SPEED", True, WHITE)
            screen.blit(title, (panel_x + (panel_width - title.get_width()) // 2, panel_y + 30))
            
            # Speed level buttons
            level_buttons = []
            button_width = 50 * SCALE_FACTOR
            button_height = 50 * SCALE_FACTOR
            button_spacing = 10 * SCALE_FACTOR
            total_width = (10 * button_width) + (9 * button_spacing)
            start_x = panel_x + (panel_width - total_width) // 2
            
            mouse_pos = pygame.mouse.get_pos()
            
            for i in range(10):
                x = start_x + (i * (button_width + button_spacing))
                y = panel_y + 100 * SCALE_FACTOR
                rect = pygame.Rect(x, y, button_width, button_height)
                level_buttons.append(rect)
                
                # Button color based on selection
                if i == selected_index:
                    color = (50, 200, 50)  # Selected color
                    # Pulse animation for selected button
                    pulse_size = int(5 * math.sin(pygame.time.get_ticks() / 200))
                    pygame.draw.rect(screen, color, rect.inflate(pulse_size, pulse_size), border_radius=5)
                elif rect.collidepoint(mouse_pos):
                    color = (100, 100, 200)  # Hover color
                else:
                    color = (70, 70, 120)  # Default color
                
                pygame.draw.rect(screen, color, rect, border_radius=5)
                
                # Level number
                level_text = font.render(str(i+1), True, WHITE)
                screen.blit(level_text, (rect.centerx - level_text.get_width() // 2, 
                                        rect.centery - level_text.get_height() // 2))
            
            # Speed indicators
            indicators_y = panel_y + 170 * SCALE_FACTOR
            turtle_x = start_x + (button_width // 2)
            person_x = start_x + (4.5 * (button_width + button_spacing))
            cheetah_x = start_x + (9.5 * (button_width + button_spacing))
            
            # Emoji indicators (using text as fallback)
            try:
                turtle = font.render("🐢", True, WHITE)
                person = font.render("🚶", True, WHITE)
                cheetah = font.render("🐆", True, WHITE)
                screen.blit(turtle, (turtle_x - turtle.get_width() // 2, indicators_y))
                screen.blit(person, (person_x - person.get_width() // 2, indicators_y))
                screen.blit(cheetah, (cheetah_x - cheetah.get_width() // 2, indicators_y))
            except:
                # Fallback text if emoji rendering fails
                turtle = small_font.render("Slow", True, WHITE)
                person = small_font.render("Normal", True, WHITE)
                cheetah = small_font.render("Fast", True, WHITE)
                screen.blit(turtle, (turtle_x - turtle.get_width() // 2, indicators_y))
                screen.blit(person, (person_x - person.get_width() // 2, indicators_y))
                screen.blit(cheetah, (cheetah_x - cheetah.get_width() // 2, indicators_y))
            
            # Current speed display
            speed_text = font.render(f"Current: {speed_levels[selected_index]}ms per step", True, WHITE)
            screen.blit(speed_text, (panel_x + (panel_width - speed_text.get_width()) // 2, 
                                    panel_y + 220 * SCALE_FACTOR))
            
            # Action buttons
            back_button = pygame.Rect(panel_x + 50, panel_y + panel_height - 70, 120, 50)
            apply_button = pygame.Rect(panel_x + panel_width - 170, panel_y + panel_height - 70, 120, 50)
            
            # Draw buttons
            pygame.draw.rect(screen, (200, 50, 50), back_button, border_radius=10)
            pygame.draw.rect(screen, (50, 150, 50), apply_button, border_radius=10)
            
            back_text = font.render("Back", True, WHITE)
            apply_text = font.render("Apply", True, WHITE)
            
            screen.blit(back_text, (back_button.centerx - back_text.get_width() // 2, 
                                   back_button.centery - back_text.get_height() // 2))
            screen.blit(apply_text, (apply_button.centerx - apply_text.get_width() // 2, 
                                    apply_button.centery - apply_text.get_height() // 2))
            
            # Applied confirmation
            if applied_visible:
                applied_text = font.render("Applied!", True, (50, 200, 50))
                alpha = min(255, applied_timer * 3)  # Fade in
                if applied_timer > 60:  # Start fading out after 1 second
                    alpha = 255 - ((applied_timer - 60) * 8)
                applied_text.set_alpha(alpha)
                screen.blit(applied_text, (panel_x + (panel_width - applied_text.get_width()) // 2, 
                                          panel_y + panel_height - 120))
            
            # Event handling
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    pygame.quit()
                    sys.exit()
                
                if event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_ESCAPE:
                        running = False
                    elif event.key == pygame.K_LEFT:
                        selected_index = max(0, selected_index - 1)
                    elif event.key == pygame.K_RIGHT:
                        selected_index = min(9, selected_index + 1)
                    elif event.key == pygame.K_RETURN and not applied_visible:
                        current_speed_index = selected_index
                        applied_visible = True
                
                if event.type == pygame.MOUSEBUTTONDOWN and not applied_visible:
                    pos = event.pos
                    
                    # Check level buttons
                    for i, button in enumerate(level_buttons):
                        if button.collidepoint(pos):
                            selected_index = i
                    
                    # Check action buttons
                    if back_button.collidepoint(pos):
                        running = False
                    elif apply_button.collidepoint(pos):
                        current_speed_index = selected_index
                        applied_visible = True
        
        pygame.display.flip()
        clock.tick(60)

# Modified function to directly show multiplayer mode selection
def show_multiplayer_menu():
    """Show the multiplayer mode selection menu (classic/war/back)"""
    buttons = []
    
    # Create buttons
    classic_btn = Button((WIDTH // 2, HEIGHT * 0.35), int(40 * SCALE_FACTOR), 
                        text="Classic", color=MULTIPLAYER_BUTTON_COLOR,
                        hover_color=MULTIPLAYER_BUTTON_HOVER)
    war_btn = Button((WIDTH // 2, HEIGHT * 0.50), int(40 * SCALE_FACTOR), 
                 text="War", color=MULTIPLAYER_BUTTON_COLOR,
                 hover_color=MULTIPLAYER_BUTTON_HOVER)
    back_btn = Button((WIDTH // 2, HEIGHT * 0.65), int(40 * SCALE_FACTOR), 
              text="Back", color=MULTIPLAYER_BUTTON_COLOR,
              hover_color=MULTIPLAYER_BUTTON_HOVER)
    
    buttons.extend([classic_btn, war_btn, back_btn])
    
    while True:
        # Draw background
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Title
        title = big_font.render("Multiplayer Mode", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT * 0.15))
        
        # Draw buttons
        mouse_pos = pygame.mouse.get_pos()
        for btn in buttons:
            btn.draw(screen, mouse_pos)
        
        pygame.display.flip()
        
        # Event handling
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                return "back"
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if classic_btn.is_clicked(pos):
                    # Show multiplayer classic setup
                    setup_data = show_multiplayer_classic_setup()
                    if setup_data:
                        player1_name, player1_skin, player2_name, player2_skin, speed_level = setup_data
                        while True:
                            result = multiplayer_classic_game_loop(
                                player1_name, player1_skin, 
                                player2_name, player2_skin, 
                                speed_level
                            )
                            if result == "menu":
                                break
                            elif result == "replay":
                                continue  # Will restart the game loop
                elif war_btn.is_clicked(pos):
                    setup = show_multiplayer_war_setup()
                    if setup:
                        p1_name, p1_skin, p2_name, p2_skin, speed = setup
                        while True:
                            res = multiplayer_war_game_loop(p1_name, p1_skin, p2_name, p2_skin, speed)
                            if res == "menu":
                                break
                            elif res == "replay":
                                continue
                elif back_btn.is_clicked(pos):
                    return "back"

def show_message(title, message):
    """Show a simple message popup"""
    popup_width = int(WIDTH * 0.6)
    popup_height = int(HEIGHT * 0.3)
    popup_x = (WIDTH - popup_width) // 2
    popup_y = (HEIGHT - popup_height) // 2
    # Shadow effect
    shadow = pygame.Surface((popup_width + 10, popup_height + 10), pygame.SRCALPHA)
    shadow.fill((0, 0, 0, 100))
    screen.blit(shadow, (popup_x - 5, popup_y - 5))
    # Main popup background
    pygame.draw.rect(screen, (240, 240, 250), (popup_x, popup_y, popup_width, popup_height), border_radius=20)
    # Title
    title_surface = font.render(title, True, BLACK)
    screen.blit(title_surface, (popup_x + (popup_width - title_surface.get_width()) // 2, popup_y + 20))
    
    # Message (split into multiple lines if needed)
    words = message.split(' ')
    lines = []
    current_line = ""
    for word in words:
        test_line = current_line + word + " "
        if font.size(test_line)[0] < popup_width - 40:
            current_line = test_line
        else:
            lines.append(current_line)
            current_line = word + " "
    lines.append(current_line)
    
    for i, line in enumerate(lines):
        line_surface = small_font.render(line, True, BLACK)
        screen.blit(line_surface, (popup_x + 20, popup_y + 60 + i * 30))
    
    # OK button
    ok_button = pygame.Rect(popup_x + popup_width // 2 - 50, popup_y + popup_height - 70, 100, 50)
    pygame.draw.rect(screen, BUTTON_COLOR, ok_button, border_radius=10)
    ok_text = font.render("OK", True, WHITE)
    screen.blit(ok_text, (ok_button.centerx - ok_text.get_width() // 2,ok_button.centery - ok_text.get_height() // 2))
    
    pygame.display.flip()
    
    # Wait for OK click
    waiting = True
    while waiting:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if ok_button.collidepoint(pos):
                    waiting = False
            if event.type == pygame.KEYDOWN and (event.key == pygame.K_RETURN or event.key == pygame.K_ESCAPE):
                waiting = False

def show_half_screen_shape_menu(cur_head, cur_body, cur_tail, side):

    assert side in ("left", "right"), "side must be 'left' or 'right'"

    panel_w, panel_h = WIDTH // 2, HEIGHT
    panel_x, panel_y = (0, 0) if side == "left" else (panel_w, 0)

    tabs = ["Head", "Body", "Tail"]
    cur_tab = 0
    head_idx, body_idx, tail_idx = cur_head, cur_body, cur_tail

    cols, rows = 5, 2                       # 10 thumbnails per tab
    cell_w, cell_h = panel_w // cols, panel_h // rows

    def parts_for(tab):
        return (SNAKE_PARTS["heads"]  if tab == 0 else
                SNAKE_PARTS["bodies"] if tab == 1 else
                SNAKE_PARTS["tails"])

    running = True
    clock = pygame.time.Clock()
    while running:
        # ── draw dimmed video background ────────────────────────────────
        frame = get_video_frame()
        if frame: screen.blit(frame, (0, 0))
        dim = pygame.Surface((panel_w, panel_h), pygame.SRCALPHA)
        dim.fill((0, 0, 0, 140))
        screen.blit(dim, (panel_x, panel_y))

        # ── tab bar ─────────────────────────────────────────────────────
        for i, t in enumerate(tabs):
            r = pygame.Rect(panel_x + i * (panel_w // 3), panel_y, panel_w // 3, 50)
            pygame.draw.rect(screen, BUTTON_HOVER if i == cur_tab else BUTTON_COLOR, r)
            lbl = font.render(t, True, WHITE)
            screen.blit(lbl, (r.centerx - lbl.get_width() // 2, r.centery - lbl.get_height() // 2))

        # ── thumbnails ─────────────────────────────────────────────────
        parts = parts_for(cur_tab)
        for idx, part in enumerate(parts):
            row, col = divmod(idx, cols)
            gx = panel_x + col * cell_w
            gy = panel_y + 60 + row * cell_h
            cell_rect = pygame.Rect(gx, gy, cell_w, cell_h)
            pygame.draw.rect(screen, BUTTON_HOVER if ((cur_tab == 0 and idx == head_idx) or (cur_tab == 1 and idx == body_idx) or  (cur_tab == 2 and idx == tail_idx)) else BUTTON_COLOR, cell_rect)

            part_size = min(cell_w, cell_h) * 0.75
            px = gx + (cell_w - part_size) // 2
            py = gy + (cell_h - part_size) // 2
            skin_entry = SNAKE_SKINS[current_skin_index]
            if "color" in skin_entry:                       # single‑color skin
                primary = skin_entry["color"]
            elif "colors" in skin_entry and skin_entry["colors"]:  # multi‑color skin
                primary = skin_entry["colors"][0]
            else:                                           # totally missing → fall back
                primary = (0, 200, 0)                       # any default you like

            if cur_tab == 0:
                draw_head_part(screen, (px, py), part_size, part, "RIGHT", primary)
            elif cur_tab == 1:
                sec = tuple(max(0, c-40) for c in primary)
                draw_body_part(screen, (px, py), part_size, part, primary, sec)
            else:
                draw_tail_part(screen, (px, py), part_size, part, "RIGHT", primary)

        # ── OK / Back buttons ───────────────────────────────────────────
        ok  = pygame.Rect(panel_x + panel_w//2 + 20, panel_h - 70, 100, 50)
        bk  = pygame.Rect(panel_x + panel_w//2 - 120, panel_h - 70, 100, 50)
        for rect, txt in ((ok, "OK"), (bk, "Back")):
            pygame.draw.rect(screen, (50,150,50) if rect == ok else (200,50,50), rect, border_radius=8)
            tl = font.render(txt, True, WHITE)
            screen.blit(tl, (rect.centerx - tl.get_width()//2, rect.centery - tl.get_height()//2))

        pygame.display.flip()
        clock.tick(60)

        # ── events ─────────────────────────────────────────────────────
        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                pygame.quit(); sys.exit()
            if e.type == pygame.KEYDOWN and e.key == pygame.K_ESCAPE:
                return cur_head, cur_body, cur_tail
            if e.type == pygame.MOUSEBUTTONDOWN:
                mx, my = e.pos
                # tab clicks
                if panel_y <= my <= panel_y+50 and panel_x <= mx < panel_x+panel_w:
                    cur_tab = (mx - panel_x) // (panel_w // 3)
                # grid clicks
                if (panel_y+60) <= my < (panel_h-80) and panel_x <= mx < panel_x+panel_w:
                    col = (mx - panel_x) // cell_w
                    row = (my - (panel_y+60)) // cell_h
                    idx = row*cols + col
                    if idx < len(parts):
                        if   cur_tab == 0: head_idx = idx
                        elif cur_tab == 1: body_idx = idx
                        else:              tail_idx = idx
                # buttons
                if ok.collidepoint((mx,my)):
                    return head_idx, body_idx, tail_idx
                if bk.collidepoint((mx,my)):
                    return cur_head, cur_body, cur_tail

def show_multiplayer_war_setup():
    player1_name = "Player 1"
    player2_name = "Player 2"
    player1_skin = 0
    player2_skin = 1
    current_head_index = 0
    current_body_index = 0
    current_tail_index = 0

    current_speed = current_speed_index
    active_input = None  # None, "player1", or "player2"
    
    # Define button positions
    player1_rect = pygame.Rect(WIDTH // 4 - 150, HEIGHT // 3, 300, 40)
    player1_skin_btn = pygame.Rect(WIDTH // 4 - 150, HEIGHT // 3 + 60, 300, 40)
    player2_rect = pygame.Rect(3 * WIDTH // 4 - 150, HEIGHT // 3, 300, 40)
    player2_skin_btn = pygame.Rect(3 * WIDTH // 4 - 150, HEIGHT // 3 + 60, 300, 40)
    player1_shape_btn = pygame.Rect(WIDTH // 4 + 170,  HEIGHT // 3 + 60, 60, 40)
    player2_shape_btn = pygame.Rect(3 * WIDTH // 4 - 230, HEIGHT // 3 + 60, 60, 40)
    speed_btn = pygame.Rect(WIDTH // 2 - 150, HEIGHT // 2, 300, 40)
    start_btn = pygame.Rect(WIDTH // 2 - 150, HEIGHT * 2 // 3, 140, 50)
    back_btn = pygame.Rect(WIDTH // 2 + 10, HEIGHT * 2 // 3, 140, 50)
    
    while True:
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Draw title
        title = big_font.render("Multiplayer War Setup", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 6))
        
        # Draw player sections
        pygame.draw.rect(screen, (40, 40, 80), player1_rect, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), player1_skin_btn, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), player2_rect, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), player2_skin_btn, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), speed_btn, border_radius=5)
        pygame.draw.rect(screen, (50, 150, 50), start_btn, border_radius=5)
        pygame.draw.rect(screen, (200, 50, 50), back_btn, border_radius=5)
        
        # Draw labels
        player1_label = font.render(f"Player 1: {player1_name}", True, WHITE)
        player2_label = font.render(f"Player 2: {player2_name}", True, WHITE)
        skin1_label = font.render(f"Skin: {SNAKE_SKINS[player1_skin]['name']}", True, WHITE)
        skin2_label = font.render(f"Skin: {SNAKE_SKINS[player2_skin]['name']}", True, WHITE)
        speed_label = font.render(f"Speed: Level {current_speed+1}", True, WHITE)
        start_label = font.render("Start", True, WHITE)
        back_label = font.render("Back", True, WHITE)
        
        screen.blit(player1_label, (player1_rect.centerx - player1_label.get_width() // 2, player1_rect.centery - player1_label.get_height() // 2))
        screen.blit(player2_label, (player2_rect.centerx - player2_label.get_width() // 2, player2_rect.centery - player2_label.get_height() // 2))
        screen.blit(skin1_label, (player1_skin_btn.centerx - skin1_label.get_width() // 2, player1_skin_btn.centery - skin1_label.get_height() // 2))
        screen.blit(skin2_label, (player2_skin_btn.centerx - skin2_label.get_width() // 2, player2_skin_btn.centery - skin2_label.get_height() // 2))
        # draw Shape buttons
        pygame.draw.rect(screen, (80, 120, 80), player1_shape_btn, border_radius=5)
        pygame.draw.rect(screen, (80, 120, 80), player2_shape_btn, border_radius=5)
        shape_lbl = small_font.render("Shape", True, WHITE)
        for btn in (player1_shape_btn, player2_shape_btn):
            screen.blit(shape_lbl, (btn.centerx - shape_lbl.get_width() // 2,btn.centery - shape_lbl.get_height() // 2))

        screen.blit(speed_label, (speed_btn.centerx - speed_label.get_width() // 2, speed_btn.centery - speed_label.get_height() // 2))
        screen.blit(start_label, (start_btn.centerx - start_label.get_width() // 2, start_btn.centery - start_label.get_height() // 2))
        screen.blit(back_label, (back_btn.centerx - back_label.get_width() // 2, back_btn.centery - back_label.get_height() // 2))
        
        # Draw skin previews
        skin_preview_size = 30
        preview1_pos = (player1_skin_btn.centerx - skin_preview_size - 50, player1_skin_btn.centery)
        preview2_pos = (player2_skin_btn.centerx - skin_preview_size - 50, player2_skin_btn.centery)
        draw_head_part(screen, preview1_pos, skin_preview_size, SNAKE_PARTS["heads"][0], "RIGHT", SNAKE_SKINS[player1_skin]["color"] if "color" in SNAKE_SKINS[player1_skin] else SNAKE_SKINS[player1_skin]["colors"][0])
        draw_head_part(screen, preview2_pos, skin_preview_size, SNAKE_PARTS["heads"][0], "RIGHT", SNAKE_SKINS[player2_skin]["color"] if "color" in SNAKE_SKINS[player2_skin] else SNAKE_SKINS[player2_skin]["colors"][0])
        
        pygame.display.flip()
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if player1_rect.collidepoint(pos):
                    active_input = "player1"
                    player1_name = ""
                elif player2_rect.collidepoint(pos):
                    active_input = "player2"
                    player2_name = ""
                elif player1_skin_btn.collidepoint(pos):
                    player1_skin = (player1_skin + 1) % len(SNAKE_SKINS)
                elif player2_skin_btn.collidepoint(pos):
                    player2_skin = (player2_skin + 1) % len(SNAKE_SKINS)
                elif player1_shape_btn.collidepoint(pos):
                    head, body, tail = show_half_screen_shape_menu(
                        current_head_index, current_body_index, current_tail_index, "left")
                    current_head_index, current_body_index, current_tail_index = head, body, tail
                elif player2_shape_btn.collidepoint(pos):
                    head, body, tail = show_half_screen_shape_menu(
                        current_head_index, current_body_index, current_tail_index, "right")
                    current_head_index, current_body_index, current_tail_index = head, body, tail
                elif speed_btn.collidepoint(pos):
                    current_speed = (current_speed + 1) % len(speed_levels)
                elif start_btn.collidepoint(pos):
                    return player1_name, player1_skin, player2_name, player2_skin, current_speed
                elif back_btn.collidepoint(pos):
                    return None
            if event.type == pygame.KEYDOWN:
                if active_input == "player1":
                    if event.key == pygame.K_RETURN:
                        active_input = None
                    elif event.key == pygame.K_BACKSPACE:
                        player1_name = player1_name[:-1]
                    elif event.unicode.isprintable():
                        player1_name += event.unicode
                elif active_input == "player2":
                    if event.key == pygame.K_RETURN:
                        active_input = None
                    elif event.key == pygame.K_BACKSPACE:
                        player2_name = player2_name[:-1]
                    elif event.unicode.isprintable():
                        player2_name += event.unicode
                elif event.key == pygame.K_ESCAPE:
                    return None

def show_multiplayer_classic_setup():
    player1_name = "Player 1"
    player2_name = "Player 2"
    player1_skin = 0
    player2_skin = 1
    current_head_index = 0
    current_body_index = 0
    current_tail_index = 0

    current_speed = current_speed_index
    active_input = None  # None, "player1", or "player2"
    
    # Define button positions
    player1_rect = pygame.Rect(WIDTH // 4 - 150, HEIGHT // 3, 300, 40)
    player1_skin_btn = pygame.Rect(WIDTH // 4 - 150, HEIGHT // 3 + 60, 300, 40)
    player2_rect = pygame.Rect(3 * WIDTH // 4 - 150, HEIGHT // 3, 300, 40)
    player2_skin_btn = pygame.Rect(3 * WIDTH // 4 - 150, HEIGHT // 3 + 60, 300, 40)
    player1_shape_btn = pygame.Rect(WIDTH // 4 + 170,  HEIGHT // 3 + 60, 60, 40)
    player2_shape_btn = pygame.Rect(3 * WIDTH // 4 - 230, HEIGHT // 3 + 60, 60, 40)
    speed_btn = pygame.Rect(WIDTH // 2 - 150, HEIGHT // 2, 300, 40)
    start_btn = pygame.Rect(WIDTH // 2 - 150, HEIGHT * 2 // 3, 140, 50)
    back_btn = pygame.Rect(WIDTH // 2 + 10, HEIGHT * 2 // 3, 140, 50)
    
    while True:
        frame = get_video_frame()
        if frame:
            screen.blit(frame, (0, 0))
        else:
            screen.fill((0, 30, 60))
        
        # Draw title
        title = big_font.render("Multiplayer Classic Setup", True, WHITE)
        screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 6))
        
        # Draw player sections
        pygame.draw.rect(screen, (40, 40, 80), player1_rect, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), player1_skin_btn, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), player2_rect, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), player2_skin_btn, border_radius=5)
        pygame.draw.rect(screen, (40, 40, 80), speed_btn, border_radius=5)
        pygame.draw.rect(screen, (50, 150, 50), start_btn, border_radius=5)
        pygame.draw.rect(screen, (200, 50, 50), back_btn, border_radius=5)
        
        # Draw labels
        player1_label = font.render(f"Player 1: {player1_name}", True, WHITE)
        player2_label = font.render(f"Player 2: {player2_name}", True, WHITE)
        skin1_label = font.render(f"Skin: {SNAKE_SKINS[player1_skin]['name']}", True, WHITE)
        skin2_label = font.render(f"Skin: {SNAKE_SKINS[player2_skin]['name']}", True, WHITE)
        speed_label = font.render(f"Speed: Level {current_speed+1}", True, WHITE)
        start_label = font.render("Start", True, WHITE)
        back_label = font.render("Back", True, WHITE)
        
        screen.blit(player1_label, (player1_rect.centerx - player1_label.get_width() // 2, player1_rect.centery - player1_label.get_height() // 2))
        screen.blit(player2_label, (player2_rect.centerx - player2_label.get_width() // 2, player2_rect.centery - player2_label.get_height() // 2))
        screen.blit(skin1_label, (player1_skin_btn.centerx - skin1_label.get_width() // 2, player1_skin_btn.centery - skin1_label.get_height() // 2))
        screen.blit(skin2_label, (player2_skin_btn.centerx - skin2_label.get_width() // 2, player2_skin_btn.centery - skin2_label.get_height() // 2))
        # draw Shape buttons
        pygame.draw.rect(screen, (80, 120, 80), player1_shape_btn, border_radius=5)
        pygame.draw.rect(screen, (80, 120, 80), player2_shape_btn, border_radius=5)
        shape_lbl = small_font.render("Shape", True, WHITE)
        for btn in (player1_shape_btn, player2_shape_btn):
            screen.blit(shape_lbl, (btn.centerx - shape_lbl.get_width() // 2,btn.centery - shape_lbl.get_height() // 2))

        screen.blit(speed_label, (speed_btn.centerx - speed_label.get_width() // 2, speed_btn.centery - speed_label.get_height() // 2))
        screen.blit(start_label, (start_btn.centerx - start_label.get_width() // 2, start_btn.centery - start_label.get_height() // 2))
        screen.blit(back_label, (back_btn.centerx - back_label.get_width() // 2, back_btn.centery - back_label.get_height() // 2))
        
        # Draw skin previews
        skin_preview_size = 30
        preview1_pos = (player1_skin_btn.centerx - skin_preview_size - 50, player1_skin_btn.centery)
        preview2_pos = (player2_skin_btn.centerx - skin_preview_size - 50, player2_skin_btn.centery)
        draw_head_part(screen, preview1_pos, skin_preview_size, SNAKE_PARTS["heads"][0], "RIGHT", SNAKE_SKINS[player1_skin]["color"] if "color" in SNAKE_SKINS[player1_skin] else SNAKE_SKINS[player1_skin]["colors"][0])
        draw_head_part(screen, preview2_pos, skin_preview_size, SNAKE_PARTS["heads"][0], "RIGHT", SNAKE_SKINS[player2_skin]["color"] if "color" in SNAKE_SKINS[player2_skin] else SNAKE_SKINS[player2_skin]["colors"][0])
        
        pygame.display.flip()
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                pos = event.pos
                if player1_rect.collidepoint(pos):
                    active_input = "player1"
                    player1_name = ""
                elif player2_rect.collidepoint(pos):
                    active_input = "player2"
                    player2_name = ""
                elif player1_skin_btn.collidepoint(pos):
                    player1_skin = (player1_skin + 1) % len(SNAKE_SKINS)
                elif player2_skin_btn.collidepoint(pos):
                    player2_skin = (player2_skin + 1) % len(SNAKE_SKINS)
                elif player1_shape_btn.collidepoint(pos):
                    head, body, tail = show_half_screen_shape_menu(
                        current_head_index, current_body_index, current_tail_index, "left")
                    current_head_index, current_body_index, current_tail_index = head, body, tail
                elif player2_shape_btn.collidepoint(pos):
                    head, body, tail = show_half_screen_shape_menu(
                        current_head_index, current_body_index, current_tail_index, "right")
                    current_head_index, current_body_index, current_tail_index = head, body, tail
                elif speed_btn.collidepoint(pos):
                    current_speed = (current_speed + 1) % len(speed_levels)
                elif start_btn.collidepoint(pos):
                    return player1_name, player1_skin, player2_name, player2_skin, current_speed
                elif back_btn.collidepoint(pos):
                    return None
            if event.type == pygame.KEYDOWN:
                if active_input == "player1":
                    if event.key == pygame.K_RETURN:
                        active_input = None
                    elif event.key == pygame.K_BACKSPACE:
                        player1_name = player1_name[:-1]
                    elif event.unicode.isprintable():
                        player1_name += event.unicode
                elif active_input == "player2":
                    if event.key == pygame.K_RETURN:
                        active_input = None
                    elif event.key == pygame.K_BACKSPACE:
                        player2_name = player2_name[:-1]
                    elif event.unicode.isprintable():
                        player2_name += event.unicode
                elif event.key == pygame.K_ESCAPE:
                    return None

def multiplayer_classic_game_loop(player1_name, player1_skin,
                                  player2_name, player2_skin,
                                  speed_level):
    global video_playing
    video_playing = False

    game = MultiplayerGameState(player1_name, player1_skin,
                                player2_name, player2_skin,
                                speed_level)

    clock        = pygame.time.Clock()
    game_over    = False
    winner       = None

    p1_dead = p2_dead = False
    target_score = None            # score survivor must beat

    while True:
        dt = clock.tick(60) / 1000.0

        # ---------- UPDATE ----------
        if not game_over:
            result1, result2 = game.update(dt)

            # recognise both "DEAD" and "WIN" as terminal events
            def ended(r): return r in ("DEAD", "WIN")

            # mark deaths
            if result1 == "DEAD" and not p1_dead:
                p1_dead = True
                if not p2_dead:
                    target_score = game.player1["score"] + 1
            if result2 == "DEAD" and not p2_dead:
                p2_dead = True
                if not p1_dead:
                    target_score = game.player2["score"] + 1

            # -------- DECIDE WHEN THE LOOP ENDS --------
            # 1) Both players dead
            if p1_dead and p2_dead:
                game_over = True

            # 2) P1 dead, P2 alive
            elif p1_dead and not p2_dead:
                if game.player2["score"] >= target_score or result2 == "WIN":
                    winner = game.player2["name"]
                    game_over = True
                elif result2 == "DEAD":   # P2 crashed first
                    winner = (game.player2["name"]
                              if game.player2["score"] > game.player1["score"]
                              else game.player1["name"])
                    game_over = True
            # 3) P2 dead, P1 alive
            elif p2_dead and not p1_dead:
                if game.player1["score"] >= target_score or result1 == "WIN":
                    winner = game.player1["name"]
                    game_over = True
                elif result1 == "DEAD":
                    winner = (game.player1["name"]
                              if game.player1["score"] > game.player2["score"]
                              else game.player2["name"])
                    game_over = True
            # If the round ended but winner still None → compare scores / tie
            if game_over and winner is None:
                if game.player1["score"] > game.player2["score"]:
                    winner = game.player1["name"]
                elif game.player2["score"] > game.player1["score"]:
                    winner = game.player2["name"]
                else:
                    winner = "Tie"
        # ---------- INPUT ----------
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                video_playing = True
                return "menu"
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    video_playing = True
                    return "menu"
                # Player‑1 controls (arrows)
                if not game_over:
                    if event.key == pygame.K_w   and game.player1["direction"] != "DOWN":
                        game.player1["next_direction"] = "UP"
                    elif event.key == pygame.K_s and game.player1["direction"] != "UP":
                        game.player1["next_direction"] = "DOWN"
                    elif event.key == pygame.K_a and game.player1["direction"] != "RIGHT":
                        game.player1["next_direction"] = "LEFT"
                    elif event.key == pygame.K_d and game.player1["direction"] != "LEFT":
                        game.player1["next_direction"] = "RIGHT"
                # Player‑2 controls (WASD)
                if not game_over:
                    if event.key == pygame.K_UP and game.player2["direction"] != "DOWN":
                        game.player2["next_direction"] = "UP"
                    elif event.key == pygame.K_DOWN and game.player2["direction"] != "UP":
                        game.player2["next_direction"] = "DOWN"
                    elif event.key == pygame.K_LEFT and game.player2["direction"] != "RIGHT":
                        game.player2["next_direction"] = "LEFT"
                    elif event.key == pygame.K_RIGHT and game.player2["direction"] != "LEFT":
                        game.player2["next_direction"] = "RIGHT"
        # ---------- DRAWING ----------
        draw_sky_gradient_background()
        draw_grid()
        draw_border()
        # centre wall
        for pos in game.center_border:
            pygame.draw.rect(
                screen, OBSTACLE_COLOR,
                (pos[0] - BLOCK_SIZE // 2, pos[1], BLOCK_SIZE, BLOCK_SIZE)
            )
        # snakes
        draw_multiplayer_snake(game.player1, game.player1["skin"])
        draw_multiplayer_snake(game.player2, game.player2["skin"])
        # Player‑1 rats (right side)
        if game.player1["regular_rat"]:
            draw_rat_at_position(game.player1["regular_rat"])
        if game.player1["big_active"] and game.player1["big_rat"]:
            draw_big_rat_at_position(game.player1["big_rat"])
        # Player‑2 rats (left side)
        if game.player2["regular_rat"]:
            draw_rat_at_position(game.player2["regular_rat"])
        if game.player2["big_active"] and game.player2["big_rat"]:
            draw_big_rat_at_position(game.player2["big_rat"])
        # UI
        draw_multiplayer_ui(game.player1, game.player2)
        # ---------- GAME‑OVER OVERLAY ----------
        if game_over:
            choice = draw_multiplayer_game_over(winner, game.player1, game.player2)
            if choice == "replay":
                return "replay"
            if choice == "menu":
                video_playing = True
                return "menu"

        pygame.display.update()

def multiplayer_war_game_loop(p1_name, p1_skin,
                              p2_name, p2_skin,
                              speed_level):
    """
    War‑mode loop implementing:
    ▸ When one snake dies the survivor keeps playing
      until they reach (opponent_score + 1).
    ▸ If that survivor dies first, the higher score wins,
      otherwise it’s a tie when both scores match.
    """
    global video_playing          # if you pause menu video elsewhere
    video_playing = False

    # ------------------------------------------------------------------ #
    #  INITIALISE                                                        #
    # ------------------------------------------------------------------ #
    game          = MultiplayerWarGameState(p1_name, p1_skin,
                                            p2_name, p2_skin,
                                            speed_level)
    clock         = pygame.time.Clock()

    p1_dead       = False
    p2_dead       = False
    target_score  = None          # score survivor must beat
    winner        = None
    game_over     = False

    # ------------------------------------------------------------------ #
    #  MAIN LOOP                                                         #
    # ------------------------------------------------------------------ #
    while True:
        dt = clock.tick(60) / 1000.0

        # ---------- UPDATE PHASE -------------------------------------- #
        if not game_over:
            r1, r2 = game.update(dt)          # "ALIVE" | "DEAD"

            # --- detect first deaths and set the target ---------------- #
            if r1 == "DEAD" and not p1_dead:
                p1_dead = True
                if not p2_dead:               # P2 still alive
                    target_score = game.player1["score"] + 1
            if r2 == "DEAD" and not p2_dead:
                p2_dead = True
                if not p1_dead:               # P1 still alive
                    target_score = game.player2["score"] + 1

            # --- stopping conditions ---------------------------------- #
            # 1. both dead → compare scores immediately
            if p1_dead and p2_dead:
                game_over = True

            # 2. P1 dead, P2 alive
            elif p1_dead and not p2_dead:
                # P2 wins as soon as they reach / pass target
                if game.player2["score"] >= target_score:
                    winner = game.player2["name"]
                    game_over = True
                # P2 died first, so compare scores
                elif r2 == "DEAD":
                    if game.player2["score"] > game.player1["score"]:
                        winner = game.player2["name"]
                    elif game.player1["score"] > game.player2["score"]:
                        winner = game.player1["name"]
                    else:
                        winner = "Tie"
                    game_over = True

            # 3. P2 dead, P1 alive
            elif p2_dead and not p1_dead:
                if game.player1["score"] >= target_score:
                    winner = game.player1["name"]
                    game_over = True
                elif r1 == "DEAD":
                    if game.player1["score"] > game.player2["score"]:
                        winner = game.player1["name"]
                    elif game.player2["score"] > game.player1["score"]:
                        winner = game.player2["name"]
                    else:
                        winner = "Tie"
                    game_over = True

            # 4. reached here via “both dead” but winner still None
            if game_over and winner is None:
                if game.player1["score"] > game.player2["score"]:
                    winner = game.player1["name"]
                elif game.player2["score"] > game.player1["score"]:
                    winner = game.player2["name"]
                else:
                    winner = "Tie"

        # ---------- EVENT HANDLING (unchanged) ------------------------- #
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                video_playing = True
                return "menu"
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    video_playing = True
                    return "menu"
                # WASD for Player‑1
                if not p1_dead:
                    if event.key == pygame.K_w and game.player1["direction"] != "DOWN":
                        game.player1["next_direction"] = "UP"
                    elif event.key == pygame.K_s and game.player1["direction"] != "UP":
                        game.player1["next_direction"] = "DOWN"
                    elif event.key == pygame.K_a and game.player1["direction"] != "RIGHT":
                        game.player1["next_direction"] = "LEFT"
                    elif event.key == pygame.K_d and game.player1["direction"] != "LEFT":
                        game.player1["next_direction"] = "RIGHT"
                # Arrow keys for Player‑2
                if not p2_dead:
                    if event.key == pygame.K_UP and game.player2["direction"] != "DOWN":
                        game.player2["next_direction"] = "UP"
                    elif event.key == pygame.K_DOWN and game.player2["direction"] != "UP":
                        game.player2["next_direction"] = "DOWN"
                    elif event.key == pygame.K_LEFT and game.player2["direction"] != "RIGHT":
                        game.player2["next_direction"] = "LEFT"
                    elif event.key == pygame.K_RIGHT and game.player2["direction"] != "LEFT":
                        game.player2["next_direction"] = "RIGHT"

        # ---------- DRAW (unchanged) ----------------------------------- #
        draw_sky_gradient_background()
        draw_grid()
        draw_border()

        draw_multiplayer_snake(game.player1, game.player1["skin"])
        draw_multiplayer_snake(game.player2, game.player2["skin"])

        if game.shared_regular_rat:
            draw_rat_at_position(game.shared_regular_rat)
        if game.shared_big_active and game.shared_big_rat:
            draw_big_rat_at_position(game.shared_big_rat)

        draw_multiplayer_ui(game.player1, game.player2)

        # ---------- GAME‑OVER OVERLAY ---------------------------------- #
        if game_over:
            choice = draw_multiplayer_game_over(winner,
                                                game.player1, game.player2)
            if choice == "replay":
                return "replay"
            if choice == "menu":
                video_playing = True
                return "menu"

        pygame.display.update()



def main_game_loop(high_score_manager, profile):
    global game, video_playing
    growth = 0
    move_timer = 0
    move_delay = speed_levels[current_speed_index] / 1000.0  # Convert ms to seconds
    
    # Pause menu video during gameplay
    video_playing = False
    
    last_time = pygame.time.get_ticks()
    
    while True:
        current_time = pygame.time.get_ticks()
        dt = (current_time - last_time) / 1000.0
        last_time = current_time
        move_timer += dt
        
        # Update big rat timer
        if game.big_rat_active and game.big_rat_timer > 0:
            game.big_rat_timer -= dt
            if game.big_rat_timer <= 0:
                game.big_rat_active = False
                game.big_rat_position = None
        
        # Handle input
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                # Resume video playback before quitting
                video_playing = True
                return False
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_q or event.key == pygame.K_ESCAPE:
                    # Resume video playback
                    video_playing = True
                    return True
                elif event.key == pygame.K_UP and game.direction != "DOWN":
                    game.next_direction = "UP"
                elif event.key == pygame.K_DOWN and game.direction != "UP":
                    game.next_direction = "DOWN"
                elif event.key == pygame.K_LEFT and game.direction != "RIGHT":
                    game.next_direction = "LEFT"
                elif event.key == pygame.K_RIGHT and game.direction != "LEFT":
                    game.next_direction = "RIGHT"
        
        # Movement handling
        if move_timer >= move_delay:
            move_timer = 0
            
            # Update direction
            if game.next_direction:
                game.direction = game.next_direction
                game.next_direction = None
            
            # Calculate new head position
            x, y = game.snake[0]
            if game.direction == "UP": 
                y -= BLOCK_SIZE
            elif game.direction == "DOWN": 
                y += BLOCK_SIZE
            elif game.direction == "LEFT": 
                x -= BLOCK_SIZE
            elif game.direction == "RIGHT": 
                x += BLOCK_SIZE
            
            # Check if new head position hits the border
            if x < BLOCK_SIZE or x >= WIDTH - BLOCK_SIZE or y < BLOCK_SIZE or y >= HEIGHT - BLOCK_SIZE:
                result = game_over(high_score_manager, profile)
                video_playing = True
                return result


            # Improved screen wrapping with exact grid alignment
            if game.direction == "RIGHT" and x >= WIDTH:
                x = 0
                # Move the entire snake to the left side when head wraps
                for i in range(len(game.snake)):
                    game.snake[i] = ((game.snake[i][0] - WIDTH) % WIDTH, game.snake[i][1])
            elif game.direction == "LEFT" and x < 0:
                x = WIDTH - BLOCK_SIZE
                # Move the entire snake to the right side when head wraps
                for i in range(len(game.snake)):
                    game.snake[i] = ((game.snake[i][0] + WIDTH) % WIDTH, game.snake[i][1])
            elif game.direction == "DOWN" and y >= HEIGHT:
                y = 0
                # Move the entire snake to the top when head wraps
                for i in range(len(game.snake)):
                    game.snake[i] = (game.snake[i][0], (game.snake[i][1] - HEIGHT) % HEIGHT)
            elif game.direction == "UP" and y < 0:
                y = HEIGHT - BLOCK_SIZE
                # Move the entire snake to the bottom when head wraps
                for i in range(len(game.snake)):
                    game.snake[i] = (game.snake[i][0], (game.snake[i][1] + HEIGHT) % HEIGHT)
            
            new_head = (x, y)
            
            # Check collisions
            if new_head in game.snake[1:] or new_head in game.obstacles:
                result = game_over(high_score_manager, profile)
                # Resume video playback before returning
                video_playing = True
                return result
            
            # Add new head
            game.snake.insert(0, new_head)
            
            # Check rat collisions
            if game.check_regular_rat_collision(new_head):
                game.score += 10
                growth += 1
                game.rats_eaten += 1
                game.regular_rat_position = game.get_random_position()
                
                # Spawn big rat after 5 regular rats
                if game.rats_eaten >= 5:
                    game.rats_eaten = 0
                    game.big_rat_position = game.get_random_position(size=2)
                    game.big_rat_timer = 5.0
                    game.big_rat_active = True
            
            if game.check_big_rat_collision(new_head):
                # Calculate points based on remaining time
                if game.big_rat_timer >= 4: points = 50
                elif game.big_rat_timer >= 3: points = 40
                elif game.big_rat_timer >= 2: points = 20
                else: points = 10
                
                game.score += points
                game.big_rat_active = False
                game.big_rat_position = None
            
            # Remove tail if not growing
            if growth > 0:
                growth -= 1
            elif len(game.snake) > 2:
                game.snake.pop()
        
        # Drawing
        draw_sky_gradient_background()
        draw_grid()
        draw_border()
        draw_reference_points()
        show_game_ui()
        draw_obstacles()
        draw_snake()
        draw_rat()
        draw_big_rat()
        pygame.display.update()
        clock.tick(60)

def main():
    global current_speed_index, current_difficulty_index, game, user_type, username, video_playing
    global current_head_index, current_body_index, current_tail_index, current_skin_index
    
    # Initialize pygame
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT), pygame.FULLSCREEN)
    pygame.display.set_caption("Snake Game")

    # Show intro video
    show_intro_logo()
    
    # Show authentication screen
    user_type, username = show_authentication_screen(screen, WIDTH, HEIGHT)
    if user_type is None:  # User pressed escape
        pygame.quit()
        sys.exit()
        
    print(f"Logged in as {username} via {user_type}")

    # Initialize game objects
    game = GameState()
    high_score_manager = HighScoreManager()
    profile = UserProfile(username, user_type) if username else None

    # Initialize menu state variables
    in_menu = True  # Start in menu
    in_settings = False  # Not in settings initially
    current_skin_index = 0  # Start with first skin

    running = True
    while running:  # MAIN GAME LOOP
        if in_menu:
            # Ensure video is playing in menu
            video_playing = True
            buttons, profile_button = draw_menu(in_settings, profile)
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                    running = False
                if event.type == pygame.MOUSEBUTTONDOWN:
                    pos = event.pos
                    
                    # Handle profile button click
                    if profile_button and profile_button.collidepoint(pos):
                        while True:
                            action = show_profile_popup(profile)
                            if action == "close" or action == "back":
                                break
                            elif action == "logout":
                                if show_logout_confirmation():
                                    # Reset game state
                                    user_type = None
                                    username = None
                                    profile = None
                                    # Return to auth screen
                                    user_type, username = show_authentication_screen(screen, WIDTH, HEIGHT)
                                    if user_type is None:
                                        running = False
                                    else:
                                        profile = UserProfile(username, user_type)
                                    break
                            elif action == "change_avatar":
                                avatar_action = show_avatar_selection(profile)
                                if avatar_action == "close" or avatar_action == "back":
                                    break
                                elif avatar_action == "random":
                                    profile.set_avatar(random.randint(0, len(DEFAULT_AVATARS)-1))
                                    profile.save()
                                elif isinstance(avatar_action, int):
                                    profile.set_avatar(avatar_action)
                                    profile.save()
                    
                    # In the main game loop where settings buttons are handled:
                    if in_settings:
                        if buttons[0].is_clicked(pos):  # Speed
                            if show_speed_selection():
                            # Speed was applied
                                pass
                        elif buttons[1].is_clicked(pos):  # Difficulty
                            current_difficulty_index = (current_difficulty_index + 1) % len(difficulty_levels)
                            game.setup_obstacles()
                        elif buttons[2].is_clicked(pos):  # Back
                            in_settings = False
                    else:
                        if buttons[0].is_clicked(pos):  # Play
                            game.reset()
                            in_menu = False
                        elif buttons[1].is_clicked(pos):  # Skin
                            skin_action = None
                            while True:
                                skin_action = show_skin_selection()
                                if skin_action == "prev":
                                    current_skin_index = (current_skin_index - 1) % len(SNAKE_SKINS)
                                elif skin_action == "next":
                                    current_skin_index = (current_skin_index + 1) % len(SNAKE_SKINS)
                                elif skin_action == "apply":
                                    break
                                elif skin_action == "back":
                                    break
                        elif buttons[2].is_clicked(pos):  # Shape
                            while True:
                                action = show_shape_selection()
                                if action == "prev":
                                    if current_tab == 0:  # Head
                                        current_head_index = (current_head_index - 1) % len(SNAKE_PARTS["heads"])
                                    elif current_tab == 1:  # Body
                                        current_body_index = (current_body_index - 1) % len(SNAKE_PARTS["bodies"])
                                    else:  # Tail
                                        current_tail_index = (current_tail_index - 1) % len(SNAKE_PARTS["tails"])
                                elif action == "next":
                                    if current_tab == 0:
                                        current_head_index = (current_head_index + 1) % len(SNAKE_PARTS["heads"])
                                    elif current_tab == 1:
                                        current_body_index = (current_body_index + 1) % len(SNAKE_PARTS["bodies"])
                                    else:
                                        current_tail_index = (current_tail_index + 1) % len(SNAKE_PARTS["tails"])
                                elif action == "apply":
                                    # Convert to integers before setting
                                    game.set_parts(
                                        current_head_index, 
                                        current_body_index, 
                                        current_tail_index
                                    )
                                    break
                                elif action == "back":
                                    break
                                elif action == "tab_change":
                                    continue
                        elif buttons[3].is_clicked(pos):  # High Scores
                            while True:
                                refresh = show_high_scores(high_score_manager)
                                if not refresh:
                                    break
                        elif buttons[4].is_clicked(pos):  # Multiplayer
                            video_playing = True
                            show_multiplayer_menu()
                        elif buttons[5].is_clicked(pos):  # Settings
                            in_settings = True
                            if buttons[0].is_clicked(pos):  # Speed button in settings
                                show_speed_selection()
                            elif buttons[1].is_clicked(pos):  # Difficulty
                                current_difficulty_index = (current_difficulty_index + 1) % len(difficulty_levels)
                                game.setup_obstacles()
                            elif buttons[2].is_clicked(pos):  # Back
                                    in_settings = False
                        elif buttons[6].is_clicked(pos):  # Exit
                            running = False

            pygame.display.flip()
            clock.tick(60)
        else:
            result = main_game_loop(high_score_manager, profile)
            if result is False:
                running = False
            else:
                in_menu = True
    
    pygame.quit()
    sys.exit()

if __name__ == "__main__":
    main()