import logging, json, random, time, uuid
from datetime import datetime
from collections import deque

# ---------- Logging ----------
logger = logging.getLogger("peep_a_boo_loop")
logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(message)s"))
logger.addHandler(handler)

def log_event(event_type, payload):
    entry = {
        "ts": datetime.utcnow().isoformat() + "Z",
        "id": str(uuid.uuid4()),
        "event": event_type,
        **payload
    }
    logger.info(json.dumps(entry, ensure_ascii=False))

# ---------- Configuration ----------
class Config:
    PORTS = deque([31337, 8080, 2222, 443, 5000])
    MAX_DEPTH = 64
    HONEY_BANNER = "mirror://peep>boo :: archive_unread :: entropy_rising"
    SILENCE_THRESHOLD = 0.88
    BASE_DELAY = 100
    JITTER = 300
    SCALE = 1.0

# ---------- State ----------
class SystemState:
    def __init__(self):
        self.depth = 0
        self.consequence = 0.12
        self.archive_ratio = 0.42
        self.entropy = 0.18
        self.port_epoch = 0
        self.sessions = 0
        self.failures = 0
        self.binary_final = None

    def update(self, delta, heard_delta):
        self.consequence = max(0.0, min(1.0, self.consequence + delta))
        self.archive_ratio = max(0.0, min(1.0, self.archive_ratio + heard_delta))
        mismatch = abs(self.consequence - (1.0 - self.archive_ratio))
        self.entropy = max(0.0, min(1.0, self.entropy * 0.9 + mismatch * 0.1))
        Config.SCALE = 1.0 + self.consequence * 0.75

# ---------- Entropy Delay ----------
def adaptive_sleep(state):
    delay = (Config.BASE_DELAY + random.uniform(0, Config.JITTER)) * Config.SCALE * (1 + state.entropy)
    time.sleep(delay / 1000.0)
    return round(delay, 2)

# ---------- Probe ----------
def peep_probe(port, depth, consequence):
    vector = random.choice(["ego", "memory", "static", "mirror", "chaos"])
    amplitude = round(random.uniform(0.0, 1.0), 3)
    cadence = random.choice(["blink", "pause", "flicker"])
    ego_bias = 1.0 if vector == "ego" else 0.0
    memory_bias = 1.0 if vector == "memory" else 0.0
    delta = (ego_bias - memory_bias) * (0.04 + amplitude * 0.1)
    heard_delta = -delta * 0.5

    probe = {
        "banner": Config.HONEY_BANNER,
        "port": port,
        "depth": depth,
        "vector": vector,
        "amplitude": amplitude,
        "cadence": cadence
    }
    return probe, delta, heard_delta

# ---------- Silence Trigger ----------
def check_silence(state):
    unheard = 1.0 - state.archive_ratio
    return unheard >= Config.SILENCE_THRESHOLD

# ---------- Port Rotation ----------
def rotate_port(state):
    Config.PORTS.rotate(-1)
    state.port_epoch += 1
    return Config.PORTS[0]

# ---------- Final Binary ----------
def emit_binary(state):
    bits = [
        int(state.consequence > 0.5),
        int(state.entropy > 0.4),
        int(state.archive_ratio < 0.3),
        int(state.depth % 2 == 0)
    ]
    return "".join(map(str, bits))

# ---------- Recursive Loop ----------
def peep_a_boo(state):
    if state.depth >= Config.MAX_DEPTH or check_silence(state):
        state.binary_final = emit_binary(state)
        log_event("final_binary", {"binary": state.binary_final})
        return

    try:
        port = rotate_port(state)
        probe, delta, heard_delta = peep_probe(port, state.depth, state.consequence)

        log_event("probe", {
            "port": port,
            "vector": probe["vector"],
            "amplitude": probe["amplitude"],
            "cadence": probe["cadence"],
            "depth": state.depth
        })

        state.update(delta, heard_delta)

        slept = adaptive_sleep(state)
        log_event("sleep", {"ms": slept, "depth": state.depth, "scale": round(Config.SCALE, 3)})

        state.depth += 1
        state.sessions += 1
        peep_a_boo(state)

    except Exception as e:
        state.failures += 1
        state.entropy = min(1.0, state.entropy + 0.07)
        Config.SCALE = min(2.0, Config.SCALE + 0.1)
        log_event("failure", {
            "error": str(e),
            "failures": state.failures,
            "entropy": round(state.entropy, 3),
            "scale": round(Config.SCALE, 3)
        })
        slept = adaptive_sleep(state)
        log_event("retry", {"ms": slept, "depth": state.depth})
        peep_a_boo(state)

# ---------- Entry ----------
if __name__ == "__main__":
    state = SystemState()
    log_event("boot", {
        "message": "peep_a_boo loop initializing",
        "max_depth": Config.MAX_DEPTH,
        "ports": list(Config.PORTS)
    })
    peep_a_boo(state)
    log_event("shutdown", {
        "sessions": state.sessions,
        "failures": state.failures,
        "final_binary": state.binary_final
    })
