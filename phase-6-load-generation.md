# Phase 6: Load Generation

## Overview
Create scripts to generate realistic traffic patterns for different data plane environments, enabling effective demonstration of the observability platform.

---

## Step 1: Create Load Generator Directory

### 1.1 Setup Directory Structure
```bash
mkdir -p load-generator
cd load-generator
```

---

## Step 2: Create Python Load Generator

### 2.1 Create `requirements.txt`
```txt
requests==2.31.0
click==8.1.7
colorama==0.4.6
```

### 2.2 Create `load_generator.py`
```python
#!/usr/bin/env python3
"""
OTEL PoC Load Generator
Generates realistic HTTP traffic to API endpoints with configurable patterns
"""

import requests
import time
import random
import json
import click
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime
from colorama import Fore, Style, init

# Initialize colorama
init(autoreset=True)

class LoadGenerator:
    def __init__(self, base_url, environment="dev"):
        self.base_url = base_url.rstrip('/')
        self.environment = environment
        self.session = requests.Session()
        self.stats = {
            'total_requests': 0,
            'successful_requests': 0,
            'failed_requests': 0,
            'total_latency': 0.0,
            'errors': []
        }

        # Endpoint configurations with weights and expected behavior
        self.endpoints = {
            '/health': {
                'weight': 40,
                'method': 'GET',
                'expected_latency': 0.02,  # 20ms
                'error_rate': 0.0
            },
            '/api/products': {
                'weight': 30,
                'method': 'GET',
                'expected_latency': 0.05,  # 50ms
                'error_rate': 0.01
            },
            '/api/orders': {
                'weight': 15,
                'method': 'POST',
                'expected_latency': 0.15,  # 150ms
                'error_rate': 0.02,
                'payload': {
                    'product_id': lambda: str(random.randint(1, 5)),
                    'quantity': lambda: random.randint(1, 10)
                }
            },
            '/api/reports': {
                'weight': 8,
                'method': 'GET',
                'expected_latency': 1.5,  # 1.5s
                'error_rate': 0.03
            },
            '/api/search': {
                'weight': 5,
                'method': 'GET',
                'expected_latency': 0.2,  # 200ms
                'error_rate': 0.01,
                'params': {
                    'q': lambda: random.choice(['laptop', 'mouse', 'keyboard', 'monitor', 'headphones'])
                }
            },
            '/api/payments': {
                'weight': 2,
                'method': 'POST',
                'expected_latency': 0.25,  # 250ms
                'error_rate': 0.05,
                'payload': {
                    'order_id': lambda: f"order-{random.randint(1000, 9999)}",
                    'amount': lambda: round(random.uniform(10.0, 1000.0), 2)
                }
            }
        }

    def _select_endpoint(self):
        """Select endpoint based on weights"""
        endpoints = list(self.endpoints.keys())
        weights = [self.endpoints[ep]['weight'] for ep in endpoints]
        return random.choices(endpoints, weights=weights)[0]

    def _build_payload(self, config):
        """Build request payload from config"""
        if 'payload' not in config:
            return None

        payload = {}
        for key, value_func in config['payload'].items():
            payload[key] = value_func() if callable(value_func) else value_func
        return payload

    def _build_params(self, config):
        """Build URL parameters from config"""
        if 'params' not in config:
            return None

        params = {}
        for key, value_func in config['params'].items():
            params[key] = value_func() if callable(value_func) else value_func
        return params

    def make_request(self):
        """Make a single HTTP request"""
        endpoint = self._select_endpoint()
        config = self.endpoints[endpoint]
        url = f"{self.base_url}{endpoint}"

        try:
            start_time = time.time()

            if config['method'] == 'GET':
                params = self._build_params(config)
                response = self.session.get(url, params=params, timeout=10)
            elif config['method'] == 'POST':
                payload = self._build_payload(config)
                response = self.session.post(
                    url,
                    json=payload,
                    headers={'Content-Type': 'application/json'},
                    timeout=10
                )
            else:
                raise ValueError(f"Unsupported method: {config['method']}")

            latency = time.time() - start_time

            self.stats['total_requests'] += 1
            self.stats['total_latency'] += latency

            if response.status_code < 400:
                self.stats['successful_requests'] += 1
                status_color = Fore.GREEN
            else:
                self.stats['failed_requests'] += 1
                status_color = Fore.RED
                self.stats['errors'].append({
                    'endpoint': endpoint,
                    'status_code': response.status_code,
                    'timestamp': datetime.now().isoformat()
                })

            # Log request
            print(f"{status_color}[{self.environment.upper()}] "
                  f"{config['method']:4} {endpoint:20} "
                  f"Status: {response.status_code:3} "
                  f"Latency: {latency*1000:6.1f}ms{Style.RESET_ALL}")

            return {
                'endpoint': endpoint,
                'status_code': response.status_code,
                'latency': latency,
                'success': response.status_code < 400
            }

        except Exception as e:
            self.stats['total_requests'] += 1
            self.stats['failed_requests'] += 1
            self.stats['errors'].append({
                'endpoint': endpoint,
                'error': str(e),
                'timestamp': datetime.now().isoformat()
            })

            print(f"{Fore.RED}[{self.environment.upper()}] "
                  f"ERROR {endpoint:20} {str(e)}{Style.RESET_ALL}")

            return {
                'endpoint': endpoint,
                'error': str(e),
                'success': False
            }

    def print_stats(self):
        """Print statistics"""
        total = self.stats['total_requests']
        if total == 0:
            print("No requests made yet")
            return

        success_rate = (self.stats['successful_requests'] / total) * 100
        avg_latency = (self.stats['total_latency'] / total) * 1000  # in ms

        print(f"\n{Fore.CYAN}{'='*60}")
        print(f"Load Generator Statistics - {self.environment.upper()}")
        print(f"{'='*60}{Style.RESET_ALL}")
        print(f"Total Requests:      {total}")
        print(f"Successful:          {self.stats['successful_requests']} "
              f"({success_rate:.1f}%)")
        print(f"Failed:              {self.stats['failed_requests']} "
              f"({100-success_rate:.1f}%)")
        print(f"Average Latency:     {avg_latency:.2f}ms")

        if self.stats['errors']:
            print(f"\n{Fore.YELLOW}Recent Errors (last 5):{Style.RESET_ALL}")
            for error in self.stats['errors'][-5:]:
                print(f"  - {error}")
        print()


def run_load_test(base_url, environment, rate, duration, workers):
    """Run load test with specified parameters"""
    generator = LoadGenerator(base_url, environment)

    print(f"{Fore.CYAN}Starting load test...{Style.RESET_ALL}")
    print(f"Target:       {base_url}")
    print(f"Environment:  {environment}")
    print(f"Rate:         {rate} req/sec")
    print(f"Duration:     {duration} seconds")
    print(f"Workers:      {workers}")
    print(f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}\n")

    start_time = time.time()
    end_time = start_time + duration
    request_interval = 1.0 / rate if rate > 0 else 0

    with ThreadPoolExecutor(max_workers=workers) as executor:
        futures = []

        while time.time() < end_time:
            # Submit request
            future = executor.submit(generator.make_request)
            futures.append(future)

            # Sleep to maintain rate
            if request_interval > 0:
                time.sleep(request_interval)

        # Wait for all requests to complete
        for future in as_completed(futures):
            future.result()

    # Print final statistics
    generator.print_stats()


@click.command()
@click.option('--target', '-t', default='http://localhost:8080',
              help='Target API base URL')
@click.option('--environment', '-e', default='dev',
              type=click.Choice(['prod', 'dev', 'qa']),
              help='Environment name (affects logging)')
@click.option('--rate', '-r', default=10, type=int,
              help='Requests per second')
@click.option('--duration', '-d', default=60, type=int,
              help='Duration in seconds')
@click.option('--workers', '-w', default=10, type=int,
              help='Number of concurrent workers')
def main(target, environment, rate, duration, workers):
    """
    OTEL PoC Load Generator

    Examples:

    \b
    # Generate 10 req/sec for 60 seconds to dev environment
    python load_generator.py -t http://localhost:8082 -e dev -r 10 -d 60

    \b
    # Generate 50 req/sec for 300 seconds to prod environment
    python load_generator.py -t http://localhost:8081 -e prod -r 50 -d 300

    \b
    # High load test: 100 req/sec with 50 workers
    python load_generator.py -t http://localhost:8081 -r 100 -w 50
    """
    try:
        run_load_test(target, environment, rate, duration, workers)
    except KeyboardInterrupt:
        print(f"\n{Fore.YELLOW}Load test interrupted by user{Style.RESET_ALL}")
    except Exception as e:
        print(f"\n{Fore.RED}Error: {e}{Style.RESET_ALL}")
        raise


if __name__ == '__main__':
    main()
```

---

## Step 3: Create Preset Load Patterns

### 3.1 Create `load_patterns.py`
```python
#!/usr/bin/env python3
"""
Predefined load patterns for different scenarios
"""

import subprocess
import time
import sys
from concurrent.futures import ThreadPoolExecutor

# Load pattern definitions
PATTERNS = {
    'steady': {
        'description': 'Steady baseline load',
        'prod': {'rate': 50, 'duration': 300},
        'dev': {'rate': 20, 'duration': 300},
        'qa': {'rate': 15, 'duration': 300}
    },
    'burst': {
        'description': 'Burst traffic pattern',
        'phases': [
            {'rate': 10, 'duration': 30},
            {'rate': 100, 'duration': 30},
            {'rate': 10, 'duration': 30},
        ]
    },
    'spike': {
        'description': 'Sudden traffic spike',
        'phases': [
            {'rate': 20, 'duration': 60},
            {'rate': 200, 'duration': 30},
            {'rate': 20, 'duration': 60},
        ]
    },
    'ramp': {
        'description': 'Gradual ramp up',
        'phases': [
            {'rate': 10, 'duration': 60},
            {'rate': 25, 'duration': 60},
            {'rate': 50, 'duration': 60},
            {'rate': 100, 'duration': 60},
            {'rate': 50, 'duration': 60},
        ]
    }
}

ENVIRONMENTS = {
    'prod': 'http://localhost:8081',
    'dev': 'http://localhost:8082',
    'qa': 'http://localhost:8083'
}


def run_load_generator(target, environment, rate, duration):
    """Run load generator subprocess"""
    cmd = [
        'python3', 'load_generator.py',
        '-t', target,
        '-e', environment,
        '-r', str(rate),
        '-d', str(duration)
    ]

    print(f"\n🚀 Starting load for {environment.upper()}: "
          f"{rate} req/sec for {duration}s")

    try:
        subprocess.run(cmd, check=True)
    except subprocess.CalledProcessError as e:
        print(f"❌ Error running load generator for {environment}: {e}")
    except KeyboardInterrupt:
        print(f"\n⚠️  Load test for {environment} interrupted")


def run_steady_pattern():
    """Run steady load across all environments"""
    print("📊 Running STEADY load pattern")
    print("=" * 60)

    pattern = PATTERNS['steady']

    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = []
        for env, config in pattern.items():
            if env == 'description':
                continue
            target = ENVIRONMENTS[env]
            futures.append(
                executor.submit(
                    run_load_generator,
                    target, env, config['rate'], config['duration']
                )
            )

        # Wait for all to complete
        for future in futures:
            future.result()


def run_burst_pattern(environment='prod'):
    """Run burst pattern on specific environment"""
    print(f"💥 Running BURST pattern on {environment.upper()}")
    print("=" * 60)

    target = ENVIRONMENTS[environment]
    pattern = PATTERNS['burst']

    for i, phase in enumerate(pattern['phases'], 1):
        print(f"\n--- Phase {i}/{len(pattern['phases'])} ---")
        run_load_generator(
            target, environment, phase['rate'], phase['duration']
        )


def run_spike_pattern(environment='prod'):
    """Run spike pattern on specific environment"""
    print(f"📈 Running SPIKE pattern on {environment.upper()}")
    print("=" * 60)

    target = ENVIRONMENTS[environment]
    pattern = PATTERNS['spike']

    for i, phase in enumerate(pattern['phases'], 1):
        print(f"\n--- Phase {i}/{len(pattern['phases'])} ---")
        run_load_generator(
            target, environment, phase['rate'], phase['duration']
        )


def run_ramp_pattern(environment='prod'):
    """Run ramp-up pattern on specific environment"""
    print(f"📊 Running RAMP pattern on {environment.upper()}")
    print("=" * 60)

    target = ENVIRONMENTS[environment]
    pattern = PATTERNS['ramp']

    for i, phase in enumerate(pattern['phases'], 1):
        print(f"\n--- Phase {i}/{len(pattern['phases'])} ---")
        run_load_generator(
            target, environment, phase['rate'], phase['duration']
        )


if __name__ == '__main__':
    import sys

    if len(sys.argv) < 2:
        print("Usage: python load_patterns.py <pattern> [environment]")
        print("\nAvailable patterns:")
        for name, config in PATTERNS.items():
            print(f"  - {name}: {config['description']}")
        print("\nEnvironments: prod, dev, qa")
        print("\nExamples:")
        print("  python load_patterns.py steady")
        print("  python load_patterns.py burst prod")
        print("  python load_patterns.py spike dev")
        sys.exit(1)

    pattern = sys.argv[1]
    environment = sys.argv[2] if len(sys.argv) > 2 else 'prod'

    if pattern == 'steady':
        run_steady_pattern()
    elif pattern == 'burst':
        run_burst_pattern(environment)
    elif pattern == 'spike':
        run_spike_pattern(environment)
    elif pattern == 'ramp':
        run_ramp_pattern(environment)
    else:
        print(f"Unknown pattern: {pattern}")
        sys.exit(1)
```

---

## Step 4: Create Bash Load Generator (Alternative)

### 4.1 Create `load_generator.sh`
```bash
#!/bin/bash
# Simple bash-based load generator
# Useful for environments without Python

set -e

# Configuration
TARGET_URL="${1:-http://localhost:8080}"
DURATION="${2:-60}"
RATE="${3:-10}"
ENVIRONMENT="${4:-dev}"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
NC='\033[0m' # No Color

# Statistics
TOTAL_REQUESTS=0
SUCCESSFUL_REQUESTS=0
FAILED_REQUESTS=0

# Endpoints with weights
declare -A ENDPOINTS
ENDPOINTS=(
    ["/health"]=40
    ["/api/products"]=30
    ["/api/search?q=laptop"]=15
    ["/api/reports"]=10
    ["/api/orders"]=5
)

# Function to select random endpoint based on weights
select_endpoint() {
    local rand=$((RANDOM % 100))
    local cumulative=0

    for endpoint in "${!ENDPOINTS[@]}"; do
        cumulative=$((cumulative + ENDPOINTS[$endpoint]))
        if [ $rand -lt $cumulative ]; then
            echo "$endpoint"
            return
        fi
    done

    echo "/health"
}

# Function to make request
make_request() {
    local endpoint=$(select_endpoint)
    local url="${TARGET_URL}${endpoint}"
    local start_time=$(date +%s%N)

    if [[ "$endpoint" == *"orders"* ]] || [[ "$endpoint" == *"payments"* ]]; then
        # POST request
        response=$(curl -s -w "\n%{http_code}" -X POST "$url" \
            -H "Content-Type: application/json" \
            -d '{"product_id":"1","quantity":2}' \
            --max-time 10 2>&1)
    else
        # GET request
        response=$(curl -s -w "\n%{http_code}" "$url" --max-time 10 2>&1)
    fi

    local end_time=$(date +%s%N)
    local latency=$(( (end_time - start_time) / 1000000 ))
    local status_code=$(echo "$response" | tail -1)

    TOTAL_REQUESTS=$((TOTAL_REQUESTS + 1))

    if [ "$status_code" -ge 200 ] && [ "$status_code" -lt 400 ]; then
        SUCCESSFUL_REQUESTS=$((SUCCESSFUL_REQUESTS + 1))
        echo -e "${GREEN}[$ENVIRONMENT] GET $endpoint Status: $status_code Latency: ${latency}ms${NC}"
    else
        FAILED_REQUESTS=$((FAILED_REQUESTS + 1))
        echo -e "${RED}[$ENVIRONMENT] GET $endpoint Status: $status_code Latency: ${latency}ms${NC}"
    fi
}

# Print statistics
print_stats() {
    local success_rate=0
    if [ $TOTAL_REQUESTS -gt 0 ]; then
        success_rate=$((SUCCESSFUL_REQUESTS * 100 / TOTAL_REQUESTS))
    fi

    echo -e "\n${CYAN}========================================${NC}"
    echo -e "${CYAN}Load Generator Statistics - ${ENVIRONMENT^^}${NC}"
    echo -e "${CYAN}========================================${NC}"
    echo "Total Requests:      $TOTAL_REQUESTS"
    echo "Successful:          $SUCCESSFUL_REQUESTS ($success_rate%)"
    echo "Failed:              $FAILED_REQUESTS"
    echo ""
}

# Trap Ctrl+C
trap print_stats EXIT

# Main loop
echo -e "${CYAN}Starting load test...${NC}"
echo "Target:       $TARGET_URL"
echo "Environment:  $ENVIRONMENT"
echo "Rate:         $RATE req/sec"
echo "Duration:     $DURATION seconds"
echo -e "${CYAN}========================================${NC}\n"

END_TIME=$(($(date +%s) + DURATION))
INTERVAL=$(echo "scale=3; 1/$RATE" | bc)

while [ $(date +%s) -lt $END_TIME ]; do
    make_request &
    sleep "$INTERVAL"
done

# Wait for background jobs
wait

print_stats
```

### 4.2 Make Script Executable
```bash
chmod +x load_generator.sh
```

---

## Step 5: Create Docker Container for Load Generator

### 5.1 Create `Dockerfile`
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy scripts
COPY load_generator.py .
COPY load_patterns.py .

# Make scripts executable
RUN chmod +x load_generator.py load_patterns.py

ENTRYPOINT ["python", "load_generator.py"]
CMD ["--help"]
```

### 5.2 Build Docker Image
```bash
docker build -t otel-load-generator:latest .
```

---

## Step 6: Integration with Docker Compose

### 6.1 Add to `docker-compose.yml` (Optional)
```yaml
  # Load Generator (optional, for automated testing)
  load-generator:
    build:
      context: ./load-generator
      dockerfile: Dockerfile
    image: otel-load-generator:latest
    container_name: load-generator
    networks:
      - otel-network
    depends_on:
      - api-prod
      - api-dev
      - api-qa
    command: >
      -t http://api-prod:8080
      -e prod
      -r 50
      -d 3600
    restart: "no"  # Run once
```

---

## Step 7: Create Demo Scenarios

### 7.1 Create `scenarios/demo_scenario_1.sh`
```bash
#!/bin/bash
# Scenario 1: Normal Operations
# All environments running with typical load

echo "🎬 Demo Scenario 1: Normal Operations"
echo "======================================"
echo ""
echo "This scenario simulates normal operations across all environments:"
echo "  - Prod: Steady 50 req/sec"
echo "  - Dev: Moderate 20 req/sec"
echo "  - QA: Light 10 req/sec"
echo ""
echo "Duration: 5 minutes"
echo ""

python3 load_patterns.py steady
```

### 7.2 Create `scenarios/demo_scenario_2.sh`
```bash
#!/bin/bash
# Scenario 2: Production Incident
# Spike in prod traffic, normal elsewhere

echo "🎬 Demo Scenario 2: Production Incident"
echo "========================================"
echo ""
echo "Simulating a production traffic spike while dev/qa remain normal"
echo ""

# Start background load on dev and qa
python3 load_generator.py -t http://localhost:8082 -e dev -r 20 -d 300 &
python3 load_generator.py -t http://localhost:8083 -e qa -r 10 -d 300 &

# Run spike pattern on prod
python3 load_patterns.py spike prod

wait
```

### 7.3 Create `scenarios/demo_scenario_3.sh`
```bash
#!/bin/bash
# Scenario 3: Load Testing QA
# Heavy load on QA for testing

echo "🎬 Demo Scenario 3: Load Testing QA"
echo "===================================="
echo ""
echo "Simulating load testing in QA environment"
echo ""

python3 load_patterns.py ramp qa
```

### 7.4 Make Scenarios Executable
```bash
chmod +x scenarios/*.sh
```

---

## Step 8: Testing

### 8.1 Install Python Dependencies
```bash
cd load-generator
pip install -r requirements.txt
```

### 8.2 Test Basic Load Generator
```bash
# Generate load to dev environment
python load_generator.py \
  --target http://localhost:8082 \
  --environment dev \
  --rate 10 \
  --duration 30

# Short test
python load_generator.py -t http://localhost:8081 -e prod -r 5 -d 10
```

### 8.3 Test Load Patterns
```bash
# Steady load across all environments
python load_patterns.py steady

# Burst pattern on prod
python load_patterns.py burst prod

# Spike pattern on dev
python load_patterns.py spike dev
```

### 8.4 Test Bash Generator
```bash
./load_generator.sh http://localhost:8081 60 10 prod
```

### 8.5 Test Docker Container
```bash
docker run --rm --network otel-network \
  otel-load-generator:latest \
  -t http://api-prod:8080 \
  -e prod \
  -r 10 \
  -d 30
```

---

## Step 9: Create README

### 9.1 Create `load-generator/README.md`
```markdown
# Load Generator

Generate realistic HTTP traffic for OTEL PoC demo.

## Quick Start

### Python Version
```bash
# Install dependencies
pip install -r requirements.txt

# Run basic load test
python load_generator.py -t http://localhost:8081 -e prod -r 50 -d 300

# Run predefined pattern
python load_patterns.py steady
```

### Bash Version
```bash
./load_generator.sh http://localhost:8081 60 10 prod
```

### Docker Version
```bash
docker run --rm --network otel-network \
  otel-load-generator:latest \
  -t http://api-prod:8080 -e prod -r 50 -d 300
```

## Load Patterns

- **steady**: Consistent load across all environments
- **burst**: Periodic traffic bursts
- **spike**: Sudden traffic spike
- **ramp**: Gradual ramp-up and ramp-down

## Demo Scenarios

```bash
./scenarios/demo_scenario_1.sh  # Normal operations
./scenarios/demo_scenario_2.sh  # Production incident
./scenarios/demo_scenario_3.sh  # QA load testing
```

## Options

- `-t, --target`: Target API URL (default: http://localhost:8080)
- `-e, --environment`: Environment name (prod, dev, qa)
- `-r, --rate`: Requests per second (default: 10)
- `-d, --duration`: Duration in seconds (default: 60)
- `-w, --workers`: Concurrent workers (default: 10)
```

---

## Expected Outcomes

✅ **Realistic Traffic**:
- Weighted endpoint distribution
- Variable latency patterns
- Controlled error rates

✅ **Multiple Patterns**:
- Steady baseline load
- Burst traffic
- Spike scenarios
- Ramp-up/down

✅ **Demo-Ready**:
- Colorized output
- Real-time statistics
- Pre-configured scenarios
- Easy to run

---

## Troubleshooting

**Issue: Connection refused**
```bash
# Verify API is running
curl http://localhost:8081/health

# Check docker network
docker network inspect otel-network
```

**Issue: Python dependencies missing**
```bash
pip install -r requirements.txt
```

**Issue: High error rate**
```bash
# Check API logs
docker logs api-prod

# Reduce load rate
python load_generator.py -r 5 -d 30
```

---

## Next Steps

Proceed to **Phase 7: Testing & Validation** to systematically validate the entire PoC.
