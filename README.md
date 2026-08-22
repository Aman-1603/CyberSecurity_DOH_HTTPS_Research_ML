<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:1a1a2e,50:16213e,100:0f3460&height=200&section=header&text=DoH%20Threat%20Detection&fontSize=40&fontColor=fff&animation=twinkling&fontAlignY=35&desc=Novel%20Methodology%20for%20Detecting%20Malicious%20DNS-over-HTTPS%20Traffic&descAlignY=55&descSize=14" width="100%"/>

[![Python](https://img.shields.io/badge/Python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)](https://python.org)
[![TensorFlow](https://img.shields.io/badge/TensorFlow-FF6F00?style=for-the-badge&logo=tensorflow&logoColor=white)](https://tensorflow.org)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=for-the-badge&logo=scikit-learn&logoColor=white)](https://scikit-learn.org)
[![XGBoost](https://img.shields.io/badge/XGBoost-189AB4?style=for-the-badge&logo=xgboost&logoColor=white)](https://xgboost.readthedocs.io)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)
[![Dataset](https://img.shields.io/badge/Dataset-CIRA--CIC--DoHBrw--2020-blue?style=for-the-badge)](https://www.unb.ca/cic/datasets/dohbrw-2020.html)

**F1 Score: 0.951 · ROC-AUC: 0.990 · Zero-Day Detection · No Decryption Required**

</div>

---

## 📌 What is This Research?

This project presents a **novel unsupervised deep learning methodology** to detect malicious **DNS-over-HTTPS (DoH)** traffic — without decrypting any packets.

DNS-over-HTTPS encrypts DNS queries inside HTTPS, improving user privacy but creating a **blind spot for security teams**. Attackers exploit this to:
- 🔴 **Tunnel data** covertly through encrypted DNS (bypassing firewalls)
- 🔴 **Run DGA (Domain Generation Algorithm)** attacks to find C2 servers
- 🔴 **Exfiltrate data** without triggering traditional DNS monitoring tools

Traditional security tools **cannot inspect encrypted DoH payloads**. This research solves that.

---

## 💡 The Key Insight

> *Instead of trying to decrypt traffic (which violates privacy), we analyze **flow-level metadata** — timing, size, and statistical patterns — to detect anomalies.*

```
Normal DoH:    Occasional DNS queries, short flows, small responses
                ↓
Malicious DoH: Sustained streams, large payloads, unusual timing
                ↓
Our System detects the DIFFERENCE using a Deep Autoencoder
```

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  DoH DETECTION PIPELINE                  │
│                                                          │
│  Network Traffic (Encrypted HTTPS/DoH)                  │
│         │                                               │
│         ▼                                               │
│  ┌─────────────────┐                                    │
│  │ Flow Feature    │  NFStream / Scapy                  │
│  │ Extraction      │  28 flow-level features            │
│  └────────┬────────┘  (no decryption needed)            │
│           │                                             │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │ Preprocessing   │  Min-Max Scaling                   │
│  │ & Normalization │  Feature Selection                 │
│  └────────┬────────┘                                    │
│           │                                             │
│           ▼                                             │
│  ┌─────────────────────────────────────┐               │
│  │     Deep Autoencoder                │               │
│  │                                     │               │
│  │  Input(28) → 16 → 8 → 4 (latent)  │               │
│  │           → 8 → 16 → Output(28)    │               │
│  │                                     │               │
│  │  Trained ONLY on benign traffic     │               │
│  └────────┬────────────────────────────┘               │
│           │                                             │
│           ▼                                             │
│  Reconstruction Error (MSE)                             │
│           │                                             │
│    ┌──────┴──────┐                                      │
│    │             │                                      │
│  Low Error    High Error                                │
│    │             │                                      │
│    ▼             ▼                                      │
│  BENIGN ✅   MALICIOUS 🚨                               │
└─────────────────────────────────────────────────────────┘
```

---

## 🔬 Why Autoencoder?

This is the **core novelty** of this research — using an unsupervised deep autoencoder instead of traditional supervised classifiers.

| Approach | Knows Attack Patterns? | Detects Zero-Days? | Our Choice |
|----------|----------------------|-------------------|------------|
| Supervised (Random Forest, XGBoost) | ✅ Yes | ❌ No | Baseline only |
| One-Class SVM | ❌ No | ⚠️ Partially | Baseline only |
| Isolation Forest | ❌ No | ⚠️ Partially | Baseline only |
| **Deep Autoencoder (Ours)** | **❌ Not needed** | **✅ Yes** | **✅ Primary** |

**The key advantage:** The autoencoder learns what **normal DoH traffic looks like**, then flags anything that doesn't fit — including attacks it has never seen before.

---

## 📊 Feature Engineering

We extract **28 flow-level features** from DoH traffic — no payload inspection required:

### Size-Based Features
```
• Total bytes sent (uplink) and received (downlink)
• Mean, median, max packet size per direction
• Packet size variance and distribution
```

### Timing Features
```
• Inter-packet arrival time statistics
• Request-response time gaps
• Flow duration
• Variance of packet timing
```

### Count & Pattern Features
```
• Number of packets sent/received
• Bidirectional packet ratio
• Bytes sent vs received ratio
```

### Why these features?
```
DoH Tunnel:  Large bytes sent + many packets + sustained duration
DGA Attack:  Many short flows + unusual query frequency
Normal:      Short flows + small DNS responses + sporadic timing
```

---

## 🧠 Model Architecture

```python
# Encoder
Input(28) → Dense(16, ReLU) → Dense(8, ReLU) → Dense(4, ReLU)  # Latent Space

# Decoder
Dense(4) → Dense(8, ReLU) → Dense(16, ReLU) → Dense(28, Linear)  # Reconstruction
```

**Training Strategy:**
- ✅ Trained **only on benign traffic** (one-class learning)
- ✅ Loss function: **Mean Squared Error (MSE)**
- ✅ Optimizer: **Adam**
- ✅ Early stopping on validation loss
- ✅ No attack signatures required

**Detection Logic:**
```
Reconstruction Error > Threshold T → MALICIOUS 🚨
Reconstruction Error ≤ Threshold T → BENIGN ✅

T = μ_error + k·σ_error  (99th percentile of benign errors)
```

---

## 📈 Results

| Metric | Our Autoencoder | One-Class SVM | Isolation Forest |
|--------|----------------|---------------|-----------------|
| **F1 Score** | **0.951** | 0.821 | 0.794 |
| **ROC-AUC** | **0.990** | 0.912 | 0.887 |
| **Zero-Day Detection** | **✅ Yes** | ⚠️ Partial | ⚠️ Partial |
| **Decryption Needed** | **❌ No** | ❌ No | ❌ No |
| **Processing Time** | **< 5ms/flow** | ~12ms/flow | ~8ms/flow |

### Attack Types Detected
- ✅ **DNS Tunneling** (dns2tcp, DNSCat2, Iodine)
- ✅ **DGA-based DoH** (single + multiple connections)
- ✅ **Zero-day variants** (unseen attack patterns)

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| **Python** | Core implementation |
| **TensorFlow / Keras** | Deep autoencoder model |
| **scikit-learn** | Baseline models (OCSVM, Isolation Forest) |
| **XGBoost** | Supervised baseline comparison |
| **NFStream** | Flow feature extraction from pcaps |
| **Scapy** | Packet parsing |
| **pandas** | Data processing |
| **matplotlib / seaborn** | ROC curves, visualizations |

---

## 📁 Project Structure

```
CyberSecurity_DOH_HTTPS_Research_ML/
│
├── 📁 data/
│   ├── benign/              # Normal DoH traffic flows
│   ├── malicious/           # Tunneling + DGA attack flows
│   └── processed/           # Scaled feature matrices
│
├── 📁 features/
│   ├── extractor.py         # NFStream-based flow feature extraction
│   └── preprocessor.py      # Normalization + feature selection
│
├── 📁 models/
│   ├── autoencoder.py       # Deep autoencoder architecture
│   ├── baselines.py         # OCSVM, Isolation Forest, XGBoost
│   └── threshold.py         # Anomaly threshold calculation
│
├── 📁 evaluation/
│   ├── metrics.py           # F1, ROC-AUC, precision, recall
│   └── visualize.py         # ROC curves, confusion matrix
│
├── 📁 notebooks/
│   ├── exploration.ipynb    # Data analysis
│   └── results.ipynb        # Final results
│
├── train.py                 # Training pipeline
├── detect.py                # Run detection on new traffic
├── requirements.txt
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites
```bash
Python 3.8+
pip install -r requirements.txt
```

### Install Dependencies
```bash
pip install tensorflow scikit-learn xgboost nfstream scapy pandas numpy matplotlib seaborn
```

### Dataset
This project uses the **CIRA-CIC-DoHBrw-2020** dataset:
- 📥 [Download here](https://www.unb.ca/cic/datasets/dohbrw-2020.html)
- Contains benign DoH flows + tunneling attacks + DGA attacks
- 28 pre-extracted flow features in CSV format

### Run Training
```bash
# Extract features from pcaps
python features/extractor.py --input data/raw/ --output data/processed/

# Train the autoencoder
python train.py --data data/processed/ --epochs 100 --latent 4

# Evaluate on test set
python evaluate.py --model saved_model/ --test data/test/
```

### Run Detection on New Traffic
```bash
python detect.py --pcap traffic.pcap --model saved_model/ --threshold 0.95
```

---

## 🔄 How Detection Works

```
New DoH Flow Arrives
        │
        ▼
Extract 28 Flow Features
(no decryption needed)
        │
        ▼
Normalize Features
(using training scale params)
        │
        ▼
Pass through Autoencoder
        │
        ▼
Compute Reconstruction MSE
        │
   ┌────┴────┐
   │         │
MSE ≤ T   MSE > T
   │         │
   ▼         ▼
BENIGN ✅  MALICIOUS 🚨
           Alert Generated
```

---

## 🔮 Future Work

- [ ] Real-time PCAP stream integration
- [ ] Browser extension for live DoH monitoring  
- [ ] Extend to detect HTTPS-based C2 traffic
- [ ] Federated learning for privacy-preserving training

## 🎯 Key Contributions

1. **Zero-Day Detection** — Detects novel DoH attacks without prior knowledge of attack signatures
2. **Privacy-Preserving** — Works on encrypted traffic without decryption
3. **Comprehensive Coverage** — Handles both DNS tunneling AND DGA-based attacks
4. **Real-Time Ready** — Millisecond inference time per flow
5. **Unsupervised** — No labeled attack data required for training

---

## 📚 References

- MontazeriShatoori et al., *"Detection of DoH Tunnels using Time-series Classification of Encrypted Traffic"*, IEEE Cyber SciTech 2020
- Salinas Monroy et al., *"Detection of Malicious DNS-over-HTTPS Traffic: An Anomaly Detection Approach using Autoencoders"*, arXiv 2023
- Moure-Garrido et al., *"Real time detection of malicious DoH traffic using statistical analysis"*, Computer Networks 2023
- [CIRA-CIC-DoHBrw-2020 Dataset](https://www.unb.ca/cic/datasets/dohbrw-2020.html)

---

## 👨‍💻 Author

**Aman Ansari** — Software Engineer | AI/ML Researcher

[![LinkedIn](https://img.shields.io/badge/LinkedIn-amanansari16-0077B5?style=flat&logo=linkedin)](https://linkedin.com/in/amanansari16)
[![GitHub](https://img.shields.io/badge/GitHub-Aman--1603-181717?style=flat&logo=github)](https://github.com/Aman-1603)
[![Portfolio](https://img.shields.io/badge/Portfolio-Visit-FF5722?style=flat&logo=vercel)](https://aman-protfolio-site-ejca.vercel.app/)

---

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:1a1a2e,50:16213e,100:0f3460&height=100&section=footer" width="100%"/>

Last update made on 21/08/2026
