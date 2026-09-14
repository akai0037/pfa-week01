import random
import math
import maya.cmds as cmds
# import maya libraries


def _make_shader(name, color):
# def = define
# create a lambert material and shading group with the given RGB color
    """Create a lambert shader + shading group in the given RGB color
    and return the shading group name, ready to be assigned to geo."""
    shader = cmds.shadingNode("lambert", asShader=True, name=name)
    cmds.setAttr(shader + ".color", *color, type="double3")
    sg = cmds.sets(renderable=True, noSurfaceShader=True, empty=True,
                   name=name + "SG")
    cmds.connectAttr(shader + ".outColor", sg + ".surfaceShader", force=True)
    return sg


def _assign_shader(sg, *nodes):
    cmds.sets(list(nodes), edit=True, forceElement=sg)


def _random_color(base=None, spread=0.15):
    """A random RGB color, optionally jittered around a base color."""
    if base is None:
        return (random.uniform(0.1, 0.9),
                random.uniform(0.1, 0.9),
                random.uniform(0.1, 0.9))
    return tuple(min(1.0, max(0.0, c + random.uniform(-spread, spread)))
                 for c in base)


# ---------------------------------------------------------------------------
# seasons -- each palette retints ground/foliage/water and adjusts the 
# odds of a tree being bare (winter/autumn) or in blossom (spring)
# ---------------------------------------------------------------------------

SEASON_ORDER = ["spring", "summer", "autumn", "winter"]

SEASON_PALETTES = {
    "spring": {
        "grass": (0.35, 0.62, 0.28),    
  # (RGB color)
        "foliage": [(0.28, 0.58, 0.22)],
        "blossom_chance": 0.25,
        "blossom_colors": [(0.95, 0.75, 0.85), (0.98, 0.95, 0.8)],
        "bare_chance": 0.0,
   # Spring has a 25% blossom chance and no bare trees.
        "water": (0.2, 0.45, 0.7),
        "bush": (0.28, 0.58, 0.22),
    },
    "summer": {
        "grass": (0.22, 0.5, 0.16),
        "foliage": [(0.16, 0.45, 0.13)],
        "blossom_chance": 0.0,
        "blossom_colors": [],
        "bare_chance": 0.0,
        "water": (0.15, 0.35, 0.65),
        "bush": (0.18, 0.45, 0.15),
    },
    "autumn": {
        "grass": (0.55, 0.45, 0.18),
        "foliage": [(0.75, 0.45, 0.1), (0.8, 0.65, 0.1), (0.6, 0.2, 0.1)],
        "blossom_chance": 0.0,
        "blossom_colors": [],
        "bare_chance": 0.2,
        "water": (0.25, 0.4, 0.5),
        "bush": (0.55, 0.38, 0.15),
    },
    "winter": {
        "grass": (0.88, 0.9, 0.93),
        "foliage": [(0.85, 0.88, 0.92)],
        "blossom_chance": 0.0,
        "blossom_colors": [],
        "bare_chance": 0.6,
        "water": (0.8, 0.87, 0.92),
        "bush": (0.82, 0.85, 0.88),
    },
}


def _resolve_season(season):
    """Normalize a season string, resolving 'random' to an actual season.
    Returns (palette_dict, resolved_season_name)."""
    season_name = (season or "summer").strip().lower()
    if season_name == "random":
        season_name = random.choice(SEASON_ORDER)
    palette = SEASON_PALETTES.get(season_name, SEASON_PALETTES["summer"])
    return palette, season_name

# prevent the objects from overlapping
class _PlacementGrid: 
    """Simple overlap-avoidance helper: remembers (x, z, radius) of
    everything placed so far (including road/plaza footprints) and
    rejects or searches for spots that stay clear of it all."""

# init = initialize
    def __init__(self, width, depth, margin=0.6): 
        self.width = width
        self.depth = depth
        self.margin = margin
        self.placed = []  

# distance check
    def is_free(self, x, z, radius):
        for (px, pz, pr) in self.placed:
            if math.hypot(x - px, z - pz) < (radius + pr + self.margin): 
                return False
        return True

    def register(self, x, z, radius):
        self.placed.append((x, z, radius))

    def find_spot(self, radius, max_tries=40):
        half_w = self.width / 2.0 - self.margin
        half_d = self.depth / 2.0 - self.margin
        for _ in range(max_tries):
            x = random.uniform(-half_w, half_w)
            z = random.uniform(-half_d, half_d)
            if self.is_free(x, z, radius):
                self.register(x, z, radius)
                return x, z
        return None

# calculate a random position beside the roads
def _roadside_spot(roads_info, obj_radius, road_width=2.0): 
    """Pick a point just off to the side of a random road, if any roads
    exist. Returns (x, z) or None."""
    if not roads_info:
        return None
    theta, plaza_radius, boundary_dist = random.choice(roads_info)
    t = random.uniform(0.15, 0.9)
    dist = plaza_radius + t * (boundary_dist - plaza_radius)
    side = random.choice([-1, 1])
    perp = theta + math.pi / 2.0
    offset = road_width / 2.0 + obj_radius + 0.3
    x = dist * math.cos(theta) + side * offset * math.cos(perp)
    z = dist * math.sin(theta) + side * offset * math.sin(perp)
    return x, z


# ---------------------------------------------------------------------------
# individual props -- each returns (list_of_created_nodes, footprint_radius)
# ---------------------------------------------------------------------------

def create_ground(name, width, depth, palette=None):
    palette = palette or SEASON_PALETTES["summer"]
    ground = cmds.polyPlane(name=name + "_ground", width=width, height=depth,
                             sx=10, sy=10)[0]
    grass_color = _random_color(base=palette["grass"], spread=0.05)
    grass_sg = _make_shader(name + "_grassShader", grass_color)
    _assign_shader(grass_sg, ground)
    return [ground], 0.0


def create_road_network(name, count, width, depth, grid,
                         road_width=2.0, plaza_radius=1.5):
# Build a central plaza and several roads extending toward the park boundary
    """Builds a central plaza with `count` straight roads radiating out
    to the park's edge at (roughly) evenly spaced angles. Registers the
    plaza and road footprints in `grid` so other props avoid them.
    Returns (nodes, roads_info) where roads_info is a list of
    (theta_radians, plaza_radius, boundary_dist) for roadside placement."""
    if count <= 0:
        return [], [] 
   # not create road

    parts = []
    roads_info = []

    plaza = cmds.polyCylinder(name=name + "_plaza", radius=plaza_radius,
                               height=0.06, sx=20)[0]
    cmds.move(0, 0.03, 0, plaza)
    parts.append(plaza)
    grid.register(0, 0, plaza_radius)

    angle_step = 360.0 / count
    for i in range(count):
        theta_deg = i * angle_step + random.uniform(-angle_step * 0.25,
                                                      angle_step * 0.25)
        theta = math.radians(theta_deg)
        cos_t, sin_t = math.cos(theta), math.sin(theta)

  # distance from the center to the rectangular park boundary along this direction
        candidates = []
        if abs(cos_t) > 1e-6:
            candidates.append((width / 2.0) / abs(cos_t))
        if abs(sin_t) > 1e-6:
            candidates.append((depth / 2.0) / abs(sin_t))
        boundary_dist = min(candidates) if candidates else max(width, depth) / 2.0

# park too small for this road, skip it
        if boundary_dist <= plaza_radius + 1.0:
            continue  

        length = boundary_dist - plaza_radius
        center_dist = plaza_radius + length / 2.0
        cx, cz = center_dist * cos_t, center_dist * sin_t

        road = cmds.polyCube(name="%s_road%d" % (name, i + 1),
                              w=length, h=0.05, d=road_width)[0]
        cmds.move(cx, 0.03, cz, road)
        cmds.rotate(0, -theta_deg, 0, road)
        parts.append(road)
        roads_info.append((theta, plaza_radius, boundary_dist))

        for t in (0.15, 0.4, 0.65, 0.9):
            dist = plaza_radius + t * length
            grid.register(dist * cos_t, dist * sin_t, road_width / 2.0)

    road_color = _random_color(base=(0.4, 0.38, 0.35), spread=0.04)
    road_sg = _make_shader(name + "_roadShader", road_color)
    _assign_shader(road_sg, *parts)
    return parts, roads_info

# create pond
def create_pond(name, x, z, radius=2.5, palette=None): 
    palette = palette or SEASON_PALETTES["summer"]
    pond = cmds.polyCylinder(name=name + "_pond", radius=radius,
                              height=0.08, sx=20)[0]
    cmds.move(x, 0.04, z, pond)
    water_color = _random_color(base=palette["water"], spread=0.05)
    water_sg = _make_shader(name + "_waterShader", water_color)
    _assign_shader(water_sg, pond)
    return [pond], radius


def create_fountain(name, x, z, palette=None):
    palette = palette or SEASON_PALETTES["summer"]
    tiers = random.choice([2, 3]) 
 # The fountain randomly has 2 or 3 tiers.
    parts = []
    r = random.uniform(1.0, 1.6)
    base_radius = r
    y = 0.0
    for i in range(tiers):
        h = random.uniform(0.3, 0.5)
        tier = cmds.polyCylinder(name="%s_tier%d" % (name, i + 1),
                                  radius=r, height=h, sx=16)[0]
        cmds.move(x, y + h / 2.0, z, tier)
        parts.append(tier)
        y += h
        r *= 0.55

    water_r = max(r, 0.2)
    water = cmds.polySphere(name=name + "_water", radius=water_r,
                             sx=10, sy=10)[0]
    cmds.move(x, y + water_r * 0.6, z, water)
    parts.append(water)

    stone_color = _random_color(base=(0.75, 0.73, 0.68), spread=0.05)
    water_color = _random_color(base=palette["water"], spread=0.05)
    stone_sg = _make_shader(name + "_stoneShader", stone_color)
    water_sg = _make_shader(name + "_fountainWaterShader", water_color)
    _assign_shader(stone_sg, *parts[:-1])
    _assign_shader(water_sg, water)

    return parts, base_radius


def create_tree(name, x, z, palette=None):
    palette = palette or SEASON_PALETTES["summer"]

    trunk_h = random.uniform(1.2, 2.4)
    trunk_r = random.uniform(0.08, 0.16)
    trunk = cmds.polyCylinder(name=name + "_trunk", radius=trunk_r,
                               height=trunk_h, sx=8)[0]
    cmds.move(x, trunk_h * 0.5, z, trunk)

    trunk_color = _random_color(base=(0.35, 0.22, 0.12), spread=0.05)
    trunk_sg = _make_shader(name + "_trunkShader", trunk_color)
    _assign_shader(trunk_sg, trunk)

# use the seasonal bare-tree probability to decide whether the tree has foliage
    if random.random() < palette["bare_chance"]:
        return [trunk], trunk_r

# choose random style of foliage
    foliage_style = random.choice(["sphere", "cone", "sphere"])
    foliage_r = random.uniform(0.6, 1.4)
    if foliage_style == "sphere":
        foliage = cmds.polySphere(name=name + "_foliage", radius=foliage_r,
                                   sx=8, sy=8)[0]
        cmds.move(x, trunk_h + foliage_r * 0.7, z, foliage)
    else:
        foliage_h = foliage_r * random.uniform(1.6, 2.2)
        foliage = cmds.polyCone(name=name + "_foliage", radius=foliage_r,
                                 height=foliage_h, sx=8)[0]
        cmds.move(x, trunk_h + foliage_h * 0.5, z, foliage)

    if palette["blossom_colors"] and random.random() < palette["blossom_chance"]:
        foliage_base = random.choice(palette["blossom_colors"])
    else:
        foliage_base = random.choice(palette["foliage"])
    foliage_color = _random_color(base=foliage_base, spread=0.08)

    foliage_sg = _make_shader(name + "_foliageShader", foliage_color)
    _assign_shader(foliage_sg, foliage)

    return [trunk, foliage], max(trunk_r, foliage_r)


def create_bush(name, x, z, palette=None):
    palette = palette or SEASON_PALETTES["summer"]
    clump = []
    blob_count = random.randint(2, 4)
    base_r = random.uniform(0.25, 0.5)
    for i in range(blob_count):
        r = base_r * random.uniform(0.7, 1.1)
        blob = cmds.polySphere(name="%s_blob%d" % (name, i + 1),
                                radius=r, sx=6, sy=6)[0]
        ox = random.uniform(-base_r * 0.5, base_r * 0.5)
        oz = random.uniform(-base_r * 0.5, base_r * 0.5)
        cmds.move(x + ox, r * 0.8, z + oz, blob)
        clump.append(blob)

    bush_color = _random_color(base=palette["bush"], spread=0.06)
    bush_sg = _make_shader(name + "_bushShader", bush_color)
    _assign_shader(bush_sg, *clump)
    return clump, base_r * 1.5


def create_bench(name, x, z):
    facing = random.choice([0, 90, 180, 270]) 
  # benches facing four random directions
    seat = cmds.polyCube(name=name + "_seat", w=1.4, h=0.08, d=0.5)[0]
    back = cmds.polyCube(name=name + "_back", w=1.4, h=0.5, d=0.08)[0]
    leg1 = cmds.polyCube(name=name + "_leg1", w=0.08, h=0.4, d=0.5)[0]
    leg2 = cmds.polyCube(name=name + "_leg2", w=0.08, h=0.4, d=0.5)[0]
   # creat four parts of the bench

    cmds.move(0, 0.4, 0, seat)
    cmds.move(0, 0.65, -0.21, back)
    cmds.move(-0.65, 0.2, 0, leg1)
    cmds.move(0.65, 0.2, 0, leg2)

    parts = [seat, back, leg1, leg2]
    bench_grp = cmds.group(parts, name=name + "_grp")
    cmds.move(x, 0, z, bench_grp)
    cmds.rotate(0, facing, 0, bench_grp)
   # move them together

    wood_color = _random_color(base=(0.45, 0.28, 0.12), spread=0.06)
    wood_sg = _make_shader(name + "_woodShader", wood_color)
    _assign_shader(wood_sg, *parts)
    return [bench_grp], 0.9


def create_lamp_post(name, x, z):
    pole_h = random.uniform(2.5, 3.5)
    pole = cmds.polyCylinder(name=name + "_pole", radius=0.05,
                              height=pole_h, sx=8)[0]
    cmds.move(x, pole_h * 0.5, z, pole)

    bulb = cmds.polySphere(name=name + "_bulb", radius=0.18, sx=8, sy=8)[0]
    cmds.move(x, pole_h + 0.1, z, bulb)

    pole_sg = _make_shader(name + "_poleShader", (0.1, 0.1, 0.1))
    bulb_sg = _make_shader(name + "_bulbShader", (0.95, 0.9, 0.6))
    _assign_shader(pole_sg, pole)
    _assign_shader(bulb_sg, bulb)

    return [pole, bulb], 0.3


# ---------------------------------------------------------------------------
# main builder
# ---------------------------------------------------------------------------

def create_random_park(name="park", size=30,
                        tree_count=12, road_count=4, road_width=2.0,
                        pond_count=1, pond_size=2.5,
                        fountain_count=1, bench_count=4, light_count=5,
                        bush_count=None, season="summer",
                        seed=None, clear_existing=True):


    if seed:
        random.seed(seed)

    palette, season_name = _resolve_season(season)

    width = depth = size
    grp_name = name + "_grp"
    if clear_existing and cmds.objExists(grp_name):
        cmds.delete(grp_name)

    if bush_count is None:
        bush_count = max(0, int(tree_count * 0.6))

    all_parts = []

    ground_geo, _ = create_ground(name, width, depth, palette=palette)
    all_parts.extend(ground_geo)

    grid = _PlacementGrid(width, depth, margin=0.6)

    plaza_radius = max(1.0, road_width * 0.75)
    road_geo, roads_info = create_road_network(name, road_count, width, depth,
                                                grid, road_width=road_width,
                                                plaza_radius=plaza_radius)
    all_parts.extend(road_geo)

    for i in range(pond_count): # loope
        spot = grid.find_spot(pond_size)
        if not spot:
            continue
        geo, _ = create_pond("%s_pond%d" % (name, i + 1), spot[0], spot[1],
                              radius=pond_size, palette=palette)
        all_parts.extend(geo)

    for i in range(fountain_count):
        spot = grid.find_spot(random.uniform(1.0, 1.6))
        if not spot:
            continue
        geo, _ = create_fountain("%s_fountain%d" % (name, i + 1), spot[0], spot[1],
                                  palette=palette)
        all_parts.extend(geo)

    for i in range(tree_count):
        spot = grid.find_spot(random.uniform(0.6, 1.2))
        if not spot:
            continue
        geo, _ = create_tree("%s_tree%d" % (name, i + 1), spot[0], spot[1],
                              palette=palette)
        all_parts.extend(geo)

    for i in range(bush_count):
        spot = grid.find_spot(random.uniform(0.3, 0.6))
        if not spot:
            continue
        geo, _ = create_bush("%s_bush%d" % (name, i + 1), spot[0], spot[1],
                              palette=palette)
        all_parts.extend(geo)

    for i in range(bench_count):
        r = 0.9
        spot = None
        if roads_info and random.random() < 0.85:
            candidate = _roadside_spot(roads_info, r, road_width=road_width)
            if candidate and grid.is_free(candidate[0], candidate[1], r):
                spot = candidate
                grid.register(spot[0], spot[1], r)
        if spot is None:
            spot = grid.find_spot(r)
        if not spot:
            continue
        geo, _ = create_bench("%s_bench%d" % (name, i + 1), spot[0], spot[1])
        all_parts.extend(geo)

    for i in range(light_count):
        r = 0.3
        spot = None
        if roads_info:
            # lights should sit beside a road -- keep trying roadside
            # spots before ever falling back to a free-standing one
            for _ in range(8):
                candidate = _roadside_spot(roads_info, r, road_width=road_width)
                if candidate and grid.is_free(candidate[0], candidate[1], r):
                    spot = candidate
                    grid.register(spot[0], spot[1], r)
                    break
        if spot is None:
            spot = grid.find_spot(r)
        if not spot:
            continue
        geo, _ = create_lamp_post("%s_light%d" % (name, i + 1), spot[0], spot[1])
        all_parts.extend(geo)

    group = cmds.group(all_parts, name=grp_name)

    print("Created %s (season=%s, size=%d, trees=%d, roads=%d [width=%.1f], "
          "ponds=%d [size=%.1f], fountains=%d, benches=%d, lights=%d)" %
          (group, season_name, size, tree_count, road_count, road_width,
           pond_count, pond_size, fountain_count, bench_count, light_count))
    return group


# ---------------------------------------------------------------------------
# GUI
# ---------------------------------------------------------------------------

_WINDOW_NAME = "randomParkWin"


def show_random_park_ui():
    if cmds.window(_WINDOW_NAME, exists=True):
        cmds.deleteUI(_WINDOW_NAME)

    window = cmds.window(_WINDOW_NAME, title="Random Park Generator",
                          widthHeight=(400, 600), sizeable=True)
    cmds.columnLayout(adjustableColumn=True, rowSpacing=6,
                       columnAttach=("both", 10))

    cmds.text(label="Random Park Generator", font="boldLabelFont", height=28)
    cmds.separator(height=8, style="in")

    size_ctrl = cmds.intSliderGrp(label="Park Size", field=True,
                                   minValue=10, maxValue=150,
                                   fieldMinValue=10, fieldMaxValue=1000,
                                   value=30)
    tree_ctrl = cmds.intSliderGrp(label="Trees", field=True,
                                   minValue=0, maxValue=60,
                                   fieldMinValue=0, fieldMaxValue=300,
                                   value=12)
    road_ctrl = cmds.intSliderGrp(label="Roads", field=True,
                                   minValue=0, maxValue=10,
                                   fieldMinValue=0, fieldMaxValue=20,
                                   value=4)
    road_width_ctrl = cmds.floatSliderGrp(label="Road Width", field=True,
                                           minValue=0.5, maxValue=6.0,
                                           fieldMinValue=0.1, fieldMaxValue=20.0,
                                           value=2.0, precision=2)
    pond_ctrl = cmds.intSliderGrp(label="Ponds", field=True,
                                   minValue=0, maxValue=5,
                                   fieldMinValue=0, fieldMaxValue=10,
                                   value=1)
    pond_size_ctrl = cmds.floatSliderGrp(label="Pond Size", field=True,
                                          minValue=0.5, maxValue=8.0,
                                          fieldMinValue=0.1, fieldMaxValue=30.0,
                                          value=2.5, precision=2)

    season_ctrl = cmds.optionMenu(label="Season")
    for item in ["Spring", "Summer", "Autumn", "Winter", "Random"]:
        cmds.menuItem(label=item)
    cmds.optionMenu(season_ctrl, edit=True, value="Summer")

    fountain_ctrl = cmds.intSliderGrp(label="Fountains", field=True,
                                       minValue=0, maxValue=5,
                                       fieldMinValue=0, fieldMaxValue=10,
                                       value=1)
    bench_ctrl = cmds.intSliderGrp(label="Benches", field=True,
                                    minValue=0, maxValue=30,
                                    fieldMinValue=0, fieldMaxValue=100,
                                    value=4)
    light_ctrl = cmds.intSliderGrp(label="Lights", field=True,
                                    minValue=0, maxValue=40,
                                    fieldMinValue=0, fieldMaxValue=100,
                                    value=5)

    cmds.separator(height=8, style="in")
    seed_ctrl = cmds.intFieldGrp(label="Seed (0 = random)", value1=0)
   # 0 means generate a new random layout each time
    clear_ctrl = cmds.checkBoxGrp(label1="Clear previous park before generating",
                                   value1=True)

    cmds.separator(height=10, style="in")

# found in MAYA.help -- intSliderGrp -- create a integer slider group
#    Create a window with a couple integer slider groups.  The first will
#    use default limit values, and the second will set up a group that has
#    a field range greater than the slider range.  Try entering values
#    greater than the slider limits in both groups.
    def _on_generate(*_args):
        size = cmds.intSliderGrp(size_ctrl, query=True, value=True)
        trees = cmds.intSliderGrp(tree_ctrl, query=True, value=True)
        roads = cmds.intSliderGrp(road_ctrl, query=True, value=True)
        road_width = cmds.floatSliderGrp(road_width_ctrl, query=True, value=True)
        ponds = cmds.intSliderGrp(pond_ctrl, query=True, value=True)
        pond_size = cmds.floatSliderGrp(pond_size_ctrl, query=True, value=True)
        season = cmds.optionMenu(season_ctrl, query=True, value=True).lower()
        fountains = cmds.intSliderGrp(fountain_ctrl, query=True, value=True)
        benches = cmds.intSliderGrp(bench_ctrl, query=True, value=True)
        lights = cmds.intSliderGrp(light_ctrl, query=True, value=True)
        seed_val = cmds.intFieldGrp(seed_ctrl, query=True, value1=True)
        clear_existing = cmds.checkBoxGrp(clear_ctrl, query=True, value1=True)

        create_random_park(size=size,
                            tree_count=trees,
                            road_count=roads,
                            road_width=road_width,
                            pond_count=ponds,
                            pond_size=pond_size,
                            season=season,
                            fountain_count=fountains,
                            bench_count=benches,
                            light_count=lights,
                            seed=(seed_val if seed_val else None),
                            clear_existing=clear_existing)

    def _on_close(*_args):
        if cmds.window(_WINDOW_NAME, exists=True):
            cmds.deleteUI(_WINDOW_NAME)

    cmds.rowLayout(numberOfColumns=2, adjustableColumn=1,
                   columnWidth2=(220, 150), columnAttach=[(1, "both", 4), (2, "both", 4)])
    cmds.button(label="Generate Park", height=36, backgroundColor=(0.4, 0.6, 0.4),
                command=_on_generate)
    cmds.button(label="Close", height=36, command=_on_close)
    cmds.setParent("..")

    cmds.showWindow(window)


if __name__ == "__main__":
    show_random_park_ui()
