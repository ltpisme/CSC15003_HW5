# Headless Execution & Resource Monitoring Commands Template

## 1. Automated JMeter Headless CLI Execution Commands

### A. Load Test Execution
```bash
# 1. Clean previous results
rm -rf results/load/raw.jtl results/load/html results/load/jmeter.log

# 2. Execute headless test plan with dynamic CLI parameters
jmeter -n \
  -t jmeter/<SCENARIO_LOAD>.jmx \
  -Jhost=<TARGET_HOST> \
  -Jport=<TARGET_PORT> \
  -Jthreads=20 \
  -Jrampup=30 \
  -Jduration=180 \
  -l results/load/raw.jtl \
  -j results/load/jmeter.log \
  -e -o results/load/html
```

### B. Stress Test Execution
```bash
rm -rf results/stress/raw.jtl results/stress/html results/stress/jmeter.log
jmeter -n \
  -t jmeter/<SCENARIO_STRESS>.jmx \
  -Jhost=<TARGET_HOST> \
  -Jport=<TARGET_PORT> \
  -l results/stress/raw.jtl \
  -j results/stress/jmeter.log \
  -e -o results/stress/html
```

### C. Spike Test Execution
```bash
rm -rf results/spike/raw.jtl results/spike/html results/spike/jmeter.log
jmeter -n \
  -t jmeter/<SCENARIO_SPIKE>.jmx \
  -Jhost=<TARGET_HOST> \
  -Jport=<TARGET_PORT> \
  -l results/spike/raw.jtl \
  -j results/spike/jmeter.log \
  -e -o results/spike/html
```

### D. Endurance Test Execution
```bash
rm -rf results/endurance/raw.jtl results/endurance/html results/endurance/jmeter.log
jmeter -n \
  -t jmeter/<SCENARIO_LOAD>.jmx \
  -Jhost=<TARGET_HOST> \
  -Jport=<TARGET_PORT> \
  -Jthreads=20 \
  -Jrampup=30 \
  -Jduration=900 \
  -Jrun_id=endurance \
  -l results/endurance/raw.jtl \
  -j results/endurance/jmeter.log \
  -e -o results/endurance/html
```

---

## 2. Resource Monitoring & Evidence Capture Instructions

1. **System Resource Monitoring:**
   - Launch terminal with `htop -p <PROCESS_PID>` or `top` filtering on backend server process.
   - Observe per-core CPU load, Memory RSS drift, and process thread count.
2. **Side-by-Side Evidence Screenshot:**
   - Arrange terminal split screen: Left window displaying foreground JMeter execution progress; Right window displaying `htop` CPU/Memory resource monitor.
   - Capture screenshot at steady-state peak load and save to `evidence/<scenario>/<scenario>_jmeter_resource.png`.
3. **Hardware Spec Verification:**
   - Execute `hostname`, `uname -a`, `lscpu`, `free -h` and capture screenshot to `evidence/hardware/hardware_hostname.png`.
