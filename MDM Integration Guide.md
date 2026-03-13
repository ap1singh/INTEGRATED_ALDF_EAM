# MDM Integration Guide

**ALDF-EAM native integration with Oracle Utilities MDM, Itron OpenWay, and Landis+Gyr Gridstream**

---

## Overview

This guide shows how to integrate ALDF-EAM with major Meter Data Management (MDM) platforms. The framework is designed for **zero-disruption integration** — it reads from your existing MDM, processes data, and writes results back without modifying core MDM functionality.

### Architecture Pattern

```
┌──────────────────┐
│   MDM System     │
│  (Oracle/Itron/  │
│   Landis+Gyr)    │
└────────┬─────────┘
         │
         │ Extract AMI data
         │ (interval reads, events, topology)
         ▼
┌─────────────────────┐
│   ALDF-EAM Engine   │
│                     │
│  1. Compute S-vector│
│  2. Select methods  │
│  3. Run analysis    │
│  4. Generate scores │
└────────┬────────────┘
         │
         │ Write results
         │ (suspects, confidence, indicators)
         ▼
┌──────────────────────┐
│  MDM Results Table   │
│  OR                  │
│  Field Dispatch API  │
└──────────────────────┘
```

### Integration Options

| Option | Deployment | Latency | Effort | Best For |
|--------|-----------|---------|--------|----------|
| **1. REST API** | ALDF-EAM as microservice | ~2-5s | Low | Quick pilot |
| **2. Native Module** | Inside MDM (Oracle PL/SQL) | <500ms | High | Production scale |
| **3. Batch ETL** | Scheduled jobs (cron/Task) | Hours | Medium | Large utilities |

---

## 1. Oracle Utilities MDM Integration

### Architecture

Oracle MDM stores AMI data in Oracle Database (typically 11g-19c). ALDF-EAM can:
- **Option A:** Query Oracle DB directly (SQLAlchemy + cx_Oracle)
- **Option B:** Use Oracle APEX REST APIs
- **Option C:** Deploy as Oracle DB PL/SQL stored procedures (best performance)

### Option A: Python Direct Connection (Recommended for Pilot)

#### Prerequisites

```bash
pip install cx_Oracle sqlalchemy pandas
```

Download Oracle Instant Client: https://www.oracle.com/database/technologies/instant-client.html

#### Connection Code

```python
from aldf_eam.integrations import OracleMDMConnector
import os

# Initialize connector
mdm = OracleMDMConnector(
    host="oracle-mdm-prod.utility.com",
    port=1521,
    service_name="MDM_PROD",
    username="aldf_eam_user",
    password=os.getenv("ORACLE_PASSWORD")
)

# Test connection
if mdm.test_connection():
    print("✅ Connected to Oracle MDM")
```

#### Data Extraction

```python
# Extract interval data for specific feeders
interval_data = mdm.extract_interval_data(
    start_date="2026-01-01",
    end_date="2026-02-28",
    feeder_ids=["F001", "F002", "F003"]
)

# Extract meter events (tamper, power quality)
meter_events = mdm.extract_meter_events(
    start_date="2026-01-01",
    end_date="2026-02-28",
    event_types=["TAMPER", "POWER_OUTAGE", "REVERSE_ENERGY"]
)

# Extract network topology
topology = mdm.extract_topology(
    feeder_ids=["F001", "F002", "F003"]
)

# Extract meter health diagnostics
diagnostics = mdm.extract_diagnostics(
    feeder_ids=["F001", "F002", "F003"]
)
```

#### Sample SQL Queries (Oracle MDM Schema)

**Interval Data:**
```sql
SELECT 
    m.METER_ID,
    m.CONSUMER_ID,
    ir.INTERVAL_DATE,
    ir.INTERVAL_TIME,
    ir.KWH_DELIVERED,
    ir.KVA_DELIVERED,
    ir.VOLTAGE_A,
    ir.CURRENT_A
FROM 
    CI_METER m
    JOIN CI_INTERVAL_READ ir ON m.METER_ID = ir.METER_ID
    JOIN CI_SP sp ON m.SP_ID = sp.SP_ID
WHERE 
    sp.FEEDER_ID IN ('F001', 'F002', 'F003')
    AND ir.INTERVAL_DATE BETWEEN :start_date AND :end_date
    AND ir.VALIDATION_STATUS = 'VALID'
ORDER BY 
    m.METER_ID, ir.INTERVAL_DATE, ir.INTERVAL_TIME
```

**Meter Events:**
```sql
SELECT 
    m.METER_ID,
    me.EVENT_TYPE,
    me.EVENT_DATE,
    me.EVENT_TIME,
    me.EVENT_VALUE
FROM 
    CI_METER m
    JOIN CI_METER_EVENT me ON m.METER_ID = me.METER_ID
    JOIN CI_SP sp ON m.SP_ID = sp.SP_ID
WHERE 
    sp.FEEDER_ID IN ('F001', 'F002', 'F003')
    AND me.EVENT_DATE BETWEEN :start_date AND :end_date
    AND me.EVENT_TYPE IN ('TAMPER_COVER_OPEN', 'TAMPER_MAGNETIC', 'CT_FAIL')
ORDER BY 
    me.EVENT_DATE DESC
```

#### Run ALDF-EAM Analysis

```python
from aldf_eam import ALDF

# Initialize ALDF
aldf = ALDF(
    config={
        'D_critical': 0.40,
        'D_min': 0.65,
        'B_high': 0.60,
        'adaptive_fusion': True
    }
)

# Prepare data
data = {
    'interval_data': interval_data,
    'meter_events': meter_events,
    'topology': topology,
    'diagnostics': diagnostics
}

# Run analysis
results = aldf.analyze(data)

# Results object contains:
# - S-vector: results.s_vector
# - Operating band: results.band ('STAGE_A', 'TRANSITIONAL', 'FULL_ANALYTICS')
# - Active methods: results.active_methods
# - Suspects: results.suspects (DataFrame)
# - Confidence: results.confidence_scores
```

#### Write Results Back to Oracle MDM

```python
# Create results table (one-time setup)
mdm.create_results_table()

# Write suspects to MDM
mdm.write_suspects(
    results=results.suspects,
    table_name="ALDF_EAM_SUSPECTS",
    analysis_date="2026-02-28"
)

# Write S-vector history (for tracking maturity over time)
mdm.write_s_vector(
    s_vector=results.s_vector,
    feeder_id="F001",
    analysis_date="2026-02-28"
)
```

**Results Table Schema:**
```sql
CREATE TABLE ALDF_EAM_SUSPECTS (
    SUSPECT_ID NUMBER PRIMARY KEY,
    ANALYSIS_DATE DATE NOT NULL,
    FEEDER_ID VARCHAR2(20),
    CONSUMER_ID VARCHAR2(50) NOT NULL,
    METER_ID VARCHAR2(50),
    RISK_SCORE NUMBER(5,3),
    CONFIDENCE VARCHAR2(10), -- 'HIGH', 'MEDIUM', 'LOW'
    BAND VARCHAR2(20), -- 'STAGE_A', 'TRANSITIONAL', 'FULL_ANALYTICS'
    PRIMARY_INDICATOR VARCHAR2(100),
    ACTIVE_METHODS VARCHAR2(200),
    S_VECTOR_O NUMBER(5,3),
    S_VECTOR_D NUMBER(5,3),
    S_VECTOR_T NUMBER(5,3),
    S_VECTOR_B NUMBER(5,3),
    CREATED_DATE TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IDX_SUSPECTS_DATE ON ALDF_EAM_SUSPECTS(ANALYSIS_DATE);
CREATE INDEX IDX_SUSPECTS_FEEDER ON ALDF_EAM_SUSPECTS(FEEDER_ID);
CREATE INDEX IDX_SUSPECTS_RISK ON ALDF_EAM_SUSPECTS(RISK_SCORE DESC);
```

### Option B: Oracle APEX REST API (For Cloud Deployments)

If Oracle MDM exposes APEX REST endpoints:

```python
import requests
import json

# APEX REST endpoint
apex_url = "https://apex.oracle.com/pls/apex/mdm_prod/aldf_eam/"

# Authenticate
auth = requests.post(
    f"{apex_url}authenticate",
    json={"username": "aldf_user", "password": os.getenv("APEX_PASSWORD")}
)
token = auth.json()['access_token']

headers = {"Authorization": f"Bearer {token}"}

# Get interval data
response = requests.get(
    f"{apex_url}interval_data",
    headers=headers,
    params={
        "start_date": "2026-01-01",
        "end_date": "2026-02-28",
        "feeder_ids": "F001,F002,F003"
    }
)
interval_data = pd.DataFrame(response.json()['items'])

# Post results
requests.post(
    f"{apex_url}suspects",
    headers=headers,
    json=results.suspects.to_dict(orient='records')
)
```

### Option C: Native PL/SQL Module (Production)

For best performance, deploy ALDF-EAM logic as Oracle stored procedures:

```sql
-- S-vector computation
CREATE OR REPLACE FUNCTION COMPUTE_S_VECTOR(
    p_feeder_id VARCHAR2,
    p_analysis_date DATE
) RETURN SYS_REFCURSOR
IS
    v_cursor SYS_REFCURSOR;
    v_o_index NUMBER;
    v_d_index NUMBER;
    v_t_index NUMBER;
    v_b_index NUMBER;
BEGIN
    -- Compute O-index
    SELECT 
        (0.25 * c_f) + (0.20 * c_dt) + (0.20 * c_c) + 
        (0.20 * c_top) + (0.15 * c_sync)
    INTO v_o_index
    FROM (
        -- Sub-query to compute c_f, c_dt, c_c, c_top, c_sync
        -- ... implementation details ...
    );
    
    -- Compute D-index
    -- ... similar pattern ...
    
    -- Return S-vector
    OPEN v_cursor FOR
        SELECT v_o_index AS O, v_d_index AS D, 
               v_t_index AS T, v_b_index AS B
        FROM DUAL;
    
    RETURN v_cursor;
END COMPUTE_S_VECTOR;
/

-- Method selection
CREATE OR REPLACE FUNCTION SELECT_METHODS(
    p_o_index NUMBER,
    p_d_index NUMBER,
    p_b_index NUMBER
) RETURN VARCHAR2
IS
    v_methods VARCHAR2(200) := '';
BEGIN
    -- M1 activation
    IF p_o_index >= 0.75 THEN
        v_methods := v_methods || 'M1,';
    END IF;
    
    -- M2 activation
    IF p_o_index < 0.75 THEN
        v_methods := v_methods || 'M2,';
    END IF;
    
    -- M3 always
    v_methods := v_methods || 'M3,';
    
    -- M4-M6 activation
    IF p_b_index > 0.60 AND p_d_index >= 0.65 THEN
        v_methods := v_methods || 'M4,M5,M6';
    END IF;
    
    RETURN TRIM(TRAILING ',' FROM v_methods);
END SELECT_METHODS;
/
```

**Performance:** Native PL/SQL can process 1000 consumers in < 5 seconds vs. 30-60 seconds for Python.

---

## 2. Itron OpenWay MDM Integration

### Architecture

Itron MDM provides REST APIs for data access. ALDF-EAM uses:
- **Itron OpenWay Centron API** for meter reads
- **Itron Analytics API** for VEE data
- **Itron GIS API** for topology

### Prerequisites

```bash
pip install requests pandas pytz
```

Obtain API credentials from Itron OpenWay admin portal.

### Connection Code

```python
from aldf_eam.integrations import ItronMDMConnector
import os

mdm = ItronMDMConnector(
    api_url="https://itron-mdm.utility.com/api/v2",
    api_key=os.getenv("ITRON_API_KEY"),
    api_secret=os.getenv("ITRON_API_SECRET")
)

# Authenticate
mdm.authenticate()
```

### Data Extraction

```python
# Get meter group (equivalent to feeder)
meter_group = mdm.get_meter_group(group_id="FEEDER_001")

# Extract interval data
interval_data = mdm.get_interval_data(
    meter_group="FEEDER_001",
    start_date="2026-01-01T00:00:00Z",
    end_date="2026-02-28T23:59:59Z",
    interval_length="PT30M",  # 30-minute intervals
    channels=["kWh_delivered", "kVA", "voltage"]
)

# Get meter events
events = mdm.get_events(
    meter_group="FEEDER_001",
    start_date="2026-01-01",
    end_date="2026-02-28",
    event_types=["tamper", "outage", "reverse_energy"]
)

# Get VEE validation results
vee_data = mdm.get_vee_results(
    meter_group="FEEDER_001",
    date_range="last_60_days"
)
```

### Sample API Calls

**Interval Data:**
```python
import requests

response = requests.get(
    f"{api_url}/meters/interval-data",
    headers={"Authorization": f"Bearer {access_token}"},
    params={
        "meterGroup": "FEEDER_001",
        "startDate": "2026-01-01T00:00:00Z",
        "endDate": "2026-02-28T23:59:59Z",
        "channels": "kWh_delivered,kVA,voltage",
        "intervalLength": "PT30M"
    }
)
interval_data = response.json()
```

**Meter Events:**
```python
response = requests.get(
    f"{api_url}/meters/events",
    headers={"Authorization": f"Bearer {access_token}"},
    params={
        "meterGroup": "FEEDER_001",
        "startDate": "2026-01-01",
        "endDate": "2026-02-28",
        "eventTypes": "tamper,outage"
    }
)
events = response.json()
```

### Run Analysis & Write Results

```python
# Run ALDF-EAM
results = aldf.analyze({
    'interval_data': interval_data,
    'meter_events': events,
    'vee_data': vee_data
})

# Write results back to Itron
mdm.create_analytics_report(
    report_name="ALDF_EAM_NTL_Suspects",
    data=results.suspects,
    timestamp="2026-02-28"
)

# OR dispatch to field via Itron Work Order API
mdm.create_field_orders(
    suspects=results.suspects.head(25),  # Top 25
    order_type="NTL_INVESTIGATION",
    priority="HIGH"
)
```

---

## 3. Landis+Gyr Gridstream Integration

### Architecture

Landis+Gyr Gridstream Connect provides:
- **Gridstream Manager** for meter data
- **Command Center** for network topology
- **Analytics Engine** for pre-processing

### Prerequisites

```bash
pip install zeep lxml pandas  # For SOAP API
```

### Connection Code

```python
from aldf_eam.integrations import LandisGyrMDMConnector

mdm = LandisGyrMDMConnector(
    wsdl_url="https://gridstream.utility.com/services?wsdl",
    username="aldf_user",
    password=os.getenv("LG_PASSWORD")
)
```

### Data Extraction

```python
# Query meter data
meters = mdm.query_meters(
    transformer_id="DT_001",
    start_date="2026-01-01",
    end_date="2026-02-28"
)

# Get interval reads
interval_data = mdm.get_interval_data(
    meter_ids=meters['meter_id'].tolist(),
    start_date="2026-01-01",
    end_date="2026-02-28"
)

# Get load profile
load_profile = mdm.get_load_profile(
    transformer_id="DT_001",
    date_range="last_60_days"
)
```

### Sample SOAP Request

```python
from zeep import Client

client = Client(wsdl_url)

# Get interval data
response = client.service.GetIntervalData(
    MeterIDs=['MTR001', 'MTR002', 'MTR003'],
    StartDate='2026-01-01T00:00:00',
    EndDate='2026-02-28T23:59:59',
    Channels=['kWh', 'kVA', 'V']
)
```

### Run Analysis & Export

```python
# Run ALDF-EAM
results = aldf.analyze({
    'interval_data': interval_data,
    'load_profile': load_profile
})

# Export to GIS (for field visualization)
mdm.export_to_gis(
    suspects=results.suspects,
    gis_layer="NTL_SUSPECTS",
    symbology={
        'HIGH': 'red',
        'MEDIUM': 'orange',
        'LOW': 'yellow'
    }
)
```

---

## 4. Generic SQL-Based MDM Integration

For MDM systems not listed above (SAP IS-U, Siemens EnergyIP, custom databases):

### Data Model Mapping

Map your MDM schema to ALDF-EAM requirements:

| ALDF-EAM Field | Description | Typical MDM Table |
|----------------|-------------|-------------------|
| `consumer_id` | Unique consumer identifier | CUSTOMER, ACCOUNT |
| `meter_id` | Meter serial number | METER, DEVICE |
| `feeder_id` | Feeder/transformer ID | NETWORK_TOPOLOGY, GIS |
| `interval_timestamp` | Reading timestamp | INTERVAL_DATA, READS |
| `kwh_delivered` | Energy consumption | INTERVAL_DATA |
| `voltage`, `current` | Electrical params | INTERVAL_DATA |
| `event_type`, `event_timestamp` | Meter events | METER_EVENTS, ALARMS |
| `diagnostic_status` | Meter health | DIAGNOSTICS, VEE |

### Generic Extraction Template

```python
import pandas as pd
from sqlalchemy import create_engine

# Connect to any SQL database
engine = create_engine('postgresql://user:pass@mdm-db:5432/mdm_prod')

# Extract interval data
query = """
SELECT 
    c.consumer_id,
    m.meter_id,
    n.feeder_id,
    i.interval_timestamp,
    i.kwh_delivered,
    i.voltage_avg,
    i.current_avg
FROM 
    interval_data i
    JOIN meters m ON i.meter_id = m.meter_id
    JOIN consumers c ON m.consumer_id = c.consumer_id
    JOIN network_topology n ON m.transformer_id = n.transformer_id
WHERE 
    n.feeder_id IN ('F001', 'F002', 'F003')
    AND i.interval_timestamp BETWEEN :start_date AND :end_date
    AND i.quality_flag = 'VALID'
"""

interval_data = pd.read_sql(query, engine, params={
    'start_date': '2026-01-01',
    'end_date': '2026-02-28'
})
```

---

## 5. Performance Optimization

### Batch Processing (Large Utilities)

For utilities with 100K+ consumers:

```python
# Process feeders in batches
feeder_ids = get_all_feeders()  # 1000+ feeders
batch_size = 50

for i in range(0, len(feeder_ids), batch_size):
    batch = feeder_ids[i:i+batch_size]
    
    # Extract data for batch
    data = mdm.extract_interval_data(feeder_ids=batch)
    
    # Run ALDF-EAM
    results = aldf.analyze(data)
    
    # Write results
    mdm.write_suspects(results)
    
    print(f"Processed batch {i//batch_size + 1}, feeders {i} to {i+batch_size}")
```

### Parallel Processing

```python
from multiprocessing import Pool

def process_feeder(feeder_id):
    data = mdm.extract_interval_data(feeder_ids=[feeder_id])
    results = aldf.analyze(data)
    mdm.write_suspects(results)
    return feeder_id

# Process 8 feeders in parallel
with Pool(8) as pool:
    pool.map(process_feeder, feeder_ids)
```

### Incremental Updates

```python
# Daily incremental analysis (not full 60-day window)
last_analysis_date = mdm.get_last_analysis_date()

data = mdm.extract_interval_data(
    start_date=last_analysis_date,
    end_date=datetime.now()
)

results = aldf.analyze(data, mode='incremental')
```

---

## 6. Deployment Architectures

### Architecture 1: Microservice (Kubernetes)

```yaml
# aldf-eam-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aldf-eam-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aldf-eam
  template:
    metadata:
      labels:
        app: aldf-eam
    spec:
      containers:
      - name: aldf-eam
        image: ap1singh/aldf-eam:latest
        ports:
        - containerPort: 8000
        env:
        - name: ORACLE_HOST
          valueFrom:
            secretKeyRef:
              name: mdm-credentials
              key: oracle-host
        - name: ORACLE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mdm-credentials
              key: oracle-password
---
apiVersion: v1
kind: Service
metadata:
  name: aldf-eam-service
spec:
  selector:
    app: aldf-eam
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

### Architecture 2: Scheduled Batch (Cron)

```bash
# /etc/cron.d/aldf-eam
0 2 * * * aldf_user /opt/aldf-eam/run_analysis.sh >> /var/log/aldf-eam.log 2>&1
```

```bash
# run_analysis.sh
#!/bin/bash
cd /opt/aldf-eam
source venv/bin/activate
python -m aldf_eam.batch_runner --config /etc/aldf-eam/config.yaml
```

### Architecture 3: Real-Time Streaming (Kafka)

```python
from kafka import KafkaConsumer
import json

# Consume meter events in real-time
consumer = KafkaConsumer(
    'meter-events',
    bootstrap_servers='kafka:9092',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    event = message.value
    
    # If tamper event, trigger ALDF-EAM analysis
    if event['type'] == 'TAMPER':
        consumer_id = event['consumer_id']
        results = aldf.analyze_single_consumer(consumer_id)
        
        if results.risk_score > 0.80:
            dispatch_field_inspection(consumer_id, results)
```

---

## 7. Security & Authentication

### Oracle MDM (Kerberos)

```python
import cx_Oracle

# Kerberos authentication
connection = cx_Oracle.connect(
    user="/",  # Use current Kerberos principal
    dsn="oracle-mdm-prod.utility.com:1521/MDM_PROD",
    mode=cx_Oracle.SYSDBA,
    encoding="UTF-8"
)
```

### API Key Management (Environment Variables)

```bash
# .env file (never commit to git)
ORACLE_PASSWORD=SecurePassword123
ITRON_API_KEY=ak_live_abc123xyz
ITRON_API_SECRET=sk_live_def456uvw
LG_PASSWORD=AnotherSecurePass
```

```python
from dotenv import load_dotenv
load_dotenv()

password = os.getenv("ORACLE_PASSWORD")
```

### Network Security

```python
# SSL/TLS for Oracle connections
connection = cx_Oracle.connect(
    user="aldf_user",
    password=password,
    dsn="""(DESCRIPTION=
        (ADDRESS=(PROTOCOL=TCPS)(HOST=oracle-mdm.com)(PORT=2484))
        (CONNECT_DATA=(SERVICE_NAME=MDM_PROD))
        (SECURITY=(SSL_SERVER_CERT_DN="CN=oracle-mdm.com,O=Utility,C=IN"))
    )"""
)
```

---

## 8. Monitoring & Logging

### Application Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/aldf-eam/analysis.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger('aldf_eam')

logger.info(f"Starting analysis for feeder {feeder_id}")
logger.info(f"S-vector computed: {results.s_vector}")
logger.warning(f"D-index below threshold: {results.s_vector['D']}")
logger.error(f"Failed to connect to MDM: {error}")
```

### Performance Metrics

```python
import time

start_time = time.time()

# Run analysis
results = aldf.analyze(data)

duration = time.time() - start_time
logger.info(f"Analysis completed in {duration:.2f} seconds")

# Log to monitoring system (Prometheus/Grafana)
from prometheus_client import Histogram

analysis_duration = Histogram('aldf_eam_analysis_duration_seconds', 
                              'Time spent in analysis')
analysis_duration.observe(duration)
```

---

## 9. Testing & Validation

### Integration Tests

```python
import unittest

class TestOracleMDMIntegration(unittest.TestCase):
    
    def setUp(self):
        self.mdm = OracleMDMConnector(
            host="test-mdm.utility.com",
            service_name="MDM_TEST"
        )
    
    def test_connection(self):
        self.assertTrue(self.mdm.test_connection())
    
    def test_extract_interval_data(self):
        data = self.mdm.extract_interval_data(
            feeder_ids=["TEST_F001"],
            start_date="2026-01-01",
            end_date="2026-01-07"
        )
        self.assertGreater(len(data), 0)
        self.assertIn('consumer_id', data.columns)
        self.assertIn('kwh_delivered', data.columns)
    
    def test_write_suspects(self):
        suspects = pd.DataFrame({
            'consumer_id': ['C001', 'C002'],
            'risk_score': [0.92, 0.85],
            'confidence': ['HIGH', 'MEDIUM']
        })
        result = self.mdm.write_suspects(suspects)
        self.assertTrue(result)
```

### Data Quality Validation

```python
def validate_mdm_data(data):
    """Ensure MDM data meets ALDF-EAM requirements"""
    
    required_columns = ['consumer_id', 'meter_id', 'interval_timestamp', 'kwh_delivered']
    missing_columns = set(required_columns) - set(data.columns)
    
    if missing_columns:
        raise ValueError(f"Missing required columns: {missing_columns}")
    
    # Check for nulls
    if data[required_columns].isnull().any().any():
        logger.warning("Null values detected in critical columns")
    
    # Check timestamp format
    if not pd.api.types.is_datetime64_any_dtype(data['interval_timestamp']):
        data['interval_timestamp'] = pd.to_datetime(data['interval_timestamp'])
    
    # Check for duplicates
    duplicates = data.duplicated(subset=['consumer_id', 'interval_timestamp'])
    if duplicates.any():
        logger.warning(f"{duplicates.sum()} duplicate records found")
        data = data[~duplicates]
    
    return data
```

---

## 10. Troubleshooting

### Common Issues

**Issue 1: Oracle connection fails**
```
Error: ORA-12541: TNS:no listener
```
**Solution:**
- Check Oracle listener is running: `lsnrctl status`
- Verify hostname/IP in connection string
- Test with `tnsping MDM_PROD`

**Issue 2: Itron API returns 401 Unauthorized**
```
Error: 401 Unauthorized - Invalid API key
```
**Solution:**
- Verify API key is active in Itron portal
- Check expiration date
- Regenerate if necessary

**Issue 3: Data extraction timeout**
```
Error: Connection timeout after 30 seconds
```
**Solution:**
```python
# Increase timeout
mdm = OracleMDMConnector(
    ...,
    timeout=300  # 5 minutes
)

# OR: Extract in smaller date ranges
for date in pd.date_range('2026-01-01', '2026-02-28', freq='7D'):
    data = mdm.extract_interval_data(
        start_date=date,
        end_date=date + timedelta(days=7)
    )
```

**Issue 4: S-vector shows D = 0.0 (invalid)**
```
S-vector: [0.557, 0.00, 0.42, 0.65]
```
**Cause:** Missing data quality metrics (diagnostics, tamper events)

**Solution:**
- Ensure `meter_events` table populated
- Check if VEE is enabled in MDM
- Manually set d_compl, d_uptime if not available

---

## 11. Support & Contact

**For MDM vendor partnerships:**
- Technology licensing inquiries
- Co-development discussions
- Integration support

**Contact:** [your.email@domain.com]

**For technical integration support:**
- GitHub Issues: https://github.com/ap1singh/ALDF-EAM/issues
- Integration examples: [`examples/mdm_integration_example.py`](../examples/mdm_integration_example.py)

---

## Appendix: MDM Vendor Comparison

| Feature | Oracle MDM | Itron OpenWay | Landis+Gyr Gridstream |
|---------|------------|---------------|----------------------|
| **API Type** | SQL + REST | REST | SOAP + REST |
| **Data Format** | Oracle DB | JSON | XML |
| **Authentication** | Oracle user/pass | OAuth 2.0 | Username/pass |
| **Best For** | Large enterprise | North America | Europe/Asia |
| **Integration Effort** | Low (native SQL) | Medium (REST) | Medium-High (SOAP) |
| **Performance** | Excellent | Good | Good |
| **ALDF-EAM Support** | ✅ Full | ✅ Full | ✅ Full |

---

**Document Version:** 1.0  
**Last Updated:** March 2026  
**Author:** Atul Pratap Singh  
**Repository:** https://github.com/ap1singh/ALDF-EAM
