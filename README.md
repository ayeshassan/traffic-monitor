# AI-Powered Network Traffic Monitor

A real-time network intrusion detection tool that captures live traffic, extracts security-relevant features, and uses a machine learning model to flag potentially malicious activity, all visualized in a live web dashboard.

## Why I built this

Most tech fields involve the same kind of computer work, but cybersecurity feels different to me, there's real meaning behind it. Data drives everything in the world now, it's even the reason AI exists, these tools learned everything by training on massive datasets. So protecting that data, especially sensitive data, feels like genuinely important work, not just code for its own sake.

I built this project because it's a baseline skill in my field. As a university student, I obviously can't do real penetration testing or active hacking, but analyzing packets and traffic gave me a real taste of what defensive security work actually feels like. I combined that with machine learning, training a model to distinguish between normal traffic and attacks, since security teams deal with a constant stream of attempted attacks every day, and a large portion of them are automated bots rather than targeted human attackers. Being able to build even a small-scale system that helps catch that automatically felt like a genuinely useful first step into the field.

## What it does

This project builds an end-to-end pipeline that mirrors (at a small scale) how real intrusion detection systems work:

1. **Captures live packets** off the network using Scapy/Npcap
2. **Extracts features** from that traffic (protocol, ports, packet size, and connection behavior over a rolling time window)
3. **Classifies traffic** as normal or potentially malicious using a Random Forest model trained on the NSL-KDD intrusion detection dataset
4. **Displays results live** in a Flask web dashboard, showing stats and a real-time event feed

## Tech stack

- **Python**: Scapy (packet capture), pandas (data processing), scikit-learn (ML model), Flask (dashboard)
- **Dataset**: NSL-KDD, a standard benchmark dataset for network intrusion detection research
- **Model**: Random Forest classifier (200 trees, class-balanced)

## Results

- Trained on 125,973 labeled connection records, tested on 22,544 unseen records
- **77.75% accuracy** on the held-out test set
- **97% precision** on attack classification (very few false alarms when it does flag something)
- **63% recall** on attack classification (misses some attacks, particularly novel attack types not seen in training)

## Key technical challenge & what I learned

The biggest challenge was translating live packet-level data into the same feature space the model was trained on. NSL-KDD's most predictive features (like `diff_srv_rate`, the fraction of recent connections going to different services) are based on full connection/flow statistics, not single packets. I addressed this by building a rolling 2-second connection-history window to approximate these features in real time.

During testing, I confirmed this pipeline correctly detects the *behavioral signature* of a port scan, `diff_srv_rate` correctly spikes to ~0.90 during a simulated Nmap scan versus ~0.0 during normal browsing, even though the current model doesn't yet have enough approximated features (like TCP error rates and full session duration) to independently confirm the classification as "attack." This reflects a genuine, known limitation in real-world ML-based IDS: **full flow-based feature extraction (proper TCP session tracking) meaningfully outperforms packet-level approximation**, and would be my main next step to improve detection accuracy.

## Project structure

```
traffic-monitor/
├── test_capture.py          # Initial packet capture test
├── feature_extract.py       # Extracts structured features from live packets
├── train_model.py           # Trains the Random Forest model on NSL-KDD
├── live_detect_v1_basic.py  # First version: basic per-connection counters
├── live_detect_v2.py        # Real-time detection with rolling-window features
├── dashboard.py             # Flask web dashboard (main demo)
├── visualize_traffic.py     # Matplotlib visualizations of captured traffic
├── Field Names.csv          # NSL-KDD column names
├── sample_traffic.csv       # Anonymised sample of a real capture
├── requirements.txt         # Python dependencies
└── README.md
```

The trained model files (`*.pkl`) and the NSL-KDD dataset are not stored in this repo. You generate the model yourself in step 3 below.

## Setup and running it

**1. Install the dependencies**

```bash
pip install -r requirements.txt
```

Live capture on Windows also needs [Npcap](https://npcap.com/) installed.

**2. Get the dataset**

Download the NSL-KDD dataset ([official page](https://www.unb.ca/cic/datasets/nsl.html)). The files are also available from [this GitHub repo](https://github.com/Jehuty4949/NSL_KDD). Save `KDDTrain+.txt` and `KDDTest+.txt` in the project folder.

**3. Train the model**

```bash
python train_model.py
```

This prints the accuracy report and creates `traffic_model.pkl`, `encoders.pkl` and `feature_cols.pkl`. The model is built locally on purpose: pickle files can run code when loaded, so only load ones you created yourself.

**4. Start the dashboard**

```bash
python dashboard.py
```

Then open `http://127.0.0.1:5000` in your browser. Packet capture needs admin/root privileges.

**Optional: capture your own traffic and chart it**

```bash
python feature_extract.py     # saves captured_traffic.csv
python visualize_traffic.py   # saves the two charts
```

`sample_traffic.csv` is an anonymised sample of a real capture (IP addresses replaced), included so you can see the data format without capturing anything.

## Responsible use

Only capture traffic on networks you own or have explicit permission to monitor.

## Future improvements

- Full TCP session/flow tracking for more accurate feature extraction (error rates, connection duration)
- Multi-class classification (specific attack types, not just binary normal/attack)
- Email/Slack alerting for flagged events
- Docker containerization for easier deployment
