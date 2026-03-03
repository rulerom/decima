If graph fails validation — load rejected with `TopologyValidationError`.

> *This is not "antivirus". This is personality physical sanity check.*

**Store: publication requirements**

When publishing personality to Store, author provides:

| Component | Status | Check |
|-----------|--------|-------|
| .d8p file | Required | Spec validation + PKI signature |
| Frontend (minimal) | Required | Not checked (user code) |
| Documentation | Required | 8 strings description, outputs interpretation |
| Run example | Required | Script / instruction |

**Why we don't check frontend:**

- Technically impossible (code in Python/Rust/Go/C++)
- Legally complex (don't want to bear responsibility)
- Philosophically wrong (Decima-8 is open standard)

**Instead of check:**

- Publication requirement (no frontend = no publication)
- User warning ("run in sandbox")
- Rating system (reviews, author reputation)

**Recommendations**

**When loading personality from Store:**

- Run in sandbox (Docker, VM, seccomp, AppArmor)
- Limit network access (if not required)
- Set memory and CPU limits (cgroups, ulimit)
- Check author reputation (rating, reviews)

**When publishing:**

- Provide minimal working frontend
- Document inputs/outputs (8 strings, BUS16, PATTERN_ID)
- Warn about risks (network, FS, external APIs)

**What we DON'T guarantee**

| Don't guarantee | Why |
|-----------------|-----|
| Bug-free frontend | Author code, you responsible |
| Conductor stability | Your code, you responsible |
| .d8p describes "good" personality | Check physics, not semantics |
| PKI key not compromised | Store keys securely |

**Summary:** Decima-8 localizes vulnerabilities. Attacking core impossible (no code, determinism). Attacking perimeter possible — but these are classic vectors with classic defenses.

> *💭 **Architectural honesty:** you know exactly where risk is and where not.*

---

## 7. DISTRIBUTION MODEL

Decima-8 develops as open specification project with curated personality marketplace. Below is how distribution and support work.

### 7.1 Open and closed components

| Component | Status | Purpose |
|-----------|--------|---------|
| Specs + Emulator | ✅ OPEN | Verification, integrations, forks |
| Format .d8p | ✅ OPEN | Personality container (TLV) |
| libd8p (parser) | ✅ OPEN | Validation, generation, signature |
| IDE | 🔒 CLOSED (Free) | Reference tool for tuning |
| Store | 🔒 CLOSED (Curated) | Personality publishing and sharing |

> *💭 **Principle:** specification is open — anyone can write their own .d8p generator, emulator or tool. Store — curated venue with signature validation and spec compliance.*

### 7.2 Store publication

Personality publication in Store requires **.d8p file PKI signature**.

**Why:**

- Authorship guarantee (key bound to account)
- File integrity (signature checked on load)
- Reputation system (reviews to author, not anonymous file)

**How to get key:**

- Subscription to Tile/Cluster/Council tiers (automatic issuance)
- Or your own PKI key from trusted center (Corporate CA, etc.)

**Important:** already published personalities **are not deleted** on subscription expiration. Subscription needed only for loading new or updating existing.

**Alternative signature: your own PKI key**

Store accepts keys from any trusted centers, not only ours.

**Process:**

1. **Get key** from your CA (corporate, government, etc.)
2. **Sign** .d8p:
```bash
openssl dgst -sha256 -sign decima_key.pem \
  -out personality.d8p.sig \
  personality.d8p
```

3. **Load to Store:** system will check trust chain to Root CA

**Nuances:**

- For public Store easier to use our PKI (Tile/Cluster/Council) — trusted by all users by default
- Your key requires users to import your Root CA
- Corporate use: internal Store + own CA

### 7.3 Development roadmap

**Next 6 months:**

- Further software development: libd8p, core, IDE
- Store launch (first personalities)
- Documentation in Russian and English

**6–24 months:**

- Converters from common formats (ONNX → D8P)
- University partnerships (research, coursework)
- FPGA prototype (hardware verification)

**2–4 years:**

- B2B pilots (robotics, predictive analytics)
- Integrations with frameworks (ROS 2, Azure IoT)
- Certified partners (FPGA/ASIC)

**4+ years:**

- IP licensing for chip vendors
- Royalties from sales (if Decima-8 standard used)
- Open SDK and firmware support

> *💭 These are not promises, but guidelines. Priorities may change depending on community and resources.*

## 🧩 Summary

| Aspect | Implementation |
|--------|----------------|
| Specification | Open, forks allowed |
| Store | Curated, with PKI signatures |
| Monetization | PKI key subscription (Tile/Cluster/Council), ASIC royalties |
| Community | Observer/Seed/Gardener — users; Tile/Cluster/Council — authors |
| Long-term | Project designed for 10+ years, not exit in 3 years |

> *💭 Decima-8 is infrastructure project. We don't sell software subscription, we build ecosystem.*

---

## 8. ARCHITECTURE EVOLUTION

Decima-8 v0.2 is **minimum viable architecture**. Not dogma, but starting point that proves principles work.

**What's fixed forever (principles):**

| Principle | Why it's foundation |
|-----------|---------------------|
| **Two-phase cycle** READ → WRITE | Determinism, no race conditions |
| **Relay activation** (graph, not packets) | 0% area for routers, zero jitter |
| **LevelN** (multi-bit activation) | "Intention strength" encoding in one clock |
| **Signed Decay** (decay to zero) | Stability, natural "forgetting" |
| **Fuse-by-Range** (threshold logic) | Flexible patterns, resonant paths |

**What can scale (parameters):**

| Parameter | v0.2 (now) | v1.0+ (future) | Why |
|-----------|------------|----------------|-----|
| **Level** | 16 (0..15) | 32 / 64 | Fine activation gradation, less quantization |
| **Weight** | SignedWeight5 [-7..+7] | SignedWeight7 [-31..+31] | More connection expressiveness |
| **Lanes** | 8 | 16 / 32 | Bandwidth, parallelism |
| **Fabric** | 8×32 .. 32×128 | 256×1024 / clusters | Complex hierarchical patterns |
| **Domains** | 16 | 32 / 64 | Fine reset and priority control |
| **Cycle time** | 22-311 μs (emulator) | <1 μs (ASIC) | Hard real-time for extreme tasks |

**Backward compatibility:**

All changes **compatible at principle level**:
- Two-phase cycle remains
- Relay activation is foundation
- Fuse-by-range, decay-to-zero are foundation

**Open specification allows:**

1. **Experiment:** fork emulator, change `Level16` → `Level32`, see how swarm behavior changes
2. **Propose extensions:** if your extension proves advantage — it can enter v1.0 via Spec RFC
3. **Build specialized variants:**
   - `Decima-8-Lite`: for IoT (fewer tiles, fewer weights, low power)
   - `Decima-8-Pro`: for HFT (more lanes, less cycle, determinism priority)
   - `Decima-8-Research`: for science (extended metrics, debug, logging)

> *💭 **Philosophy:** we fix *principles*, not *parameters*. Level16 and SignedWeight5 are not dogma, but starting point.*

---

## 9. CONCLUSION

Decima-8 is architecture that encodes activation level (Level16) in one clock, uses relay activation instead of packet routing, and guarantees deterministic execution time.

**Key properties:**

- **Level16:** 4 bits per activation, one clock per value
- **SignedWeight5:** signed weights [-7..+7], hardware-level lateral inhibition
- **Relay activation:** dependency graph instead of routers, 0% area for routing
- **Two-phase cycle:** READ → WRITE, fixed latency, zero jitter
- **Open specification:** specification, emulator, .d8p format — under MIT/Apache 2.0

**We don't promise AGI.**

We provide deterministic computational fabric for tasks where predictability, efficiency, and pattern expressiveness matter.

**What to do next**

**Verify:**

- Emulator: github.com/rulerom/decima8
- Specification: decima.rulerom.com/ru/CONTRACT/
- Run benchmarks on your hardware

**Experiment:**

- Write .d8p generator in Python/Rust/Go
- Modify emulator (Level32, different weights, new modes)
- Propose extension via Spec RFC

**Use:**

- IDE (1.3 MB, offline) for visual personality tuning
- Store for publishing and sharing (with PKI signature)
- Emulator for CI/CD integration, auto-tests, prototypes

**Resources**

| Resource | Description |
|----------|-------------|
| Contract v0.2 | decima.rulerom.com/ru/CONTRACT |
| Emulator (GitHub) | github.com/rulerom/decima8 |
| Bakery (reference) | bakery.rulerom.com |
| PKI center | pki.rulerom.com |
| libwui (UI engine) | libwui.org |

**Decima-8 is not "another neuromorphic project". This is attempt to build computation on energy levels, resonance, and relay activation.**

> *💭 If "from physics, not from marketing" approach is closer to you — welcome.*

---

## FAQ

**Q: Why not float32/float16?**
A: Level16 (0..15) is not "imprecise float", but semantic unit: energy level. For neuromorphic patterns 16 gradations enough, and fixed range gives determinism and hardware efficiency.

**Q: How to train?**
A: Manually via IDE: adjust thr_lo/hi, decay, routing, observing swarm response. This is not ML training (gradient descent), but personality sculpture — you set behavior through parameters.
Bakery (bakery.rulerom.com) is pattern reference, not auto-trainer. Plans — API for AI agents, but final validation remains with human.

**Q: Can use cycles in activation graph?**
A: Yes. Determinism preserved thanks to locked_before — state snapshot at READ phase start.

**Q: What if two tiles in same domain fuse simultaneously?**
A: Winner selected by priority8, on tie — by minimum tile_id. COLLIDE flag signals collision.

**Q: What is "double strait" and when to use it?**
A: Mode where core performs two internal clocks per one EV_FLASH for selectivity increase. Used for small Hamming distance pattern recognition (e.g., ASCII characters in VSB). In IDE enabled by "Double Strait" checkbox, in .d8p sets BAKE_FLAG_DOUBLE_STRAIT. Cost: ~40 μs instead of ~20 μs.

**Q: Why open specification?**
A: So anyone can verify math, write their own .d8p generator or fork emulator. Decima-8 is standard, not closed product.

**Q: What if I want to use .d8p locally, without Store?**
A: Please. Signature not required for local use. Emulator accepts any .d8p after CRC32 validation. PKI — only for Store publication.

**Q: Can sign .d8p with own PKI key?**
A: Yes. Store accepts keys from any trusted centers (Corporate CA, government UC). For public Store easier to use our PKI (Tile/Cluster/Council) — trusted by users by default.

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
