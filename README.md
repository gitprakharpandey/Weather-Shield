# WeatherShield 🌧️

Delivery workers don't stop working when it rains. They stop getting paid.

WeatherShield fixes that. When a verified red-alert weather event hits a worker's active zone, they get paid — automatically, within minutes, no forms filed. Think of it as a smoke detector for financial risk: it doesn't wait for you to notice the fire.

---

## Why This Exists

Parametric insurance has been around in agriculture and aviation for years. Nobody brought it to the guy on the bike delivering your biryani in a thunderstorm. We did.

The platform is stupid-simple on the surface: weather crosses a threshold → worker in that zone → money moves. The hard part — the part that took real work — is making sure the money moves to the *right* person and not to someone running a spoofing script on their couch.

---

## Get It Running (3 Steps)

```bash
git clone https://github.com/your-org/weathershield && cd weathershield
cp .env.example .env   # Drop in your IMD API key, UPI credentials, DB string
docker-compose up      # Spins up app + Postgres + Redis + Neo4j
```

That's it. Hit `localhost:3000`. The ML pipeline seeds with synthetic claim data so the fraud models aren't blind on day one.

---

## How the Money Moves

```
IMD / AccuWeather fires a Red Alert
         │
         ▼
System checks: who has an active order in this zone right now?
         │
         ▼
Each active worker runs through the Verification Engine
         │
         ▼
Score > 70  →  Auto-pay in ~4 minutes
Score 40–70 →  Provisional pay (70%) + silent 24hr audit
Score < 40  →  Human review queue, worker notified gently
```

No score is ever shown to the worker. They just see: "Processing" or "Paid."

| Alert Level   | Window      | Payout |
|---------------|-------------|--------|
| Orange Alert  | 2 hours     | ₹150   |
| Red Alert     | 30 minutes  | ₹300   |
| Extreme Event | Immediate   | ₹500   |

Funded by ₹5–15/day micro-premiums. Pool health is live-monitored. When velocity spikes, circuit breakers fire before the pool takes damage.

---

## The Intelligence Layer

Three models, one job: figure out if you're actually outside.

**Signal Fusion Model** — Your phone is a witness. GPS says where you are. Cell towers, Wi-Fi, accelerometer, battery drain, and network latency *confirm* it — independently. All six have to agree. One lie, one red flag.

**Behavioral Anomaly Detector** — Trained on 90 days of rolling baselines. Genuine claims trickle in over 20–40 minutes as workers get caught by weather. Fraud rings explode in a 90-second window. That shape is unmistakable.

**Graph Fraud Detector (Neo4j + GraphSAGE)** — The individual is the wrong unit of analysis for organized fraud. The graph is. Workers who onboarded together, share a router, opened the same deep-link in the same 8-second window — that's the ring. That's what gets caught.

---

## Adversarial Defense & Anti-Spoofing Strategy

> **The Attack:** A 500-person syndicate coordinates on Telegram, installs GPS-spoofing apps, and simultaneously fakes their location into an active red-alert zone. Liquidity pool drains in minutes.

This is not a hypothetical. It is the most sophisticated attack vector for any parametric payout system. Here's the full architectural response.

---

### The Core Differentiation: Real Bodies vs. Spoofed Coordinates

GPS is one wire. Cut it, fake it, done. Physical reality is a 47-wire harness.

A worker genuinely caught in a Delhi cloudburst produces a signal that looks like chaos — battery at 34% and falling fast, accelerometer vibrating with road texture and wind buffet, cell tower handoffs every 90 seconds as they move through degraded coverage, navigation app pinned to the foreground. Their Wi-Fi shows nothing; they're in the middle of a road.

A worker at home with a spoofing app produces a signal that looks like a stock photo of distress — GPS in the storm zone, everything else perfectly calm. Connected to home Wi-Fi. Stationary accelerometer. Charging cable in. Netflix running in the background.

Our **Multi-Signal Presence Verification Engine (MSPVE)** scores these eight signals independently, then fuses them. Spoof one signal and you raise suspicion. Spoof all eight from your living room and you'll need equipment that doesn't exist yet.

```
GENUINE PRESENCE SCORE  (0 – 100)
│
├── GPS Coordinates           [low weight alone — starting point only]
├── Cell Tower Triangulation  [SIM-level, GPS-independent — hard floor of truth]
├── Wi-Fi Fingerprint         [home SSID during a storm claim = immediate flag]
├── Accelerometer / Gyro      [storm = movement, vibration, shelter-seeking behavior]
├── Battery Drain Velocity    [screens on + GPS active + bad signal = measurable drain]
├── Network Latency Profile   [congested towers have a known signature in real events]
├── App Foreground State      [real workers navigate; fraudsters watch YouTube]
└── Route Coherence           [were you even working before this alert fired?]
```

---

### The Data: What Gets Analyzed Beyond GPS

#### Device-Level (The Physical Layer)

**Cell tower ID and signal strength** — Spoofing apps live in the software stack. Your SIM talks to actual antennas in the physical world. If GPS says Connaught Place but your SIM is pinging a tower in Dwarka, that gap is the tell. This single check catches ~60% of basic spoofing attempts.

**Wi-Fi network fingerprint** — Every claim carries the SSID and BSSID of nearby networks. We maintain a map of known residential networks per worker. If your home router shows up during a flood-zone claim, that's not ambiguous.

**Accelerometer and gyroscope** — Real weather creates real physical noise. Wind resistance on a moving bike, a worker ducking under an overhang, rain vibration on a delivery box — these leave a micro-signature. A spoofed claim from a stationary device at home is eerily still.

**Battery drain velocity** — Under storm conditions: screen on, GPS active, network constantly searching. Drain rate climbs measurably. A flat battery curve during a claimed emergency is a quiet contradiction.

#### Behavioral (The Intent Layer)

**Active order status** — No live delivery order at claim time means no active exposure. No exposure means no valid claim. This one filter alone removes a significant fraction of opportunistic fraud before any ML runs.

**Claim timing relative to alert** — Real distress distributes across time. Workers encounter the storm at different moments on different routes. Fraudsters all get the same Telegram message at the same second. The timing histogram of a genuine event looks like a bell curve. A fraud ring looks like a single vertical bar.

**App foreground state** — Navigation apps, weather apps, customer call logs — these are the digital fingerprints of someone actually working. A claim submitted while the app has been backgrounded for 40 minutes, with high social media activity running, reads differently.

**Historical route coherence** — Did this worker's GPS trace actually approach the claimed zone before the alert? A worker who was stationary in a residential area for three hours before suddenly "appearing" in a storm zone has a coherence problem.

#### Ring-Level (The Graph Layer)

This is where individual analysis fails and graph analysis wins. No single worker in a coordinated ring looks suspicious in isolation. The pattern only emerges at the network level.

**Claim velocity monitoring** — A 500-person simultaneous claim spike is 40–50 standard deviations above normal. This doesn't need ML to catch; it needs a counter and a threshold. The circuit breaker fires and pauses auto-payouts before the damage is done.

**Device clustering** — Multiple claims from devices sharing a router MAC address or overlapping Bluetooth beacon range means those workers were physically co-located. Five workers in the "same flood zone" who were all in the same apartment are not in a flood zone.

**Referral and onboarding graph** — A fraud ring recruits together. Workers who onboarded in the same 48-hour window through the same referral chain, and who are now claiming together, share a suspicious origin. This graph edge is invisible per-worker, visible at network level.

**Deep-link timing analysis** — The platform does not read Telegram content. It doesn't need to. If 300 workers all opened the same push notification or deep-link within an 8-second window — the statistical signature of a mass broadcast — that coordination signal is logged and weighted against the claim cluster.

**Geographic density anomaly** — Genuine storm exposure distributes across delivery routes across the city. Fraud rings often cluster in the same residential neighborhood. Three hundred claims from a 2km radius that isn't a delivery-dense commercial zone is not a weather event; it's a Telegram thread.

---

### The UX Balance: Not Punishing Honest Workers

The cruelest failure mode of any fraud system is wrongly flagging a genuine worker who's soaked through, battery at 8%, network dropping in and out — and then sending them a denial.

Design principle: **pay first, audit quietly, only hold when the evidence is decisive.**

#### Flagging Workflow

```
Claim arrives
      │
      ▼
MSPVE score calculated
      │
   ┌──┴────────┐
 > 70         < 70
   │              │
 Auto-pay      First-time flag?
               │
          ┌────┴─────┐
         YES          NO
          │            │
    Provisional     Hold +
    70% payout      Human review
    + silent        (24hr SLA)
    background
    audit
```

**Provisional payout for first-time flags** — Clean history workers who score below threshold still receive 70% of their payout immediately. The remaining 30% releases after a 24-hour background audit. The message they see: *"Your payout is being processed and will complete within 24 hours."* No accusation. No stigma.

**Weather-correlated leniency** — Bad weather legitimately degrades signal quality. When carrier data confirms cell network congestion in a zone, the MSPVE applies a degradation allowance. A weak signal in a real storm is expected. The model knows this. It does not penalize workers for the very conditions they're claiming against.

**Self-verification path** — Workers whose claims are held receive an in-app prompt within 15 minutes: *"We're taking a moment to process this. A quick photo of your location speeds things up."* Genuine workers can do this. Workers at home cannot. No interrogation — just an easy path to resolution.

**No silent denials, ever** — Every hold triggers a plain-language in-app message and a one-tap appeal. Appeals reach a human agent within 4 business hours. No IVR. No ticket queue that disappears.

**Ring enforcement is surgical** — If a coordinated fraud cluster is confirmed, enforcement targets the identified network graph — not every worker with a low score. An honest worker with weak signal who has zero connection to the fraud graph is treated as a genuine claimant, full stop.

---

### The Circuit Breaker: Liquidity Is the Last Line

Per-claim detection is necessary but not sufficient. A 500-person coordinated attack is a **systems-level problem**, not a claims-level problem. Even perfect individual scoring can't respond fast enough when 500 requests hit simultaneously.

The liquidity circuit breaker is a separate, independent layer that watches macro velocity, not individual scores:

- **2× 90-day baseline velocity** — On-call fraud team alerted. Watching.
- **3× baseline** — Auto-payouts paused. All new claims enter manual queue. Team mobilized.
- **5× baseline** — Emergency protocol. Pool locked. Legal and ops escalated. Zero releases until full audit.

The attack has to defeat eight signal dimensions *and* stay under the velocity radar. It can't do both.

---

### The Full Defense Picture

```
ATTACK VECTOR
500 workers, GPS-spoofed, Telegram-coordinated, simultaneous

LAYER 1 — Physical Reality:    Cell towers + Wi-Fi + accelerometer = you can't fake your body
LAYER 2 — Intent Signals:      No active order + bad timing + wrong app state = caught
LAYER 3 — Graph Analysis:      Velocity spike + device clustering + deep-link timing = ring confirmed
LAYER 4 — Circuit Breaker:     Pool locked before drain completes, regardless of claim scores
LAYER 5 — Human Review:        Every flagged cluster audited with full signal trail

RESULT
Honest workers: paid in minutes, never interrogated
Fraud ring: detected, held, reported, and very publicly not paid
```

---

## Stack

| What           | How                                  |
|----------------|--------------------------------------|
| Mobile         | React Native — iOS + Android         |
| Sensors        | Custom signal fusion SDK             |
| ML             | Python · scikit-learn · PyTorch      |
| Graph Engine   | Neo4j + GraphSAGE                    |
| Weather        | IMD API + AccuWeather                |
| Payments       | UPI · Direct bank transfer           |
| Backend        | Node.js · PostgreSQL · Redis         |
| Infrastructure | AWS — ap-south-1 (Mumbai)            |

---

## What's Next

**Q1** — Beta. 1,000 workers. One city. Learn everything.  
**Q2** — Anti-spoofing model v1 trained on real claim data, not synthetic.  
**Q3** — Graph fraud detection goes live. Ring detection at scale.  
**Q4** — Multi-city. Reinsurance layer. Liquidity pool v2.

---

*The delivery worker caught in a storm isn't asking for much. Just for the system to notice, and respond. We built the system.*
