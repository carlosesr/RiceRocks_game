# RiceRocks_game
My implementation of the RiceRocks game from my final project of Cousera

# program template for Spaceship
import simplegui
import math
import random

# globals for user interface
WIDTH = 800
HEIGHT = 600
score = 0
lives = 3
time = 0
started = False

class ImageInfo:
    def __init__(self, center, size, radius = 0, lifespan = None, animated = False):
        self.center = center
        self.size = size
        self.radius = radius
        if lifespan:
            self.lifespan = lifespan
        else:
            self.lifespan = float('inf')
        self.animated = animated

    def get_center(self):
        return self.center

    def get_size(self):
        return self.size

    def get_radius(self):
        return self.radius

    def get_lifespan(self):
        return self.lifespan

    def get_animated(self):
        return self.animated
    
# art assets created by Kim Lathrop, may be freely re-used in non-commercial projects, please credit Kim
    
# debris images - debris1_brown.png, debris2_brown.png, debris3_brown.png, debris4_brown.png
#                 debris1_blue.png, debris2_blue.png, debris3_blue.png, debris4_blue.png, debris_blend.png
debris_info = ImageInfo([320, 240], [640, 480])
debris_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/debris1_blue.png")

# nebula images - nebula_brown.png, nebula_blue.png
nebula_info = ImageInfo([400, 300], [800, 600])
nebula_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/nebula_blue.f2014.png")

# splash image
splash_info = ImageInfo([200, 150], [400, 300])
splash_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/splash.png")

# ship image
ship_info = ImageInfo([45, 45], [90, 90], 35)
ship_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/double_ship.png")

# missile image - shot1.png, shot2.png, shot3.png
missile_info = ImageInfo([5,5], [10, 10], 3, 50)
missile_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/shot2.png")

# asteroid images - asteroid_blue.png, asteroid_brown.png, asteroid_blend.png
asteroid_info = ImageInfo([45, 45], [90, 90], 40)
asteroid_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/asteroid_blue.png")

# animated explosion - explosion_orange.png, explosion_blue.png, explosion_blue2.png, explosion_alpha.png
explosion_info = ImageInfo([64, 64], [128, 128], 17, 24, True)
explosion_image = simplegui.load_image("http://commondatastorage.googleapis.com/codeskulptor-assets/lathrop/explosion_alpha.png")

# sound assets purchased from sounddogs.com, please do not redistribute
soundtrack = simplegui.load_sound("http://commondatastorage.googleapis.com/codeskulptor-assets/sounddogs/soundtrack.mp3")
missile_sound = simplegui.load_sound("http://commondatastorage.googleapis.com/codeskulptor-assets/sounddogs/missile.mp3")
missile_sound.set_volume(.5)
ship_thrust_sound = simplegui.load_sound("http://commondatastorage.googleapis.com/codeskulptor-assets/sounddogs/thrust.mp3")
explosion_sound = simplegui.load_sound("http://commondatastorage.googleapis.com/codeskulptor-assets/sounddogs/explosion.mp3")

# alternative upbeat soundtrack by composer and former IIPP student Emiel Stopler
# please do not redistribute without permission from Emiel at http://www.filmcomposer.nl
# soundtrack = simplegui.load_sound("https://storage.googleapis.com/codeskulptor-assets/ricerocks_theme.mp3")

# helper functions to handle transformations
def angle_to_vector(ang):
    return [math.cos(ang), math.sin(ang)]

def dist(p,q):
    return math.sqrt((p[0] - q[0]) ** 2 +(p[1] - q[1]) ** 2)

# helper function to draw sets
def process_sprite_group(a_set, canvas):
    for a_sprite in set(a_set):
        if a_sprite.update(): a_set.remove(a_sprite)
        a_sprite.draw(canvas)

# helper functions for collitions        
def group_collide(group, an_object):
    collide = False
    for element in set(group):
        if element.collide(an_object):
            group.remove(element)
            explosion = Sprite(element.get_position(), [0, 0], 0, 0, 
                               explosion_image, explosion_info, explosion_sound)
            explosion_group.add(explosion)
            collide = True
    return collide

def group_group_collide(group1, group2):
    hits = 0
    for elm in set(group1):
        if group_collide(group2, elm):
            hits += 1
            group1.discard(elm)
    return hits

# Ship class
class Ship:
    def __init__(self, pos, vel, angle, image, info):
        self.pos = [pos[0], pos[1]]
        self.vel = [vel[0], vel[1]]
        self.thrust = False
        self.angle = angle
        self.angle_vel = 0
        self.image = image
        self.image_center = info.get_center()
        self.image_size = info.get_size()
        self.radius = info.get_radius()
        
    def get_position(self):
        return self.pos
    
    def get_radius(self):
        return self.radius
        
    def set_angle_vel(self, v):
        self.angle_vel = v
        
    def switch_thrusters(self, switch):
        self.thrust = switch
        if switch:
            ship_thrust_sound.play()
        else:
            ship_thrust_sound.rewind()
    
    def shoot(self):
        global a_missile
        
        if started:
            a_missile = Sprite([self.pos[0] + self.radius * self.forward_dir[0], self.pos[1] + self.radius * self.forward_dir[1]],
                               [self.vel[0] + 5 * self.forward_dir[0], self.vel[1] + 5 * self.forward_dir[1]],
                                0, 0, missile_image, missile_info, missile_sound)
            missile_group.add(a_missile)
        
    def draw(self,canvas):
        if self.thrust:
            canvas.draw_image(self.image, [self.image_center[0] + self.image_size[0], self.image_center[1]],
                              self.image_size, self.pos, self.image_size, self.angle)
        else:
            canvas.draw_image(self.image, self.image_center, self.image_size, self.pos, self.image_size, self.angle)

    def update(self):
        self.pos[0] = self.pos[0] % WIDTH + self.vel[0]
        self.pos[1] = self.pos[1] % HEIGHT + self.vel[1]
        self.angle += self.angle_vel
        self.vel[0] *= 0.988
        self.vel[1] *= 0.988
        self.forward_dir = angle_to_vector(self.angle)
        if self.thrust:
            self.vel[0] += 0.09 * self.forward_dir[0]
            self.vel[1] += 0.09 * self.forward_dir[1]
    
# Sprite class
class Sprite:
    def __init__(self, pos, vel, ang, ang_vel, image, info, sound = None):
        self.pos = [pos[0],pos[1]]
        self.vel = [vel[0],vel[1]]
        self.angle = ang
        self.angle_vel = ang_vel
        self.image = image
        self.image_center = info.get_center()
        self.image_size = info.get_size()
        self.radius = info.get_radius()
        self.lifespan = info.get_lifespan()
        self.animated = info.get_animated()
        self.age = 0
        if sound:
            sound.rewind()
            sound.play()
            
    def get_position(self):
        return self.pos
    
    def get_radius(self):
        return self.radius
    
    def collide(self, other_object):
        return dist(self.get_position(), other_object.get_position()) <= self.get_radius() + other_object.get_radius()
   
    def draw(self, canvas):
        if self.animated:
            canvas.draw_image(self.image, [self.image_center[0] + self.image_size[0] * self.age, self.image_center[1]],
                              self.image_size, self.pos, self.image_size)
        else:
            canvas.draw_image(self.image, self.image_center, self.image_size, self.pos, self.image_size, self.angle)
    
    def update(self):
        self.pos[0] = self.pos[0] % WIDTH + self.vel[0]
        self.pos[1] = self.pos[1] % HEIGHT + self.vel[1]
        self.angle += self.angle_vel
        self.age += 1
        if self.age <= self.lifespan: return False
        else: return True

def keydown(key):
    if simplegui.KEY_MAP['right'] == key:
        my_ship.set_angle_vel(math.pi / 60)
    elif simplegui.KEY_MAP['left'] == key:
        my_ship.set_angle_vel(-math.pi / 60)
    elif simplegui.KEY_MAP['up'] == key:
        my_ship.switch_thrusters(True)
    elif simplegui.KEY_MAP['space'] == key:
        my_ship.shoot()
        
def keyup(key):
    if simplegui.KEY_MAP['right'] == key or simplegui.KEY_MAP['left'] == key:
        my_ship.set_angle_vel(0)
    elif simplegui.KEY_MAP['up'] == key:
        my_ship.switch_thrusters(False)
        
def click(pos):
    global started

    in_width = (WIDTH - splash_info.get_size()[0]) / 2, (WIDTH + splash_info.get_size()[0]) / 2
    in_height = (HEIGHT - splash_info.get_size()[1]) / 2, (HEIGHT + splash_info.get_size()[1]) / 2
    if in_width[0] <= pos[0] <= in_width[1] and in_height[0] <= pos[1] <= in_height[1]:
        started = True
    
def draw(canvas):
    global time, lives, score, started, rock_group, missile_group, explosion_group
    
    # animate background
    time += 1
    wtime = (time / 4) % WIDTH
    center = debris_info.get_center()
    size = debris_info.get_size()
    canvas.draw_image(nebula_image, nebula_info.get_center(), nebula_info.get_size(), [WIDTH / 2, HEIGHT / 2], [WIDTH, HEIGHT])
    canvas.draw_image(debris_image, center, size, (wtime - WIDTH / 2, HEIGHT / 2), (WIDTH, HEIGHT))
    canvas.draw_image(debris_image, center, size, (wtime + WIDTH / 2, HEIGHT / 2), (WIDTH, HEIGHT))

    # draw ship and sprites
    my_ship.draw(canvas)
    if started:
        process_sprite_group(rock_group, canvas)
        process_sprite_group(missile_group, canvas)
        process_sprite_group(explosion_group, canvas)
    
    # draw splash screen
    if not started:
        canvas.draw_image(splash_image, splash_info.get_center(), splash_info.get_size(),
                         (WIDTH / 2, HEIGHT / 2), splash_info.get_size())
    
    # update ship
    my_ship.update()

    # set lives and score
    canvas.draw_text('Lives', (WIDTH * 0.10, HEIGHT * 0.10), 24, 'White')
    canvas.draw_text(str(lives), (WIDTH * 0.10, HEIGHT * 0.15), 24, 'White')
    canvas.draw_text('Score', (WIDTH * 0.80, HEIGHT * 0.10), 24, 'White')
    canvas.draw_text(str(score), (WIDTH * 0.80, HEIGHT * 0.15), 24, 'White')
    if group_collide(rock_group, my_ship): lives -= 1
    score += group_group_collide(rock_group, missile_group)
    
    # reset the game
    if lives == 0:
        score = 0
        lives = 3
        rock_group = set()
        missile_group = set()
        explosion_group = set()
        soundtrack.rewind()
        started = False
            
# timer handler that spawns a rock    
def rock_spawner():
    if len(rock_group) < 12 and started:
        a_rock = Sprite([random.randint(0, WIDTH), random.randint(0, HEIGHT)],
                        [(random.random() * 0.5 + 0.5) * random.choice([1, -1]), (random.random() * 0.5 + 0.5) * random.choice([1, -1])],
                        0, random.random() * 0.05 + 0.03, asteroid_image, asteroid_info)
        d = dist(my_ship.get_position(), a_rock.get_position())
        if d > 250:
            rock_group.add(a_rock)
        soundtrack.play()
    
# initialize frame
frame = simplegui.create_frame("Asteroids", WIDTH, HEIGHT)

# initialize ship and two sprites
my_ship = Ship([WIDTH / 2, HEIGHT / 2], [0, 0], 0, ship_image, ship_info)
rock_group = set()
missile_group = set()
explosion_group = set()

# register handlers
frame.set_draw_handler(draw)
frame.set_keydown_handler(keydown)
frame.set_keyup_handler(keyup)
frame.set_mouseclick_handler(click)
timer = simplegui.create_timer(1000.0, rock_spawner)

# get things rolling
timer.start()
frame.start()
