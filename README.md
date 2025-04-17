# Pantheon PA3 - Congestion Control Comparison

**Course:** CSCI 5300 - Computer Networks  
**Student:** Harshil Sharma  
**Platform:** Ubuntu 20.04 (VirtualBox)  
**Submission Date:** April 15, 2025

---

## 📌 Overview

This project uses the [Pantheon](https://github.com/StanfordSNR/pantheon) framework to evaluate and compare congestion control algorithms under varied network conditions using [Mahimahi](https://github.com/ravinet/mahimahi).

---

## ⚙️ Environment Setup (Part A)

### Step 1: Clone Pantheon

```
git clone https://github.com/StanfordSNR/pantheon.git
cd pantheon
```

### Step 2: Install Dependencies

```
sudo apt update
sudo apt install curl -y
./tools/fetch_submodules.sh
./tools/install_deps.sh
```

### Step 3: Build Pantheon Tunnel

```
cd third_party/pantheon-tunnel
./autogen.sh
./configure
make clean
make
```

### Step 4: Clone & Build Mahimahi

```
git clone https://github.com/ravinet/mahimahi.git
cd mahimahi

sudo apt-get install build-essential cmake autoconf automake libtool \
libprotobuf-dev protobuf-compiler libgoogle-glog-dev \
libcurl4-openssl-dev libpcap-dev libssl-dev

mkdir build && cd build
cmake ..
make
sudo make install
```

---

## 🚦 Protocols Used

- **Part A:** `cubic`, `bbr`
- **Part B & C:** `vegas`, `cubic`, `vivace`

---

## 🧪 Running Experiments

### Initial Setup and Basic Test

```
sudo python2 src/experiments/setup.py --setup --all
sudo python2 src/experiments/test.py local --schemes "vegas cubic vivace"
```

### Create Custom Network Traces

```
seq 1 60 | awk '{print 50}' > src/experiments/50mbps.trace
seq 1 60 | awk '{print 1}'  > src/experiments/1mbps.trace
```

### Run Low Latency - High Bandwidth Scenario (50 Mbps, 5 ms)

```
python2 src/experiments/test.py local \
  --schemes "vegas cubic vivace" \
  --run-times 1 \
  --runtime 60 \
  --data-dir result/part_low_latency \
  --uplink-trace src/experiments/50mbps.trace \
  --downlink-trace src/experiments/50mbps.trace \
  --prepend-mm-cmds "mm-delay 5"
```

### Run High Latency - Low Bandwidth Scenario (1 Mbps, 100 ms)

```
python2 src/experiments/test.py local \
  --schemes "vegas cubic vivace" \
  --run-times 1 \
  --runtime 60 \
  --data-dir result/part_high_latency \
  --uplink-trace src/experiments/1mbps.trace \
  --downlink-trace src/experiments/1mbps.trace \
  --prepend-mm-cmds "mm-delay 100"
```

---

## 📊 Part C – Data Analysis & RTT Graphs

### Script to Generate RTT Graphs and Summary Table

```
mkdir -p result/part_c && \
echo -e "Scenario\tScheme\tAvg_RTT(ms)\tRTT_95th(ms)" > result/part_c/summary_metrics.tsv && \
for SCENARIO in result/part_low_latency result/part_high_latency; do \
  if [[ "$SCENARIO" == *"low"* ]]; then LABEL="50Mbps_10ms"; else LABEL="1Mbps_200ms"; fi; \
  for SCHEME in vegas cubic vivace; do \
    LOG="${SCENARIO}/${SCHEME}_datalink_run1.log"; \
    OUTDIR="result/part_c"; \
    CLEANED="${OUTDIR}/${SCHEME}_${LABEL}_cleaned.dat"; \
    if [ -f "$LOG" ]; then \
      awk '!/^#/ {print NR, $1}' $LOG > $CLEANED; \
      gnuplot -e "set terminal png size 800,600; \
      set output '${OUTDIR}/${SCHEME}_${LABEL}_rtt.png'; \
      set title '${SCHEME^} RTT - ${LABEL}'; \
      set xlabel 'Packet Index'; set ylabel 'RTT (ms)'; \
      plot '${CLEANED}' using 1:2 with lines title '${SCHEME}'"; \
      AVG_RTT=$(awk '{s+=$2} END {print (NR>0)?s/NR:0}' $CLEANED); \
      RTT_95=$(awk '{print $2}' $CLEANED | sort -n | awk '{a[NR]=$1} END {print a[int(NR*0.95)]}'); \
      echo -e "${LABEL}\t${SCHEME}\t${AVG_RTT}\t${RTT_95}" >> ${OUTDIR}/summary_metrics.tsv; \
    else echo "Missing log: $LOG"; fi; \
  done; \
done && \
gnuplot -persist <<EOF
set terminal png size 800,600
set output 'result/part_c/scatter_rtt_all.png'
set title 'RTT 95th vs Avg RTT (All Schemes)'
set xlabel 'RTT 95th Percentile (ms)'
set ylabel 'Avg RTT (ms)'
set xrange [*:*] reverse
set grid
plot 'result/part_c/summary_metrics.tsv' using 4:3:2 with labels point pt 7 offset char 1,1 notitle
EOF
```

---

## 📈 Graph Outputs Summary

- **Total Graphs Generated:** 7  
  - 6 time-series RTT graphs (one for each scheme × scenario)
  - 1 scatter plot showing 95th percentile RTT vs average RTT

| Scheme  | 50Mbps / 10ms | 1Mbps / 200ms |
|---------|----------------|----------------|
| Vegas   | ✅              | ✅              |
| Cubic   | ✅              | ✅              |
| Vivace  | ✅              | ✅              |

---

## 📁 Directory Structure

```
pantheon/
├── result/
│   ├── part_low_latency/
│   ├── part_high_latency/
│   └── part_c/
│       ├── *_cleaned.dat
│       ├── *_rtt.png
│       ├── summary_metrics.tsv
│       └── scatter_rtt_all.png
├── src/
│   ├── experiments/
│   │   ├── test.py
│   │   ├── 50mbps.trace
│   │   └── 1mbps.trace
│   └── analysis/
│       └── analyze.py
```

---

## 📧 Contact

If reproduction fails or you're missing a dependency, feel free to reach out or raise an issue.
