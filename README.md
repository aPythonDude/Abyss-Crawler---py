# Abyss-Crawler - py
# Roguelike Dungeon Crawler

"""
╔══════════════════════════════════════════════════════════════════╗
║   A B Y S S   C R A W L E R   —   Roguelike Dungeon Crawler     ║
║                                                                  ║
║   pip install pygame          (only dependency)                  ║
║                                                                  ║
║   WASD / Arrows  → Move           SPACE → Attack (hold=charge)   ║
║   E              → Quick-use item     I → Inventory              ║
║   ESC            → Menu / Close       Q → Drop item in inv       ║
║                                                                  ║
║   Inventory:  ↑↓ select   E/Enter equip/use   Q drop   ESC close ║
╚══════════════════════════════════════════════════════════════════╝
"""
import pygame, sys, math, random, copy
from dataclasses import dataclass
from typing import Optional, List, Tuple, Dict

pygame.init()
try:    pygame.mixer.init(44100, -16, 2, 512); _AUDIO = True
except: _AUDIO = False

# ═══════════════════════════════════════════════════════════════════
#  WINDOW / CONSTANTS
# ═══════════════════════════════════════════════════════════════════
SW, SH  = 1100, 750
TILE    = 40
FPS     = 60
MAP_W   = 48
MAP_H   = 36
VIEW_W  = 22
VIEW_H  = 18

screen = pygame.display.set_mode((SW, SH))
pygame.display.set_caption("ABYSS CRAWLER  ◆  Roguelike")
clock  = pygame.time.Clock()

# ── Palette ───────────────────────────────────────────────────────
C = dict(
    bg      =(8,6,16),    floor  =(28,22,42),  floor2 =(34,28,50),
    wall_t  =(55,45,75),  wall_b =(12,8,22),   door   =(120,85,30),
    ui_bg   =(12,10,22),  ui_bdr =(70,55,100),
    player  =(80,160,255),player2=(40,90,180),
    goblin  =(80,160,60), orc    =(140,90,40),  troll  =(100,130,80),
    shade   =(160,70,200),mage   =(60,100,200), golem  =(130,110,90),
    boss1   =(200,60,60), boss2  =(180,50,220), boss3  =(220,150,30),
    hit     =(255,80,80), heal   =(80,255,120), xp     =(200,180,50),
    gold    =(255,210,40),potion =(220,50,90),  sword  =(180,180,220),
    scroll  =(230,200,120),ring  =(220,160,40), staff  =(120,80,200),
    white   =(255,255,255),gray  =(160,150,180),yellow =(255,220,50),
    red     =(255,60,60), green  =(60,220,80),  cyan   =(60,210,240),
    orange  =(255,145,30),purple =(200,80,255), pink   =(255,100,180),
    dark    =(40,35,55),  better =(60,220,100), worse  =(220,60,60),
    same    =(180,180,180),
)

# ── Fonts ─────────────────────────────────────────────────────────
Fb = pygame.font.SysFont("consolas", 26, bold=True)
Fm = pygame.font.SysFont("consolas", 20, bold=True)
Fs = pygame.font.SysFont("consolas", 16)
Ft = pygame.font.SysFont("consolas", 14)
Fg = pygame.font.SysFont("consolas", 72, bold=True)

def draw_text(surf, text, font, color, x, y, center=False, shadow=True):
    if shadow:
        sr = font.render(str(text), True, (0,0,0))
        surf.blit(sr, sr.get_rect(center=(x+1,y+1)) if center else (x+1,y+1))
    r = font.render(str(text), True, color)
    surf.blit(r, r.get_rect(center=(x,y)) if center else (x,y))

# ═══════════════════════════════════════════════════════════════════
#  AUDIO — procedural synth (numpy optional)
# ═══════════════════════════════════════════════════════════════════
_sounds = {}
if _AUDIO:
    try:
        import numpy as np
        SR = 44100
        def _synth(freq, dur, vol=0.3, wave='sin', decay=1.0):
            n = int(SR*dur); t = np.linspace(0,dur,n,False)
            w = (np.sin(2*np.pi*freq*t)     if wave=='sin' else
                 np.sign(np.sin(2*np.pi*freq*t)) if wave=='sq'  else
                 2*(t*freq-np.floor(t*freq+.5))   if wave=='saw' else
                 np.random.uniform(-1,1,n))
            env = np.exp(-decay*t/dur*5)
            arr = np.clip(w*env*vol*32767,-32767,32767).astype(np.int16)
            return pygame.sndarray.make_sound(np.column_stack([arr,arr]))
        _sounds = dict(
            hit    =_synth(180,.12,.25,'noise',2), swing=_synth(600,.08,.15,'sq',3),
            pickup =_synth(880,.15,.20,'sin',2),   stairs=_synth(440,.30,.25,'sin',1),
            level  =_synth(660,.50,.30,'sin',.5),  boss  =_synth(55,.80,.40,'sq',.5),
            charge =_synth(300,.20,.20,'saw',2),
        )
    except: _AUDIO = False

def play(s):
    if _AUDIO and s in _sounds: _sounds[s].play()

# ═══════════════════════════════════════════════════════════════════
#  SCREEN SHAKE
# ═══════════════════════════════════════════════════════════════════
_shake=[0,0]
def shake(mag,f=8): _shake[0]=max(_shake[0],f); _shake[1]=max(_shake[1],mag)
def get_shake():
    if _shake[0]>0:
        _shake[0]-=1
        o=(random.randint(-_shake[1],_shake[1]),random.randint(-_shake[1],_shake[1]))
        if _shake[0]==0: _shake[1]=0
        return o
    return 0,0

# ═══════════════════════════════════════════════════════════════════
#  PARTICLES
# ═══════════════════════════════════════════════════════════════════
@dataclass
class Particle:
    x:float; y:float; vx:float; vy:float
    color:tuple; life:int; max_life:int; size:float
    gravity:float=0.0; text:str=""

    def update(self):
        self.x+=self.vx; self.y+=self.vy; self.vy+=self.gravity
        self.vx*=0.92;   self.vy*=0.95;  self.life-=1; return self.life>0

    def draw(self, surf, ox, oy):
        a=self.life/self.max_life
        if self.text:
            c=tuple(int(v*a) for v in self.color)
            r=Fs.render(self.text,True,c)
            surf.blit(r,r.get_rect(center=(int(self.x-ox),int(self.y-oy))))
        else:
            c=tuple(int(v*a) for v in self.color)
            sz=max(1,int(self.size*a))
            pygame.draw.circle(surf,c,(int(self.x-ox),int(self.y-oy)),sz)

particles:List[Particle]=[]

def emit(x,y,color,n=12,speed=3,size=4,life=30,gravity=0):
    for _ in range(n):
        a=random.uniform(0,6.28); v=random.uniform(.3,speed)
        particles.append(Particle(x,y,math.cos(a)*v,math.sin(a)*v,
            color,random.randint(life//2,life),life,random.uniform(1,size),gravity))

def blood(x,y,n=18):  emit(x,y,C['hit'],n,4,5,40,.08)
def sparkle(x,y,c,n=8): emit(x,y,c,n,3,3,25)

def float_text(x,y,text,color):
    particles.append(Particle(x,y,0,-1.2,color,55,55,0,0,text))

# ═══════════════════════════════════════════════════════════════════
#  ITEMS  (balanced stats — small base, modest floor scaling)
# ═══════════════════════════════════════════════════════════════════
@dataclass
class Item:
    name:str; kind:str; color:tuple; icon:str
    atk:int=0; dfn:int=0; spd:int=0; mag:int=0
    hp_restore:int=0; max_hp_bonus:int=0
    effect:str=""; value:int=10; tier:int=1; desc:str=""

# tier = min floor where this can appear
ALL_ITEMS = [
    # ── Weapons ────────────────────────────────────────────────────
    Item("Iron Sword",    "weapon",C['sword'],  "W",atk=8,          value=15,tier=1,desc="Reliable blade"),
    Item("Silver Dagger", "weapon",C['cyan'],   "W",atk=5,  spd=2,  value=18,tier=1,desc="Fast & sharp"),
    Item("Orc Cleaver",   "weapon",C['orc'],    "W",atk=13,         value=30,tier=2,desc="Heavy blow"),
    Item("Mage Staff",    "weapon",C['staff'],  "W",atk=9,  mag=5,  value=35,tier=3,desc="Channels magic"),
    Item("Shadow Blade",  "weapon",C['shade'],  "W",atk=12, spd=2,  value=45,tier=4,desc="Strikes from dark"),
    Item("Holy Avenger",  "weapon",C['yellow'], "W",atk=17, mag=6,  value=70,tier=6,desc="Blessed weapon"),
    Item("Void Reaper",   "weapon",C['purple'],  "W",atk=22, mag=8,  value=90,tier=8,desc="Devours souls"),
    # ── Armor ──────────────────────────────────────────────────────
    Item("Leather Vest",  "armor", C['orc'],    "A",dfn=3,          value=12,tier=1,desc="Light protection"),
    Item("Chain Mail",    "armor", C['gray'],   "A",dfn=7,          value=28,tier=2,desc="Metal rings"),
    Item("Plate Armor",   "armor", C['gray'],   "A",dfn=13,spd=-1,  value=50,tier=4,desc="Heavy but tough"),
    Item("Shadow Cloak",  "armor", C['shade'],  "A",dfn=6,  spd=3,  value=45,tier=4,desc="Dark leather"),
    Item("Dragon Scale",  "armor", C['boss3'],  "A",dfn=18, spd=1,  value=80,tier=7,desc="Dragonhide"),
    # ── Potions ────────────────────────────────────────────────────
    Item("Health Potion", "potion",C['potion'], "P",hp_restore=40,  value=15,tier=1,desc="Restores 40 HP"),
    Item("Greater Potion","potion",C['pink'],   "P",hp_restore=80,  value=28,tier=3,desc="Restores 80 HP"),
    Item("Elixir",        "potion",C['gold'],   "P",hp_restore=150, value=55,tier=5,desc="Full restoration"),
    Item("Strength Brew", "potion",C['orange'], "P",atk=3,          value=22,tier=2,desc="+3 ATK permanent"),
    Item("Iron Skin",     "potion",C['golem'],  "P",dfn=3,          value=22,tier=2,desc="+3 DEF permanent"),
    # ── Rings ──────────────────────────────────────────────────────
    Item("Ring of Speed", "ring",  C['ring'],   "R",spd=3,          value=35,tier=2,desc="+3 SPD"),
    Item("Ring of Power", "ring",  C['ring'],   "R",atk=5,          value=40,tier=3,desc="+5 ATK"),
    Item("Amulet of Life","ring",  C['heal'],   "R",max_hp_bonus=30, value=50,tier=3,desc="+30 Max HP"),
    Item("Arcane Band",   "ring",  C['purple'],  "R",atk=4, mag=5,   value=60,tier=5,desc="+4 ATK +5 MAG"),
    # ── Scrolls ────────────────────────────────────────────────────
    Item("Scroll of Fire","scroll",C['scroll'], "S",effect="fireball",value=18,tier=1,desc="Blast nearby foes"),
    Item("Scroll of Time","scroll",C['cyan'],   "S",effect="slow",    value=22,tier=2,desc="Slow all enemies"),
    Item("Scroll of Warp","scroll",C['purple'],  "S",effect="teleport",value=14,tier=1,desc="Random teleport"),
]

def random_item(depth:int) -> Item:
    """Pick an item whose tier ≤ depth, apply mild scaling."""
    pool = [i for i in ALL_ITEMS if i.tier <= depth]
    if not pool: pool = ALL_ITEMS[:]
    # Weight toward higher-tier items as depth increases
    weights = [1 + (i.tier / max(1, depth)) for i in pool]
    itm = random.choices(pool, weights=weights, k=1)[0]
    i2  = copy.copy(itm)
    # Gentle scaling: +8% per floor above tier, capped
    bonus = 1 + min(0.6, (depth - i2.tier) * 0.08)
    if i2.atk:         i2.atk         = int(i2.atk         * bonus)
    if i2.dfn:         i2.dfn         = int(i2.dfn         * bonus)
    if i2.hp_restore:  i2.hp_restore  = int(i2.hp_restore  * bonus)
    return i2

# ═══════════════════════════════════════════════════════════════════
#  DUNGEON GENERATION  (BSP)
# ═══════════════════════════════════════════════════════════════════
TILE_EMPTY=0; TILE_FLOOR=1; TILE_WALL=2; TILE_DOOR=3
TILE_STAIR=4; TILE_CHEST=5

@dataclass
class Room:
    x:int; y:int; w:int; h:int
    def center(self): return self.x+self.w//2, self.y+self.h//2
    def contains(self,tx,ty): return self.x<=tx<self.x+self.w and self.y<=ty<self.y+self.h
    def random_floor(self):
        return (random.randint(self.x+1,max(self.x+1,self.x+self.w-2)),
                random.randint(self.y+1,max(self.y+1,self.y+self.h-2)))

class Dungeon:
    def __init__(self, depth=1):
        self.depth=depth
        self.map=[[TILE_EMPTY]*MAP_W for _ in range(MAP_H)]
        self.rooms:List[Room]=[]
        self.items:Dict[Tuple[int,int],Item]={}
        self.visible=set(); self.explored=set()
        self._generate()

    def _carve(self,r:Room):
        for y in range(r.y,r.y+r.h):
            for x in range(r.x,r.x+r.w):
                if 0<x<MAP_W-1 and 0<y<MAP_H-1: self.map[y][x]=TILE_FLOOR

    def _corridor(self,x1,y1,x2,y2):
        if random.random()<.5:
            for x in range(min(x1,x2),max(x1,x2)+1):
                if 0<x<MAP_W-1 and 0<y1<MAP_H-1: self.map[y1][x]=TILE_FLOOR
            for y in range(min(y1,y2),max(y1,y2)+1):
                if 0<x2<MAP_W-1 and 0<y<MAP_H-1: self.map[y][x2]=TILE_FLOOR
        else:
            for y in range(min(y1,y2),max(y1,y2)+1):
                if 0<x1<MAP_W-1 and 0<y<MAP_H-1: self.map[y][x1]=TILE_FLOOR
            for x in range(min(x1,x2),max(x1,x2)+1):
                if 0<x<MAP_W-1 and 0<y2<MAP_H-1: self.map[y2][x]=TILE_FLOOR

    def _generate(self):
        pending=[(2,2,MAP_W-3,MAP_H-3)]; splits=[]
        while pending:
            rx,ry,rw,rh=pending.pop()
            if rw<8 or rh<8: splits.append((rx,ry,rw,rh)); continue
            if rw>rh and random.random()<.6:
                lo,hi=rw//3,2*rw//3; mid=random.randint(lo,max(lo,hi))
                pending+=[(rx,ry,mid,rh),(rx+mid,ry,max(4,rw-mid),rh)]
            else:
                lo,hi=rh//3,2*rh//3; mid=random.randint(lo,max(lo,hi))
                pending+=[(rx,ry,rw,mid),(rx,ry+mid,rw,max(4,rh-mid))]

        for rx,ry,rw,rh in splits:
            mw=max(4,min(rw-2,12)); mh=max(4,min(rh-2,10))
            pw=random.randint(4,mw); ph=random.randint(4,mh)
            sx2=max(0,rw-pw-1); sy2=max(0,rh-ph-1)
            px=rx+(random.randint(1,sx2) if sx2>0 else 1)
            py=ry+(random.randint(1,sy2) if sy2>0 else 1)
            room=Room(px,py,pw,ph); self._carve(room); self.rooms.append(room)

        random.shuffle(self.rooms)
        for i in range(len(self.rooms)-1):
            x1,y1=self.rooms[i].center(); x2,y2=self.rooms[i+1].center()
            self._corridor(x1,y1,x2,y2)

        for y in range(MAP_H):
            for x in range(MAP_W):
                if self.map[y][x]==TILE_EMPTY:
                    for dy in (-1,0,1):
                        for dx in (-1,0,1):
                            nx2,ny2=x+dx,y+dy
                            if 0<=nx2<MAP_W and 0<=ny2<MAP_H and self.map[ny2][nx2]==TILE_FLOOR:
                                self.map[y][x]=TILE_WALL; break

        sx,sy=self.rooms[-1].center()
        self.map[sy][sx]=TILE_STAIR; self.stair_pos=(sx,sy)

        for room in random.sample(self.rooms[1:],min(3,len(self.rooms)-1)):
            cx,cy=room.random_floor()
            self.map[cy][cx]=TILE_CHEST; self.items[(cx,cy)]=random_item(self.depth)

        item_count=max(3,self.depth+2)
        for _ in range(item_count):
            r=random.choice(self.rooms); ix,iy=r.random_floor()
            if self.map[iy][ix]==TILE_FLOOR: self.items[(ix,iy)]=random_item(self.depth)

        # Extra healing guaranteed on floor 4+ so player isn't starved
        if self.depth>=4:
            for _ in range(2):
                r=random.choice(self.rooms); ix,iy=r.random_floor()
                if self.map[iy][ix]==TILE_FLOOR:
                    pool=[i for i in ALL_ITEMS if i.kind=='potion']
                    self.items[(ix,iy)]=copy.copy(random.choice(pool))

    def compute_fov(self,px,py,radius=9):
        self.visible.clear(); self.visible.add((px,py))
        for angle in range(0,360):
            rad=math.radians(angle); dx,dy=math.cos(rad),math.sin(rad)
            x,y=float(px)+.5,float(py)+.5
            for _ in range(radius):
                ix,iy=int(x),int(y)
                if not(0<=ix<MAP_W and 0<=iy<MAP_H): break
                self.visible.add((ix,iy)); self.explored.add((ix,iy))
                if self.map[iy][ix]==TILE_WALL: break
                x+=dx; y+=dy

    def is_walkable(self,tx,ty):
        if not(0<=tx<MAP_W and 0<=ty<MAP_H): return False
        return self.map[ty][tx] in (TILE_FLOOR,TILE_DOOR,TILE_STAIR,TILE_CHEST)

    def tile_color(self,tx,ty,lit):
        t=self.map[ty][tx]; shade=1.0 if lit else 0.28
        def dim(c): return tuple(int(v*shade) for v in c)
        if t==TILE_WALL:
            v=(tx*7+ty*13)%5
            return dim((C['wall_t'][0]+v*3,C['wall_t'][1]+v*2,C['wall_t'][2]+v*4))
        if t==TILE_FLOOR:
            v=(tx*3+ty*11)%4
            return dim((C['floor'][0]+v*4,C['floor'][1]+v*3,C['floor'][2]+v*5))
        if t==TILE_STAIR: return dim(C['yellow'])
        if t==TILE_CHEST: return dim(C['gold'])
        if t==TILE_DOOR:  return dim(C['door'])
        return dim(C['bg'])

# ═══════════════════════════════════════════════════════════════════
#  ENTITY BASE
# ═══════════════════════════════════════════════════════════════════
class Entity:
    def __init__(self,tx,ty,color,color2,radius=14):
        self.tx,self.ty=tx,ty
        self.px,self.py=tx*TILE+TILE//2,ty*TILE+TILE//2
        self.color,self.color2=color,color2
        self.radius=radius; self.alive=True
        self.move_t=1.0; self.prev_px=self.px; self.prev_py=self.py
        self.hit_flash=0

    def lerp_pos(self, speed=0.22):
        if self.move_t<1.0:
            self.move_t=min(1.0,self.move_t+speed)
            t=self._ease(self.move_t)
            self.px=self.prev_px+(self.tx*TILE+TILE//2-self.prev_px)*t
            self.py=self.prev_py+(self.ty*TILE+TILE//2-self.prev_py)*t

    def _ease(self,t): return t*t*(3-2*t)

    def move_to(self,tx,ty):
        self.prev_px,self.prev_py=self.px,self.py
        self.tx,self.ty=tx,ty; self.move_t=0.0

    def snap_to_tile(self):
        self.px=self.tx*TILE+TILE//2; self.py=self.ty*TILE+TILE//2; self.move_t=1.0

    def draw_shadow(self,surf,sx,sy):
        shadow_surf=pygame.Surface((self.radius*2,8),pygame.SRCALPHA)
        pygame.draw.ellipse(shadow_surf,(0,0,0,80),(0,0,self.radius*2,8))
        surf.blit(shadow_surf,(sx-self.radius,sy+self.radius-4))

# ═══════════════════════════════════════════════════════════════════
#  PLAYER
# ═══════════════════════════════════════════════════════════════════
class Player(Entity):
    def __init__(self,tx,ty):
        super().__init__(tx,ty,C['player'],C['player2'],15)
        self.hp=100; self.max_hp=100
        self.base_atk=12; self.base_dfn=5; self.base_spd=4
        self.xp=0; self.xp_next=80; self.lvl=1
        self.gold=0; self.floor=1
        self.inventory:List[Item]=[]
        self.weapon:Optional[Item]=None
        self.armor:Optional[Item]=None
        self.ring:Optional[Item]=None
        self.charge=0; self.max_charge=60
        self.atk_cd=0; self.atk_anim=0
        self.kills=0; self.steps=0; self.items_found=0
        self.log:List[Tuple[str,tuple]]=[]

    # ── derived stats ──────────────────────────────────────────────
    def get_atk(self):
        return self.base_atk+(self.weapon.atk if self.weapon else 0)+(self.ring.atk if self.ring else 0)
    def get_dfn(self):
        return self.base_dfn+(self.armor.dfn if self.armor else 0)+(self.ring.dfn if self.ring else 0)
    def get_spd(self):
        s=self.base_spd+(self.weapon.spd if self.weapon else 0)+(self.armor.spd if self.armor else 0)+(self.ring.spd if self.ring else 0)
        return max(1,s)
    def get_mag(self):
        return (self.weapon.mag if self.weapon else 0)+(self.ring.mag if self.ring else 0)

    def log_msg(self,msg,color=None):
        self.log.insert(0,(msg,color or C['gray']))
        if len(self.log)>14: self.log.pop()

    def gain_xp(self,amount):
        self.xp+=amount
        if self.xp>=self.xp_next:
            self.xp-=self.xp_next; self.xp_next=int(self.xp_next*1.55)
            self.lvl+=1
            self.max_hp+=12; self.hp=min(self.hp+25,self.max_hp)
            self.base_atk+=2; self.base_dfn+=1
            self.log_msg(f"LEVEL UP! Lv.{self.lvl}!",C['yellow']); play('level')
            return True
        return False

    def heal(self,amt):
        old=self.hp; self.hp=min(self.max_hp,self.hp+amt); return self.hp-old

    def take_damage(self,raw):
        # Softened: defence reduces damage but never below 1, variance ±2
        dmg=max(1, raw - self.get_dfn() + random.randint(-2,2))
        self.hp=max(0,self.hp-dmg); self.hit_flash=10; shake(3,5)
        return dmg

    # ── equip — NEVER auto-swap without player choice ─────────────
    def equip(self,item):
        slot_attr={'weapon':'weapon','armor':'armor','ring':'ring'}
        attr=slot_attr.get(item.kind)
        if attr is None: return
        current=getattr(self,attr)
        if current: self.inventory.append(current)     # send old one back to bag
        setattr(self,attr,item)
        self.log_msg(f"Equipped {item.name}",C['cyan'])
        sparkle(self.px,self.py,item.color,10)

    def use_item(self,item,dungeon,enemies):
        if item.kind=='potion':
            if item.hp_restore:
                h=self.heal(item.hp_restore)
                self.log_msg(f"Healed {h} HP!",C['heal'])
                emit(self.px,self.py,C['heal'],14,3,4,35)
            if item.atk:  self.base_atk+=item.atk;  self.log_msg(f"+{item.atk} ATK perm!",C['orange'])
            if item.dfn:  self.base_dfn+=item.dfn;  self.log_msg(f"+{item.dfn} DEF perm!",C['orange'])
            if item.max_hp_bonus:
                self.max_hp+=item.max_hp_bonus; self.hp+=item.max_hp_bonus
                self.log_msg(f"Max HP +{item.max_hp_bonus}!",C['cyan'])
            return True
        elif item.kind=='scroll': return self._scroll(item,dungeon,enemies)
        return False

    def _scroll(self,item,dungeon,enemies):
        if item.effect=='fireball':
            for e in enemies:
                if abs(e.tx-self.tx)<=5 and abs(e.ty-self.ty)<=5 and e.alive:
                    dmg=40+self.get_mag(); e.hp-=dmg
                    blood(e.px,e.py,20); emit(e.px,e.py,C['orange'],18,5,6,40)
                    shake(5,8)
                    if e.hp<=0: e.alive=False; self.kills+=1; self.gain_xp(e.xp_reward)
            self.log_msg("Fireball! Enemies scorched!",C['orange']); return True
        elif item.effect=='slow':
            for e in enemies: e.slow_cd=200
            self.log_msg("Time slows for your enemies!",C['cyan']); return True
        elif item.effect=='teleport':
            r=random.choice(dungeon.rooms); tx2,ty2=r.random_floor()
            self.move_to(tx2,ty2); self.snap_to_tile()
            emit(self.px,self.py,C['purple'],20,4,5,40)
            self.log_msg("You vanish and reappear!",C['purple']); return True
        return False

    def draw(self,surf,ox,oy):
        sx,sy=int(self.px)-ox,int(self.py)-oy
        self.draw_shadow(surf,sx,sy)
        if self.charge>10:
            alpha=int(130*self.charge/self.max_charge)
            g=pygame.Surface((70,70),pygame.SRCALPHA)
            pygame.draw.circle(g,(*C['yellow'],alpha),(35,35),35)
            surf.blit(g,(sx-35,sy-35))
        if self.hit_flash>0:
            g=pygame.Surface((52,52),pygame.SRCALPHA)
            pygame.draw.circle(g,(255,60,60,90),(26,26),26)
            surf.blit(g,(sx-26,sy-26)); self.hit_flash-=1
        pts=[(sx,sy-17),(sx-13,sy-2),(sx-16,sy+14),(sx,sy+10),(sx+16,sy+14),(sx+13,sy-2)]
        pygame.draw.polygon(surf,self.color,pts)
        pygame.draw.polygon(surf,self.color2,pts,2)
        pygame.draw.circle(surf,self.color2,(sx,sy-19),10)
        pygame.draw.circle(surf,C['cyan'],(sx,sy-19),6)
        if self.atk_anim>0:
            a=math.radians(self.atk_anim*14)
            ex,ey=sx+math.cos(a)*24,sy+math.sin(a)*24
            pygame.draw.line(surf,C['yellow'],(sx,sy),(int(ex),int(ey)),4)
            pygame.draw.circle(surf,C['white'],(int(ex),int(ey)),5)
            self.atk_anim-=1

# ═══════════════════════════════════════════════════════════════════
#  ENEMY BASE  (balanced — atk/hp scale gently)
# ═══════════════════════════════════════════════════════════════════
class Enemy(Entity):
    def __init__(self,tx,ty,name,color,color2,hp,atk,dfn,spd,xp,gold,radius=13):
        super().__init__(tx,ty,color,color2,radius)
        self.name=name
        self.hp=self.max_hp=hp; self.atk=atk; self.dfn=dfn; self.spd=spd
        self.xp_reward=xp; self.gold_reward=gold
        self.move_cd=0; self.atk_cd=0; self.slow_cd=0; self.path=[]

    def _path_to(self,tx,ty,dungeon):
        path=[]; cx,cy=self.tx,self.ty
        for _ in range(14):
            if cx==tx and cy==ty: break
            opts=[]
            for ddx,ddy in [(-1,0),(1,0),(0,-1),(0,1)]:
                nx2,ny2=cx+ddx,cy+ddy
                if dungeon.is_walkable(nx2,ny2):
                    dist=abs(nx2-tx)+abs(ny2-ty)
                    opts.append((dist,nx2,ny2))
            if not opts: break
            opts.sort(); _,cx,cy=opts[0]; path.append((cx,cy))
        self.path=path

    def tick(self,dungeon,player,all_enemies):
        if not self.alive: return
        self.lerp_pos(0.26)
        cd=2 if self.slow_cd>0 else 1
        if self.slow_cd>0: self.slow_cd-=1
        self.move_cd=max(0,self.move_cd-cd)
        self.atk_cd=max(0,self.atk_cd-cd)
        if abs(self.tx-player.tx)>VIEW_W or abs(self.ty-player.ty)>VIEW_H: return
        dist=abs(self.tx-player.tx)+abs(self.ty-player.ty)
        if dist==1 and self.atk_cd==0:
            self.atk_cd=max(1,42-self.spd*3)
            raw=self.atk+random.randint(-2,3)
            dmg=player.take_damage(raw)
            blood(player.px,player.py,8); float_text(player.px,player.py-12,f"-{dmg}",C['red'])
            play('hit'); return
        if self.move_cd==0:
            self.move_cd=max(1,58-self.spd*4)
            self._path_to(player.tx,player.ty,dungeon)
            if self.path:
                nx2,ny2=self.path[0]
                occupied=any(e.tx==nx2 and e.ty==ny2 for e in all_enemies if e is not self and e.alive)
                if not occupied: self.move_to(nx2,ny2)

    def draw(self,surf,ox,oy):
        sx,sy=int(self.px)-ox,int(self.py)-oy
        self.draw_shadow(surf,sx,sy)
        if self.hit_flash>0:
            g=pygame.Surface((52,52),pygame.SRCALPHA)
            pygame.draw.circle(g,(255,100,100,90),(26,26),26)
            surf.blit(g,(sx-26,sy-26)); self.hit_flash-=1
        self._draw_body(surf,sx,sy)
        bw=self.radius*2+6; bh=5; by=sy-self.radius-12
        pygame.draw.rect(surf,(60,0,0),(sx-bw//2,by,bw,bh))
        fw=int(bw*max(0,self.hp)/self.max_hp)
        col=(C['green'] if self.hp/self.max_hp>.5 else C['yellow'] if self.hp/self.max_hp>.25 else C['red'])
        if fw>0: pygame.draw.rect(surf,col,(sx-bw//2,by,fw,bh))
        if self.slow_cd>0: pygame.draw.circle(surf,C['cyan'],(sx,sy-self.radius-16),3)

    def _draw_body(self,surf,sx,sy):
        pygame.draw.circle(surf,self.color,(sx,sy),self.radius)
        pygame.draw.circle(surf,self.color2,(sx,sy),self.radius,2)

# ── ENEMY TYPES  (balanced: hp/atk scale ×1.10 per floor, not ×depth*10) ──
def _scale(base, depth, per_floor=0.10):
    """Gentle multiplicative scale. +10% per floor above 1, hard-capped at ×2.2."""
    return int(base * min(2.2, 1 + (depth-1)*per_floor))

class Goblin(Enemy):
    def __init__(self,tx,ty,depth=1):
        super().__init__(tx,ty,"Goblin",C['goblin'],(40,100,30),
            _scale(28,depth,.10), _scale(6,depth,.08), 1,
            _scale(5,depth,.05), _scale(20,depth,.12), _scale(3,depth,.10))
    def _draw_body(self,surf,sx,sy):
        pts=[(sx,sy-13),(sx-10,sy+5),(sx,sy+10),(sx+10,sy+5)]
        pygame.draw.polygon(surf,self.color,pts)
        pygame.draw.polygon(surf,self.color2,pts,2)
        pygame.draw.circle(surf,(180,220,80),(sx,sy-13),7)
        pygame.draw.circle(surf,(0,0,0),(sx-2,sy-14),2)
        pygame.draw.circle(surf,(0,0,0),(sx+2,sy-14),2)

class Orc(Enemy):
    def __init__(self,tx,ty,depth=1):
        super().__init__(tx,ty,"Orc",C['orc'],(100,60,20),
            _scale(65,depth,.10), _scale(11,depth,.09), _scale(4,depth,.08),
            _scale(3,depth,.04), _scale(45,depth,.12), _scale(6,depth,.10),16)
    def _draw_body(self,surf,sx,sy):
        pygame.draw.ellipse(surf,self.color,(sx-14,sy-12,28,26))
        pygame.draw.ellipse(surf,self.color2,(sx-14,sy-12,28,26),2)
        pygame.draw.circle(surf,(160,110,50),(sx,sy-14),10)
        pygame.draw.line(surf,(200,180,140),(sx-4,sy-7),(sx-7,sy-2),3)
        pygame.draw.line(surf,(200,180,140),(sx+4,sy-7),(sx+7,sy-2),3)

class Troll(Enemy):
    def __init__(self,tx,ty,depth=1):
        super().__init__(tx,ty,"Troll",C['troll'],(60,90,50),
            _scale(120,depth,.10), _scale(15,depth,.09), _scale(7,depth,.08),
            2, _scale(75,depth,.12), _scale(10,depth,.10),20)
    def _draw_body(self,surf,sx,sy):
        pygame.draw.ellipse(surf,self.color,(sx-18,sy-10,36,30))
        pygame.draw.ellipse(surf,self.color2,(sx-18,sy-10,36,30),2)
        pygame.draw.circle(surf,(80,110,70),(sx,sy-16),14)
        pygame.draw.circle(surf,self.color2,(sx,sy-16),14,2)
        pygame.draw.circle(surf,(220,160,120),(sx-2,sy-16),8)

class Shade(Enemy):
    def __init__(self,tx,ty,depth=1):
        super().__init__(tx,ty,"Shade",C['shade'],(100,30,150),
            _scale(50,depth,.10), _scale(9,depth,.09), _scale(2,depth,.06),
            _scale(6,depth,.05), _scale(55,depth,.12), _scale(5,depth,.10))
    def _draw_body(self,surf,sx,sy):
        t=pygame.time.get_ticks()/400; r=int(11+2*math.sin(t))
        g=pygame.Surface((r*4,r*4),pygame.SRCALPHA)
        pygame.draw.circle(g,(*self.color,130),(r*2,r*2),r*2)
        surf.blit(g,(sx-r*2,sy-r*2))
        pygame.draw.circle(surf,self.color,(sx,sy),r)
        pygame.draw.circle(surf,(220,160,255),(sx,sy),r//2)

class Mage(Enemy):
    def __init__(self,tx,ty,depth=1):
        super().__init__(tx,ty,"Dark Mage",C['mage'],(30,60,150),
            _scale(40,depth,.10), _scale(13,depth,.10), 2,
            _scale(4,depth,.04), _scale(65,depth,.12), _scale(8,depth,.10))
        self.spell_cd=0
    def _draw_body(self,surf,sx,sy):
        pts=[(sx,sy-16),(sx-11,sy+12),(sx+11,sy+12)]
        pygame.draw.polygon(surf,self.color,pts); pygame.draw.polygon(surf,self.color2,pts,2)
        pygame.draw.circle(surf,(100,140,220),(sx,sy-16),9)
        pygame.draw.line(surf,C['scroll'],(sx-13,sy+12),(sx-19,sy-8),3)
        pygame.draw.circle(surf,C['cyan'],(sx-19,sy-8),5)
    def tick(self,dungeon,player,all_enemies):
        if not self.alive: return
        self.lerp_pos(0.26)
        cd=2 if self.slow_cd>0 else 1
        if self.slow_cd>0: self.slow_cd-=1
        self.move_cd=max(0,self.move_cd-cd); self.atk_cd=max(0,self.atk_cd-cd)
        self.spell_cd=max(0,self.spell_cd-cd)
        dist=abs(self.tx-player.tx)+abs(self.ty-player.ty)
        if dist>6 and self.move_cd==0:
            self.move_cd=max(1,62-self.spd*4)
            self._path_to(player.tx,player.ty,dungeon)
            if self.path:
                nx2,ny2=self.path[0]
                occupied=any(e.tx==nx2 and e.ty==ny2 for e in all_enemies if e is not self and e.alive)
                if not occupied: self.move_to(nx2,ny2)
        elif dist<=6 and self.spell_cd==0:
            self.spell_cd=100
            dmg=player.take_damage(self.atk+random.randint(0,4))
            emit(player.px,player.py,C['mage'],12,4,4,35)
            float_text(player.px,player.py-12,f"-{dmg}",C['purple']); play('hit')

class StoneGolem(Enemy):
    def __init__(self,tx,ty,depth=1):
        super().__init__(tx,ty,"Stone Golem",C['golem'],(90,75,60),
            _scale(180,depth,.10), _scale(16,depth,.09), _scale(10,depth,.08),
            1, _scale(110,depth,.12), _scale(14,depth,.10),22)
    def _draw_body(self,surf,sx,sy):
        pygame.draw.rect(surf,self.color,(sx-16,sy-18,32,36))
        pygame.draw.rect(surf,self.color2,(sx-16,sy-18,32,36),3)
        pygame.draw.rect(surf,(110,90,70),(sx-10,sy-14,20,28))
        pygame.draw.circle(surf,C['red'],(sx-6,sy-6),4); pygame.draw.circle(surf,C['red'],(sx+6,sy-6),4)
        pygame.draw.circle(surf,(255,150,50),(sx-6,sy-6),2); pygame.draw.circle(surf,(255,150,50),(sx+6,sy-6),2)

# ── BOSSES  (hp/atk balanced — floors 3,6,9 with fixed totals) ───
class Boss(Enemy):
    def __init__(self,tx,ty,name,color,color2,hp,atk,dfn,spd,xp,gold,radius=30):
        super().__init__(tx,ty,name,color,color2,hp,atk,dfn,spd,xp,gold,radius)
        self.phase=1; self.special_cd=0; self.is_boss=True

class SkullLord(Boss):
    def __init__(self,tx,ty,depth):
        d=depth
        super().__init__(tx,ty,"Skull Lord",C['boss1'],(150,30,30),
            350+d*20, 16+d*2, 7+d, 4, 400+d*30, 40+d*8, 28)
        self.summon_cd=0
    def _draw_body(self,surf,sx,sy):
        t=pygame.time.get_ticks()/300
        pygame.draw.circle(surf,self.color,(sx,sy),int(26+2*math.sin(t)))
        pygame.draw.circle(surf,(230,80,80),(sx,sy),14)
        for i in range(3):
            a=t+i*(math.pi*2/3); bx2,by2=sx+math.cos(a)*26,sy+math.sin(a)*26
            pygame.draw.circle(surf,(200,200,200),(int(bx2),int(by2)),7)
            pygame.draw.circle(surf,(60,0,0),(int(bx2)-2,int(by2)-2),2)
            pygame.draw.circle(surf,(60,0,0),(int(bx2)+2,int(by2)-2),2)
        pygame.draw.circle(surf,C['yellow'],(sx-8,sy-5),5)
        pygame.draw.circle(surf,C['yellow'],(sx+8,sy-5),5)
        pygame.draw.circle(surf,C['red'],(sx,sy+5),4)
    def tick(self,dungeon,player,all_enemies):
        super().tick(dungeon,player,all_enemies)
        if not self.alive: return
        if self.hp<self.max_hp*.5 and self.phase==1:
            self.phase=2; self.atk+=6; self.spd+=1
            player.log_msg("Skull Lord enrages!",C['red']); shake(10,15)
            emit(self.px,self.py,C['red'],25,6,7,55)

class VoidWeaver(Boss):
    def __init__(self,tx,ty,depth):
        d=depth
        super().__init__(tx,ty,"Void Weaver",C['boss2'],(120,30,180),
            480+d*28, 15+d*2, 5+d, 5, 550+d*35, 55+d*10, 30)
    def _draw_body(self,surf,sx,sy):
        t=pygame.time.get_ticks()/200; r=int(28+4*math.sin(t*1.5))
        g=pygame.Surface((r*4,r*4),pygame.SRCALPHA)
        pygame.draw.circle(g,(*C['boss2'],55),(r*2,r*2),r*2)
        surf.blit(g,(sx-r*2,sy-r*2))
        for i in range(6):
            a=t*1.2+i*(math.pi/3); x2,y2=sx+math.cos(a)*(r+8),sy+math.sin(a)*(r+8)
            pygame.draw.line(surf,C['boss2'],(sx,sy),(int(x2),int(y2)),4)
            pygame.draw.circle(surf,(220,100,255),(int(x2),int(y2)),6)
        pygame.draw.circle(surf,self.color,(sx,sy),r)
        pygame.draw.circle(surf,C['white'],(sx,sy),12); pygame.draw.circle(surf,C['boss2'],(sx,sy),8)

class DragonKing(Boss):
    def __init__(self,tx,ty,depth):
        d=depth
        super().__init__(tx,ty,"Dragon King",C['boss3'],(160,100,10),
            650+d*38, 21+d*3, 10+d*2, 3, 900+d*45, 70+d*12, 34)
    def _draw_body(self,surf,sx,sy):
        t=pygame.time.get_ticks()/250
        for side in (-1,1):
            pts=[(sx,sy),(sx+side*20,sy-25),(sx+side*45,sy-10),(sx+side*35,sy+20)]
            pygame.draw.polygon(surf,(180,120,20),pts)
            pygame.draw.polygon(surf,C['boss3'],pts,2)
        pygame.draw.ellipse(surf,self.color,(sx-26,sy-20,52,42))
        pygame.draw.ellipse(surf,(200,160,30),(sx-26,sy-20,52,42),2)
        pygame.draw.circle(surf,self.color,(sx,sy-26),18)
        pygame.draw.circle(surf,C['red'],(sx-7,sy-28),6); pygame.draw.circle(surf,C['red'],(sx+7,sy-28),6)
        pygame.draw.circle(surf,C['yellow'],(sx-7,sy-28),3); pygame.draw.circle(surf,C['yellow'],(sx+7,sy-28),3)
        if int(t*3)%5<2:
            for _ in range(3):
                fx=sx+random.randint(-8,8); fy=sy-40
                pygame.draw.circle(surf,random.choice([C['orange'],C['yellow'],C['red']]),
                                   (fx,fy),random.randint(4,9))

# ═══════════════════════════════════════════════════════════════════
#  SPAWN LOGIC  (enemy count capped; fewer mobs on boss floors)
# ═══════════════════════════════════════════════════════════════════
BOSS_FLOORS={3:SkullLord, 6:VoidWeaver, 9:DragonKing}

def spawn_enemies(dungeon,player):
    d=dungeon.depth; enemies=[]
    if   d==1: types=[Goblin]
    elif d==2: types=[Goblin,Orc]
    elif d==3: types=[Goblin,Orc,Shade]          # boss floor — lighter trash
    elif d<=5: types=[Orc,Shade,Mage]
    elif d==6: types=[Orc,Shade,Mage]            # boss floor
    elif d<=8: types=[Shade,Mage,Troll,StoneGolem]
    else:      types=[Mage,Troll,StoneGolem,Shade]

    is_boss_floor = d in BOSS_FLOORS
    # Cap enemy count — boss floors get fewer trash mobs
    max_trash = 3+d if not is_boss_floor else max(2, d-1)

    for room in dungeon.rooms[1:]:
        if room.contains(player.tx,player.ty): continue
        n=random.randint(1,min(2,1+d//3))
        for _ in range(n):
            if len(enemies)>=max_trash: break
            tx2,ty2=room.random_floor()
            enemies.append(random.choice(types)(tx2,ty2,d))

    if is_boss_floor:
        bx,by=dungeon.rooms[-1].center()
        boss=BOSS_FLOORS[d](bx,by,d)
        enemies.append(boss)
        player.log_msg(f"A {boss.name} lurks ahead!",C['red'])
        play('boss'); shake(12,20)

    return enemies

# ═══════════════════════════════════════════════════════════════════
#  FOG-OF-WAR LIGHT SURFACE
# ═══════════════════════════════════════════════════════════════════
def make_light_surf(radius=9):
    size=radius*TILE*2+1; s=pygame.Surface((size,size),pygame.SRCALPHA)
    cx=cy=size//2
    for r in range(radius*TILE,0,-1):
        frac=r/(radius*TILE)
        pygame.draw.circle(s,(0,0,0,int(220*frac)),(cx,cy),r)
    return s

LIGHT_SURF=make_light_surf(9)

# ═══════════════════════════════════════════════════════════════════
#  UI HELPERS
# ═══════════════════════════════════════════════════════════════════
def draw_bar(surf,x,y,w,h,val,max_val,fg,bg=(50,0,0),border=None):
    border=border or C['ui_bdr']
    pygame.draw.rect(surf,bg,(x,y,w,h))
    fw=int(w*max(0,val)/max(1,max_val))
    if fw>0: pygame.draw.rect(surf,fg,(x,y,fw,h))
    pygame.draw.rect(surf,border,(x,y,w,h),1)

def draw_panel(surf,x,y,w,h,alpha=220):
    s=pygame.Surface((w,h),pygame.SRCALPHA)
    s.fill((*C['ui_bg'],alpha))
    pygame.draw.rect(s,C['ui_bdr'],(0,0,w,h),2)
    for cx2,cy2 in [(0,0),(w-8,0),(0,h-8),(w-8,h-8)]:
        pygame.draw.rect(s,C['purple'],(cx2,cy2,8,8))
    surf.blit(s,(x,y))

def stat_delta_color(delta):
    if delta>0:  return C['better']
    if delta<0:  return C['worse']
    return C['same']

def fmt_delta(val, sign=True):
    if val==0: return "—"
    return ("+"+str(val)) if val>0 else str(val)

# ═══════════════════════════════════════════════════════════════════
#  INVENTORY SCREEN  (full compare panel, clear equipped state)
# ═══════════════════════════════════════════════════════════════════
class InventoryScreen:
    def __init__(self): self.open=False; self.cursor=0

    def toggle(self): self.open=not self.open; self.cursor=0

    def handle(self,key,player,dungeon,enemies):
        inv=player.inventory
        if key in (pygame.K_i,pygame.K_ESCAPE): self.open=False; return
        if key==pygame.K_UP:   self.cursor=max(0,self.cursor-1)
        if key==pygame.K_DOWN and inv: self.cursor=min(len(inv)-1,self.cursor+1)
        if key in (pygame.K_e,pygame.K_RETURN) and inv:
            itm=inv[self.cursor]
            if itm.kind in ("weapon","armor","ring"):
                player.equip(itm); inv.remove(itm)
                self.cursor=min(self.cursor,max(0,len(inv)-1))
                play('pickup')
            elif itm.kind in ("potion","scroll"):
                used=player.use_item(itm,dungeon,enemies)
                if used: inv.remove(itm); self.cursor=min(self.cursor,max(0,len(inv)-1))
                play('pickup')
        # U = unequip highlighted slot (cycle weapon -> armor -> ring)
        if key==pygame.K_u:
            for attr in ('weapon','armor','ring'):
                if getattr(player,attr) is not None:
                    old=getattr(player,attr)
                    setattr(player,attr,None)
                    player.inventory.append(old)
                    player.log_msg(f"Unequipped {old.name}",C['gray'])
                    break
        if key==pygame.K_q and inv:
            inv.pop(self.cursor); self.cursor=min(self.cursor,max(0,len(inv)-1))

    def draw(self,surf,player):
        # ── Background overlay ────────────────────────────────────
        overlay=pygame.Surface((SW,SH),pygame.SRCALPHA)
        overlay.fill((0,0,0,160)); surf.blit(overlay,(0,0))

        # ── Main list panel ───────────────────────────────────────
        PX,PY,PW,PH=60,55,540,640
        draw_panel(surf,PX,PY,PW,PH)
        draw_text(surf,"── INVENTORY ──",Fb,C['cyan'],PX+PW//2,PY+16,center=True)

        inv=player.inventory
        if not inv:
            draw_text(surf,"(empty — pick up items!)",Fm,C['gray'],PX+PW//2,PY+PH//2,center=True)
        else:
            visible=13
            start=max(0,self.cursor-visible+1) if self.cursor>=visible else 0
            for i,itm in enumerate(inv[start:start+visible]):
                idx=start+i; y=PY+52+i*35; sel=idx==self.cursor
                if sel:
                    pygame.draw.rect(surf,(60,48,100),(PX+8,y-2,PW-16,32))
                    pygame.draw.rect(surf,C['purple'],(PX+8,y-2,PW-16,32),1)

                # Color-coded kind tag
                kind_col={'weapon':C['cyan'],'armor':C['orange'],'ring':C['gold'],
                          'potion':C['pink'],'scroll':C['purple']}.get(itm.kind,C['gray'])
                pygame.draw.circle(surf,itm.color,(PX+24,y+13),8)
                tag=itm.kind[0].upper()
                draw_text(surf,tag,Ft,C['white'],PX+24,y+8,center=True,shadow=False)

                name_col=C['yellow'] if sel else C['white']
                draw_text(surf,itm.name,Fs,name_col,PX+40,y+2)

                # Quick stats right-aligned
                stats_parts=[]
                if itm.atk:         stats_parts.append((f"ATK+{itm.atk}",C['orange']))
                if itm.dfn:         stats_parts.append((f"DEF+{itm.dfn}",C['cyan']))
                if itm.spd:         stats_parts.append((f"SPD{'+' if itm.spd>0 else ''}{itm.spd}",C['green']))
                if itm.hp_restore:  stats_parts.append((f"HP+{itm.hp_restore}",C['heal']))
                if itm.max_hp_bonus:stats_parts.append((f"MHP+{itm.max_hp_bonus}",C['cyan']))
                if itm.effect:      stats_parts.append((itm.effect.upper(),C['yellow']))
                rx=PX+PW-12
                for s_text,s_col in reversed(stats_parts):
                    r=Ft.render(s_text,True,s_col); rx-=r.get_width()+6
                    surf.blit(r,(rx,y+6))

            # Scroll hint
            if len(inv)>visible:
                draw_text(surf,f"↑↓ scroll ({self.cursor+1}/{len(inv)})",
                          Ft,C['gray'],PX+PW//2,PY+PH-20,center=True)

        draw_text(surf,"[E] Equip/Use    [U] Unequip    [Q] Drop    [I/ESC] Close",
                  Ft,C['gray'],PX+PW//2,PY+PH-6,center=True)

        # ── Equipped panel (always visible) ───────────────────────
        EX,EY,EW,EH=PX+PW+16,PY,340,230
        draw_panel(surf,EX,EY,EW,EH)
        draw_text(surf,"── EQUIPPED ──",Fm,C['cyan'],EX+EW//2,EY+16,center=True)
        draw_text(surf,"[U] Unequip first slot",Ft,C['gray'],EX+EW//2,EY+EH-16,center=True)
        slot_defs=[("Weapon",player.weapon,C['cyan']),
                   ("Armor", player.armor, C['orange']),
                   ("Ring",  player.ring,  C['gold'])]
        for i,(lbl,slot,slot_col) in enumerate(slot_defs):
            y=EY+44+i*54
            # Slot background — bright border if equipped, dim if empty
            bdr_col=slot_col if slot else C['dark']
            pygame.draw.rect(surf,(25,20,42),(EX+10,y,EW-20,46))
            pygame.draw.rect(surf,bdr_col,(EX+10,y,EW-20,46),2)
            draw_text(surf,lbl+":",Ft,slot_col,EX+18,y+4)
            if slot:
                pygame.draw.circle(surf,slot.color,(EX+26,y+30),7)
                draw_text(surf,slot.name,Fs,C['white'],EX+38,y+21)
                sp=[]
                if slot.atk: sp.append(f"ATK+{slot.atk}")
                if slot.dfn: sp.append(f"DEF+{slot.dfn}")
                if slot.spd and slot.spd!=0: sp.append(f"SPD{'+' if slot.spd>0 else ''}{slot.spd}")
                if slot.mag: sp.append(f"MAG+{slot.mag}")
                draw_text(surf,"  ".join(sp) if sp else slot.desc[:22],Ft,C['gray'],EX+38,y+36)
            else:
                draw_text(surf,"─ nothing equipped ─",Ft,C['dark'],EX+18,y+22)

        # ── Comparison panel (selected item vs equipped) ──────────
        if inv:
            sel_itm=inv[self.cursor]
            CX,CY,CW,CH=PX+PW+16,EY+EH+12,340,270
            draw_panel(surf,CX,CY,CW,CH)
            draw_text(surf,"── COMPARE ──",Fm,C['cyan'],CX+CW//2,CY+16,center=True)
            pygame.draw.circle(surf,sel_itm.color,(CX+22,CY+44),9)
            draw_text(surf,sel_itm.name,Fm,C['yellow'],CX+38,CY+36)
            draw_text(surf,f"[{sel_itm.kind}]  {sel_itm.desc}",Ft,C['gray'],CX+16,CY+60)

            # Find currently equipped in same slot
            cur=None
            if sel_itm.kind=='weapon': cur=player.weapon
            elif sel_itm.kind=='armor': cur=player.armor
            elif sel_itm.kind=='ring':  cur=player.ring

            row=CY+86
            if sel_itm.kind in ("weapon","armor","ring"):
                if cur:
                    draw_text(surf,f"vs  {cur.name}",Ft,C['gray'],CX+16,row); row+=20
                else:
                    draw_text(surf,"(nothing equipped)",Ft,C['dark'],CX+16,row); row+=20

                comparisons=[]
                if sel_itm.kind in ("weapon","ring"):
                    comparisons.append(("ATK", sel_itm.atk, cur.atk if cur else 0))
                if sel_itm.kind in ("armor","ring"):
                    comparisons.append(("DEF", sel_itm.dfn, cur.dfn if cur else 0))
                if sel_itm.kind in ("weapon","armor","ring"):
                    comparisons.append(("SPD", sel_itm.spd, cur.spd if cur else 0))
                if sel_itm.kind in ("weapon","ring"):
                    comparisons.append(("MAG", sel_itm.mag, cur.mag if cur else 0))

                for stat,new_val,old_val in comparisons:
                    delta=new_val-old_val
                    dc=stat_delta_color(delta)
                    draw_text(surf,f"{stat}:",Ft,C['gray'],CX+16,row)
                    draw_text(surf,str(old_val),Fs,C['gray'],CX+75,row)
                    draw_text(surf,"→",Ft,C['gray'],CX+105,row)
                    draw_text(surf,str(new_val),Fs,dc,CX+128,row)
                    delta_str=fmt_delta(delta)
                    draw_text(surf,f"({delta_str})",Ft,dc,CX+178,row)
                    row+=22
                if row==CY+106: draw_text(surf,"No stat changes",Ft,C['dark'],CX+16,row)
            elif sel_itm.kind=='potion':
                if sel_itm.hp_restore: draw_text(surf,f"Heal {sel_itm.hp_restore} HP",Fm,C['heal'],CX+16,row); row+=26
                if sel_itm.atk:       draw_text(surf,f"+{sel_itm.atk} ATK (permanent)",Fm,C['orange'],CX+16,row); row+=26
                if sel_itm.dfn:       draw_text(surf,f"+{sel_itm.dfn} DEF (permanent)",Fm,C['cyan'],CX+16,row); row+=26
            elif sel_itm.kind=='scroll':
                draw_text(surf,f"Effect: {sel_itm.effect.upper()}",Fm,C['yellow'],CX+16,row); row+=26
                draw_text(surf,sel_itm.desc,Fs,C['gray'],CX+16,row)

            # Action hint
            if sel_itm.kind in ("weapon","armor","ring"):
                hint_col=C['green'] if (not cur or (sel_itm.atk+sel_itm.dfn)>=(cur.atk+cur.dfn)) else C['gray']
                draw_text(surf,"[E] Equip this item",Fm,hint_col,CX+CW//2,CY+CH-18,center=True)
            else:
                draw_text(surf,"[E] Use this item",Fm,C['green'],CX+CW//2,CY+CH-18,center=True)

        # ── Player total stats (always shown) ─────────────────────
        SX,SY,SSVW,SSH=PX+PW+16,EY+EH+12+280+12,340,110
        draw_panel(surf,SX,SY,SSVW,SSH)
        draw_text(surf,"── TOTAL STATS ──",Fm,C['cyan'],SX+SSVW//2,SY+14,center=True)
        for i,(lbl,val,col) in enumerate([
            ("ATK",player.get_atk(),C['orange']),
            ("DEF",player.get_dfn(),C['cyan']),
            ("SPD",player.get_spd(),C['green']),
            ("MAG",player.get_mag(),C['purple']),
        ]):
            col2=i%2; row2=SY+38+(i//2)*32; rx2=SX+8 if col2==0 else SX+SSVW//2+8
            draw_text(surf,f"{lbl}: {val}",Fm,col,rx2,row2)

# ═══════════════════════════════════════════════════════════════════
#  SMOOTH INPUT  (analog repeat, not stutter)
# ═══════════════════════════════════════════════════════════════════
class InputHandler:
    """Tracks held keys, outputs move events at configurable repeat rate."""
    INITIAL_DELAY = 180   # ms before repeat starts
    REPEAT_RATE   = 75    # ms between repeats

    def __init__(self):
        self._held:Dict[int,int]={} # key -> time first pressed (ms)
        self._last:Dict[int,int]={} # key -> time last repeated

    def update(self, events):
        now=pygame.time.get_ticks()
        for ev in events:
            if ev.type==pygame.KEYDOWN:
                self._held[ev.key]=now; self._last[ev.key]=now
            elif ev.type==pygame.KEYUP:
                self._held.pop(ev.key,None); self._last.pop(ev.key,None)

    def pressed(self, key):
        """True on the first frame a key is pressed."""
        return key in self._held and self._held[key]==self._last.get(key)

    def repeat(self, key):
        """True on the first press AND repeatedly after INITIAL_DELAY."""
        if key not in self._held: return False
        now=pygame.time.get_ticks()
        first=self._held[key]
        if now-first<self.INITIAL_DELAY: return now-first<16  # only on first frame
        last=self._last[key]
        if now-last>=self.REPEAT_RATE:
            self._last[key]=now; return True
        return False

    def any_dir(self):
        dirs=[(pygame.K_LEFT,-1,0),(pygame.K_a,-1,0),
              (pygame.K_RIGHT,1,0),(pygame.K_d,1,0),
              (pygame.K_UP,0,-1),(pygame.K_w,0,-1),
              (pygame.K_DOWN,0,1),(pygame.K_s,0,1)]
        for k,dx,dy in dirs:
            if self.repeat(k): return dx,dy
        return 0,0

# ═══════════════════════════════════════════════════════════════════
#  MESSAGE LOG
# ═══════════════════════════════════════════════════════════════════
def draw_log(surf,player,x,y,w,h):
    draw_panel(surf,x,y,w,h,200)
    draw_text(surf,"── Messages ──",Ft,C['ui_bdr'],x+w//2,y+8,center=True)
    for i,(msg,col) in enumerate(player.log[:8]):
        a=max(0.3,1.0-i*0.11)
        c=tuple(int(v*a) for v in col)
        draw_text(surf,msg,Ft,c,x+8,y+26+i*17,shadow=False)

# ═══════════════════════════════════════════════════════════════════
#  MAIN GAME CLASS
# ═══════════════════════════════════════════════════════════════════
class Game:
    PANEL_W=270; MAP_W_PX=SW-270; MAP_H_PX=SH

    def __init__(self): self.inp=InputHandler(); self.reset()

    def reset(self):
        self.dungeon=Dungeon(depth=1)
        r0=self.dungeon.rooms[0]; sx,sy=r0.center()
        self.player=Player(sx,sy)
        self.enemies=spawn_enemies(self.dungeon,self.player)
        self.inv_ui=InventoryScreen()
        self.cam_x=float(max(0,sx*TILE-self.MAP_W_PX//2))
        self.cam_y=float(max(0,sy*TILE-self.MAP_H_PX//2))
        self.dungeon.compute_fov(sx,sy); self.state='play'
        self.player.log_msg("You descend into the Abyss…",C['cyan'])
        self.player.log_msg("WASD:Move  SPACE:Attack(hold=charge)",C['gray'])
        particles.clear()

    def next_floor(self):
        d=self.player.floor+1
        op=self.player; self.dungeon=Dungeon(depth=d)
        r0=self.dungeon.rooms[0]; sx,sy=r0.center()
        op.tx,op.ty=sx,sy; op.snap_to_tile(); op.floor=d
        op.hp=min(op.max_hp,op.hp+25)
        op.log_msg(f"Floor {d} — deeper you go…",C['yellow'])
        self.player=op; self.enemies=spawn_enemies(self.dungeon,self.player)
        self.cam_x=float(max(0,sx*TILE-self.MAP_W_PX//2))
        self.cam_y=float(max(0,sy*TILE-self.MAP_H_PX//2))
        self.dungeon.compute_fov(sx,sy); play('stairs'); particles.clear()
        emit(sx*TILE+TILE//2,sy*TILE+TILE//2,C['cyan'],20,4,5,40)

    def _smooth_cam(self):
        pl=self.player
        tx=pl.px-self.MAP_W_PX//2; ty=pl.py-self.MAP_H_PX//2
        max_x=max(0,MAP_W*TILE-self.MAP_W_PX); max_y=max(0,MAP_H*TILE-self.MAP_H_PX)
        self.cam_x+=(max(0,min(float(max_x),tx))-self.cam_x)*0.18
        self.cam_y+=(max(0,min(float(max_y),ty))-self.cam_y)*0.18

    def player_try_move(self,dx,dy):
        pl=self.player; d=self.dungeon; nx,ny=pl.tx+dx,pl.ty+dy
        if not d.is_walkable(nx,ny): return
        for e in self.enemies:
            if e.alive and e.tx==nx and e.ty==ny: return
        pl.move_to(nx,ny); pl.steps+=1

        key=(nx,ny)
        if key in d.items:
            itm=d.items.pop(key)
            pl.inventory.append(itm); pl.items_found+=1; play('pickup')
            pl.log_msg(f"Picked up: {itm.name}",itm.color)
            float_text(pl.px,pl.py-22,itm.name[:14],itm.color)
            emit(pl.px,pl.py,itm.color,10,2,3,25)
            # ONLY auto-equip if the slot is completely empty and it's better
            slot=getattr(pl,itm.kind,None) if itm.kind in ('weapon','armor','ring') else None
            if itm.kind in ('weapon','armor','ring') and slot is None:
                # Slot empty — safe to auto-equip
                pl.equip(itm); pl.inventory.remove(itm)
            # If slot occupied, player must manually choose via inventory

        if d.map[ny][nx]==TILE_STAIR: self.next_floor()

    def player_attack(self):
        pl=self.player
        if pl.atk_cd>0: return
        pl.atk_cd=max(1,24-pl.get_spd()*2)
        charged=pl.charge>=pl.max_charge

        for e in self.enemies:
            if not e.alive: continue
            dist=abs(e.tx-pl.tx)+abs(e.ty-pl.ty)
            if dist<=1 or (charged and dist<=2):
                base=pl.get_atk()+random.randint(-3,5)
                if charged: base=int(base*2.0)
                dmg=max(1,base-e.dfn//2)
                e.hp-=dmg; e.hit_flash=10
                blood(e.px,e.py,14 if charged else 10)
                float_text(e.px,e.py-18,f"-{dmg}",C['yellow'] if charged else C['white'])
                play('swing')
                if e.hp<=0:
                    e.alive=False; pl.kills+=1
                    xp=e.xp_reward; gld=e.gold_reward+random.randint(0,e.gold_reward//2)
                    pl.gold+=gld; pl.gain_xp(xp)
                    pl.log_msg(f"Killed {e.name}! +{xp}XP +{gld}G",C['xp'])
                    emit(e.px,e.py,e.color,28,5,7,50)
                    if getattr(e,'is_boss',False):
                        shake(14,22); play('boss')
                        emit(e.px,e.py,C['gold'],40,6,9,70)
                        pl.log_msg(f"BOSS SLAIN!",C['yellow'])
                    if random.random()<0.20:
                        self.dungeon.items[(e.tx,e.ty)]=random_item(self.dungeon.depth)

        pl.atk_anim=8
        if charged: shake(4,7); emit(pl.px,pl.py,C['yellow'],12,5,4,28); play('charge')
        pl.charge=0

    # ── MAIN LOOP ─────────────────────────────────────────────────
    def run(self):
        while True:
            dt=clock.tick(FPS)
            raw_events=pygame.event.get()
            self.inp.update(raw_events)

            for ev in raw_events:
                if ev.type==pygame.QUIT: pygame.quit(); sys.exit()
                if ev.type==pygame.KEYDOWN:
                    k=ev.key
                    if self.state=='play':
                        if k==pygame.K_ESCAPE:
                            if self.inv_ui.open: self.inv_ui.open=False
                            else: self.state='menu'
                        elif k==pygame.K_i: self.inv_ui.toggle()
                        elif self.inv_ui.open:
                            self.inv_ui.handle(k,self.player,self.dungeon,self.enemies)
                        elif k==pygame.K_e:
                            for itm in self.player.inventory:
                                if itm.kind in ('potion','scroll'):
                                    used=self.player.use_item(itm,self.dungeon,self.enemies)
                                    if used: self.player.inventory.remove(itm); break
                    elif self.state=='menu':
                        if k==pygame.K_RETURN: self.reset(); self.state='play'
                        elif k==pygame.K_ESCAPE: pygame.quit(); sys.exit()
                    elif self.state in ('dead','win'):
                        if k in (pygame.K_RETURN,pygame.K_r): self.reset(); self.state='play'
                        elif k==pygame.K_ESCAPE: pygame.quit(); sys.exit()

            if self.state=='play' and not self.inv_ui.open:
                pl=self.player
                # Smooth directional input
                dx,dy=self.inp.any_dir()
                if dx or dy: self.player_try_move(dx,dy)

                # Charge + attack
                keys=pygame.key.get_pressed()
                if keys[pygame.K_SPACE]:
                    pl.charge=min(pl.max_charge,pl.charge+2)
                else:
                    if pl.charge>0 and pl.atk_cd==0: self.player_attack()
                    pl.charge=0
                if pl.atk_cd>0: pl.atk_cd-=1

                # Lerp player
                pl.lerp_pos(0.32)
                self.dungeon.compute_fov(pl.tx,pl.ty)
                self._smooth_cam()

                # Enemies
                for e in self.enemies: e.tick(self.dungeon,pl,self.enemies)
                self.enemies=[e for e in self.enemies if e.alive]

                # Particles
                particles[:]=[p for p in particles if p.update()]

                if pl.hp<=0: self.state='dead'
                if pl.floor>=10: self.state='win'

            # ── DRAW ─────────────────────────────────────────────
            ox,oy=get_shake()
            screen.fill(C['bg'])

            if self.state=='play':
                self.draw_map(ox,oy); self.draw_entities(ox,oy)
                self.draw_particles(ox,oy); self.draw_hud()
                if self.inv_ui.open: self.inv_ui.draw(screen,self.player)
            elif self.state=='menu':    self.draw_menu()
            elif self.state=='dead':    self.draw_end(won=False)
            elif self.state=='win':     self.draw_end(won=True)

            pygame.display.flip()

    # ── DRAW MAP ──────────────────────────────────────────────────
    def draw_map(self,ox=0,oy=0):
        d=self.dungeon; cam_x=int(self.cam_x); cam_y=int(self.cam_y)
        tw=self.MAP_W_PX//TILE+3; th=self.MAP_H_PX//TILE+3
        tx0=max(0,cam_x//TILE); ty0=max(0,cam_y//TILE)
        tick_ms=pygame.time.get_ticks()

        for ty in range(ty0,min(MAP_H,ty0+th)):
            for tx in range(tx0,min(MAP_W,tx0+tw)):
                sx=tx*TILE-cam_x+ox; sy=ty*TILE-cam_y+oy
                vis=(tx,ty) in d.visible; exp=(tx,ty) in d.explored
                if not exp: continue
                col=d.tile_color(tx,ty,vis)
                t=d.map[ty][tx]
                pygame.draw.rect(screen,col,(sx,sy,TILE,TILE))
                if vis:
                    if t==TILE_WALL:
                        pygame.draw.rect(screen,C['wall_b'],(sx,sy,TILE,3))
                        pygame.draw.rect(screen,(75,60,100),(sx,sy,TILE,1))
                    elif t==TILE_FLOOR:
                        alt=(tx*3+ty*11)%2
                        if alt: pygame.draw.rect(screen,C['floor2'],(sx+1,sy+1,TILE-2,TILE-2))
                    elif t==TILE_STAIR:
                        bob=int(2*math.sin(tick_ms/300+tx+ty))
                        pygame.draw.polygon(screen,C['yellow'],
                            [(sx+5,sy+TILE-5),(sx+TILE//2,sy+7+bob),(sx+TILE-5,sy+TILE-5)])
                        pygame.draw.polygon(screen,C['orange'],
                            [(sx+5,sy+TILE-5),(sx+TILE//2,sy+7+bob),(sx+TILE-5,sy+TILE-5)],2)
                    elif t==TILE_CHEST:
                        pygame.draw.rect(screen,C['gold'],(sx+6,sy+10,TILE-12,TILE-14))
                        pygame.draw.rect(screen,C['yellow'],(sx+6,sy+10,TILE-12,TILE-14),2)
                        pygame.draw.rect(screen,C['orange'],(sx+TILE//2-4,sy+14,8,6))
                if vis and (tx,ty) in d.items:
                    itm=d.items[(tx,ty)]
                    bob=int(3*math.sin(tick_ms/400+tx*1.3+ty*0.9))
                    pygame.draw.circle(screen,itm.color,(sx+TILE//2,sy+TILE//2+bob),9)
                    pygame.draw.circle(screen,C['white'],(sx+TILE//2,sy+TILE//2+bob),9,1)

        lx=int(self.player.px-cam_x-LIGHT_SURF.get_width()//2+ox)
        ly=int(self.player.py-cam_y-LIGHT_SURF.get_height()//2+oy)
        screen.blit(LIGHT_SURF,(lx,ly),special_flags=pygame.BLEND_RGBA_MULT)

    def draw_entities(self,ox=0,oy=0):
        cam_x=int(self.cam_x); cam_y=int(self.cam_y)
        for e in self.enemies:
            if (e.tx,e.ty) in self.dungeon.explored: e.draw(screen,cam_x-ox,cam_y-oy)
        self.player.draw(screen,cam_x-ox,cam_y-oy)

    def draw_particles(self,ox=0,oy=0):
        cam_x=int(self.cam_x); cam_y=int(self.cam_y)
        for p in particles: p.draw(screen,cam_x-ox,cam_y-oy)

    # ── HUD ───────────────────────────────────────────────────────
    def draw_hud(self):
        pl=self.player; px=self.MAP_W_PX; w=self.PANEL_W
        draw_panel(screen,px,0,w,SH)

        # Header
        pygame.draw.circle(screen,C['player'],(px+w//2,46),22)
        pygame.draw.circle(screen,C['player2'],(px+w//2,46),22,3)
        draw_text(screen,"☩",Fm,C['cyan'],px+w//2,40,center=True)
        draw_text(screen,"ABYSS CRAWLER",Ft,C['cyan'],px+w//2,80,center=True)

        y=98
        draw_text(screen,f"Floor {pl.floor}  •  Lv.{pl.lvl}",Ft,C['yellow'],px+w//2,y,center=True)

        # HP
        y=116
        hp_col=C['green'] if pl.hp/pl.max_hp>.5 else C['yellow'] if pl.hp/pl.max_hp>.25 else C['red']
        draw_text(screen,"HP",Ft,C['white'],px+10,y)
        draw_bar(screen,px+36,y,w-46,14,pl.hp,pl.max_hp,hp_col,(60,0,0))
        draw_text(screen,f"{pl.hp}/{pl.max_hp}",Ft,C['white'],px+w//2,y,center=True)

        # XP
        y+=20
        draw_text(screen,"XP",Ft,C['xp'],px+10,y)
        draw_bar(screen,px+36,y,w-46,9,pl.xp,pl.xp_next,C['xp'],(30,25,0))

        # Charge
        y+=16
        if pl.charge>0:
            draw_text(screen,"CH",Ft,C['yellow'],px+10,y)
            draw_bar(screen,px+36,y,w-46,9,pl.charge,pl.max_charge,C['yellow'],(30,30,0))
            y+=14

        # Stats block
        y+=8
        pygame.draw.line(screen,C['ui_bdr'],(px+8,y),(px+w-8,y),1); y+=6
        for lbl,val,col in [("ATK",pl.get_atk(),C['orange']),("DEF",pl.get_dfn(),C['cyan']),
                             ("SPD",pl.get_spd(),C['green']),("GOLD",pl.gold,C['gold'])]:
            draw_text(screen,lbl,Ft,C['gray'],px+12,y)
            draw_text(screen,str(val),Fm,col,px+w-14,y+1,center=False)
            draw_text(screen,str(val),Fm,col,px+w-14,y+1)
            y+=20

        # Equipped — compact
        y+=4
        pygame.draw.line(screen,C['ui_bdr'],(px+8,y),(px+w-8,y),1); y+=6
        draw_text(screen,"── Equipped ──",Ft,C['ui_bdr'],px+w//2,y,center=True); y+=16
        for slot_lbl,slot in [("W",pl.weapon),("A",pl.armor),("R",pl.ring)]:
            draw_text(screen,slot_lbl+":",Ft,C['gray'],px+12,y)
            if slot:
                pygame.draw.circle(screen,slot.color,(px+32,y+8),6)
                draw_text(screen,slot.name[:15],Ft,C['white'],px+44,y)
            else:
                draw_text(screen,"─",Ft,C['dark'],px+44,y)
            y+=20

        # Inventory count + hint
        y+=4
        pygame.draw.line(screen,C['ui_bdr'],(px+8,y),(px+w-8,y),1); y+=6
        ic=len(pl.inventory)
        draw_text(screen,f"Bag: {ic}/20 items",Ft,C['gray'] if ic<18 else C['red'],px+w//2,y,center=True)

        # Controls
        y=SH-138
        pygame.draw.line(screen,C['ui_bdr'],(px+8,y),(px+w-8,y),1); y+=6
        for line,col in [("[WASD] Move",C['gray']),("[SPACE] Swing",C['gray']),
                         ("[Hold SPACE] Charge",C['yellow']),("[E] Quick potion",C['gray']),
                         ("[I] Inventory+Equip",C['cyan'])]:
            draw_text(screen,line,Ft,col,px+10,y,shadow=False); y+=18

        # Kill / step counts
        draw_text(screen,f"Kills:{pl.kills}  Steps:{pl.steps}",Ft,C['gray'],px+10,y)

        # Log
        draw_log(screen,pl,px,SH-270,w,130)
        # Minimap
        self.draw_minimap(px+10,SH-270-120-10,w-20,115)

    def draw_minimap(self,mx,my,mw,mh):
        draw_panel(screen,mx-2,my-2,mw+4,mh+4,160)
        draw_text(screen,"MAP",Ft,C['ui_bdr'],mx+mw//2,my-14,center=True)
        cw=mw/MAP_W; ch=mh/MAP_H; d=self.dungeon
        for ty in range(MAP_H):
            for tx in range(MAP_W):
                if (tx,ty) not in d.explored: continue
                t=d.map[ty][tx]
                if t==TILE_EMPTY: continue
                col=(C['floor'] if t==TILE_FLOOR else C['wall_t'] if t==TILE_WALL
                     else C['yellow'] if t==TILE_STAIR else C['gold'])
                pygame.draw.rect(screen,col,(int(mx+tx*cw),int(my+ty*ch),max(1,int(cw)),max(1,int(ch))))
        px2=int(mx+self.player.tx*cw); py2=int(my+self.player.ty*ch)
        pygame.draw.circle(screen,C['player'],(px2,py2),3)
        for e in self.enemies:
            if e.alive: pygame.draw.circle(screen,C['red'],(int(mx+e.tx*cw),int(my+e.ty*ch)),2)

    def draw_menu(self):
        t=pygame.time.get_ticks()/1000; screen.fill(C['bg'])
        for i in range(0,360,15):
            a=math.radians(i+t*30); r=280+math.sin(t*2+i)*40
            x=SW//2+math.cos(a)*r; y=SH//2+math.sin(a)*r
            cv=int(30+20*math.sin(t+i*.1))
            pygame.draw.circle(screen,(cv,0,cv*2),(int(x),int(y)),2)
        tc=(int(180+60*math.sin(t)),int(80+40*math.cos(t*1.2)),255)
        draw_text(screen,"ABYSS",Fg,tc,SW//2,170,center=True)
        draw_text(screen,"CRAWLER",Fg,(100,200,255),SW//2,248,center=True)
        draw_text(screen,"◆  Roguelike Dungeon Crawler  ◆",Fm,C['purple'],SW//2,318,center=True)
        blink=abs(math.sin(t*2))
        draw_text(screen,"[ PRESS ENTER TO DESCEND ]",Fb,(int(255*blink),int(200*blink),0),SW//2,410,center=True)
        for i,line in enumerate([
            "WASD / Arrows: Move    SPACE: Attack (hold = Charge)",
            "E: Quick-use potion    I: Open Inventory",
            "Inventory: ↑↓ pick   E equip/use   U unequip   Q drop",
        ]):
            draw_text(screen,line,Ft,C['gray'],SW//2,490+i*24,center=True)
        for i,f in enumerate([
            "◈ Procedural dungeons — every run unique",
            "◈ Balanced scaling: enemies grow with you",
            "◈ Smart inventory: compare before you equip",
            "◈ 6 enemy types + 3 Bosses on floors 3, 6, 9",
        ]):
            draw_text(screen,f,Ft,(60,int(160+80*math.sin(t+i*.5)),int(160+80*math.sin(t+i*.5))),
                      SW//2,580+i*22,center=True)

    def draw_end(self,won):
        t=pygame.time.get_ticks()/1000; screen.fill(C['bg']); pl=self.player
        if won:
            c=(int(180+60*math.sin(t)),int(200+40*math.sin(t*1.3)),50)
            draw_text(screen,"VICTORY!",Fg,c,SW//2,130,center=True)
            draw_text(screen,"You conquered the Abyss!",Fb,C['gold'],SW//2,218,center=True)
            emit(random.randint(100,SW-100),SH,C['gold'],2,random.uniform(2,5),4,60,-0.05)
        else:
            draw_text(screen,"YOU DIED",Fg,C['red'],SW//2,130,center=True)
            draw_text(screen,"The Abyss claimed your soul.",Fb,C['gray'],SW//2,218,center=True)
        py2=268; draw_panel(screen,SW//2-210,py2,420,240)
        draw_text(screen,"── RUN STATISTICS ──",Fm,C['cyan'],SW//2,py2+18,center=True)
        for i,(lbl,val,col) in enumerate([
            ("Floor Reached",pl.floor,C['yellow']),("Level",pl.lvl,C['orange']),
            ("Enemies Killed",pl.kills,C['red']),("Gold",pl.gold,C['gold']),
            ("Steps",pl.steps,C['gray']),("Items Found",pl.items_found,C['cyan']),
        ]):
            y=py2+50+i*29
            draw_text(screen,lbl,Fm,C['gray'],SW//2-130,y)
            draw_text(screen,str(val),Fb,col,SW//2+120,y)
        blink=abs(math.sin(t*2))
        draw_text(screen,"[ ENTER ] Play Again     [ ESC ] Quit",
                  Fb,(int(255*blink),int(200*blink),50),SW//2,560,center=True)
        particles[:]=[p for p in particles if p.update()]
        for p in particles: p.draw(screen,0,0)

# ═══════════════════════════════════════════════════════════════════
#  ENTRY POINT
# ═══════════════════════════════════════════════════════════════════
if __name__=="__main__":
    game=Game(); game.state='menu'; game.run()
