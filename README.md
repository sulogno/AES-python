# Correlation Power Analysis (CPA) Attack on AES-128

A Python implementation of a **Correlation Power Analysis (CPA)** side-channel attack targeting AES-128 encryption. This project simulates power consumption leakage during AES encryption and statistically recovers the full 128-bit secret key using Pearson correlation — without ever having access to the key directly.

---

## What is a Side-Channel Attack?

Cryptographic algorithms like AES are mathematically secure — breaking them by brute force is computationally infeasible. However, a *physical* implementation of AES (on a microcontroller, FPGA, or smartcard) leaks information through measurable side channels such as:

- **Power consumption** — how much current the device draws per clock cycle
- **Electromagnetic emissions**
- **Timing variations**

A **Correlation Power Analysis (CPA)** attack exploits the power consumption side channel. By collecting many power traces during encryption and correlating them against a theoretical power model, an attacker can recover the secret key **statistically** — typically with a few hundred traces.

---

## How CPA Works

CPA targets the first round of AES, specifically the **SubBytes** operation, which is the first nonlinear step and the most exploitable leakage point.

### Attack Pipeline

```
[Known Plaintexts] + [Key Guess k*]
          │
          ▼
  Hypothesis: SBOX[plaintext[i] XOR k*]
          │
          ▼
  Power Model: Hamming Weight of SBOX output
          │
          ▼
  Pearson Correlation vs. Measured Power Traces
          │
          ▼
  Highest correlation → Correct key byte
```

### Step by Step

1. **Collect traces** — Encrypt N random plaintexts with the target device. Record power consumption per encryption.
2. **For each key byte (0–15):**
   - For each of the 256 possible key byte guesses:
     - Predict the intermediate value: `SBOX[pt[i] XOR k_guess]`
     - Compute its **Hamming Weight** (number of `1` bits) — this is the power model
     - Compute **Pearson correlation** between predicted HW values and actual measured traces
   - The guess with the **highest correlation** is the correct key byte.
3. **Repeat for all 16 bytes** → full 128-bit key recovered.

### Why Hamming Weight?

CMOS circuits consume power proportional to the number of bit transitions. The **Hamming Weight** of a data value (number of set bits) is a well-established first-order approximation of that power consumption, making it an effective leakage model.

---

## Repository Structure

```
CPA-python/
├── main.py        # AES-128 implementation + power trace simulator (single trace visualization)
└── cpa_attack     # Full CPA attack engine using numpy + Pearson correlation
```

### `main.py` — AES Implementation & Trace Visualizer

A from-scratch AES-128 implementation that also **simulates power leakage**:

- Full AES-128 encrypt and decrypt (SubBytes, ShiftRows, MixColumns, AddRoundKey, key schedule)
- `capture_hw_leakage()` — captures Hamming Weight of the S-Box output state per round
- `print_terminal_trace()` — renders a text-based bar chart of leakage in the terminal
- Saves a `matplotlib` power trace plot as `aes_power_trace.png`

**Run:**
```bash
python main.py
```

**Sample output:**
```
--- AES Side-Channel Simulation ---
Ciphertext (hex): ...

--- Terminal Power Trace Visualization (Hamming Weight) ---
Byte Index | HW | Visualization
---------------------------------------------
000 | 4 | ████
001 | 6 | ██████
...
Graph also saved as 'aes_power_trace.png'
Decrypted String: Attack at dawn!!
```

---

### `cpa_attack` — Full CPA Engine

The main attack script. Uses `numpy` for vectorised operations and implements the complete CPA pipeline:

**Classes:**

- **`AESAnalyzer`** — AES-128 engine with:
  - Full key schedule (`get_key_expansion`)
  - `simulate_trace(pt, key, noise_level)` — encrypts a plaintext and returns a 16-point power trace (HW of Round 1 S-Box output + Gaussian noise)
  - Precomputed Hamming Weight lookup table (`HW_LUT`) for performance

- **`CPAEngine`** — CPA attack engine with:
  - `perform_attack(plaintexts, traces)` — iterates over all 16 key bytes × 256 hypotheses, computes Pearson correlation for each, returns recovered key
  - `plot_results(secret_key)` — plots correlation curves for all 16 bytes, marking correct key bytes in red

**Configuration** (inside `main()`):

| Parameter | Default | Description |
|---|---|---|
| `TRACES_COUNT` | 300 | Number of power traces collected |
| `NOISE_SIGMA` | 0.1 | Standard deviation of Gaussian noise added to traces |
| `SECRET_KEY` | `b"hhisis128bitkey!"` | The 128-bit target key |

**Run:**
```bash
python cpa_attack
```

**Sample output:**
```
------------------------------------------------------------
 AES-128 SIDE-CHANNEL LEAKAGE SIMULATOR & CPA ATTACK
------------------------------------------------------------
INFO: Step 1: Capturing Power Traces...
INFO: Step 2: Analyzing Leakage Patterns...
INFO: Initiating CPA on 300 traces...
INFO: Byte 00 | Guess: 0x68 | Max Corr: 0.9812
INFO: Byte 01 | Guess: 0x68 | Max Corr: 0.9734
...
============================================================
ORIGINAL KEY (HEX):  48484953495331323842495459212121
RECOVERED KEY (HEX): 48484953495331323842495459212121
DECODED STRING     : hhisis128bity!!!
============================================================
[!] SUCCESS: 128-bit Encryption Key Fully Compromised!
```

---

## Installation

```bash
git clone https://github.com/sulogno/CPA-python.git
cd CPA-python
pip install numpy matplotlib
```

No other dependencies required. The AES implementation is built from scratch with no external crypto libraries.

---

## Results & Observations

- With **300 traces** and **noise σ = 0.1**, all 16 key bytes are recovered successfully.
- Reducing traces to ~100 may cause some bytes to fail — increasing `TRACES_COUNT` improves reliability.
- Setting `noise_level = 0.0` gives a near-perfect attack with even fewer traces.
- The correlation plot clearly shows a **sharp spike** at the correct key byte for each of the 16 bytes, with all wrong hypotheses showing low, flat correlation.

---

## Countermeasures

This attack is effective against *unprotected* implementations. Common defences include:

| Countermeasure | How It Helps |
|---|---|
| **Boolean Masking** | XORs intermediate values with a random mask, decorrelating power from data |
| **Noise Injection** | Adds random operations to obscure the true leakage signal |
| **Power Balancing** | Circuit design that consumes equal power regardless of data |
| **Shuffling** | Randomises the order of S-Box operations across bytes |
| **Higher-Order CPA Resistance** | Requires combining multiple leakage points to mount an attack |

---

## References

- Brier, Clavier, Olivier — *Correlation Power Analysis with a Leakage Model* (CHES 2004)
- Mangard, Oswald, Popp — *Power Analysis Attacks: Revealing the Secrets of Smart Cards* (2007)
- NIST FIPS 197 — *Advanced Encryption Standard (AES)*

---

## Author

**Sulogno Sarkar**  
B.Tech Computer Science (Data Science), Haldia Institute of Technology  
[github.com/sulogno](https://github.com/sulogno) · [LinkedIn](https://linkedin.com/in/sulogno-sarkar-8007a82b6)

> *This project is for educational and research purposes only. Understanding attacks is the first step to building better defences.*
