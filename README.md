# TransitRoutePlanner_3106

An Ottawa transit planner that ranks routes to Carleton by the **probability of arriving before a deadline**, using a Bayesian delay model learned from OC Transpo's real-time feed. COMP 3106 (Fall 2026) project by Rayan Hassan and Vedant Dhawal.

## Setup

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Create `.env` in the repo root (it's gitignored) with your OC Transpo developer key from <https://nextrip-public-api.developer.azure-api.net/> (Profile → Primary key):

```
OCPRIMARYKEY=your-key-here
```

## Data collection

OC Transpo publishes only live data, with no history, so we archive the GTFS-Realtime feed ourselves.

| Feed | What it gives us |
|---|---|
| Trip updates | For each active bus trip: predicted arrival time at each upcoming stop, and canceled trips |
| Vehicle positions | GPS position and timestamp per bus (no stop or status fields) |

Notes from inspecting the feed (Nov 18, 2026):
- Buses only. **The O-Train (Lines 1, 2, 4) is not in the real-time feed.**
- There is no `delay` field. Delay = predicted arrival − scheduled arrival, joined to the static GTFS on `trip_id` + `stop_sequence`.
- Stops a bus has already passed drop out of its trip update. The last prediction before a stop drops out is our "actual" arrival.

### Run locally

```bash
.venv/bin/python collector/collector.py --once         # one snapshot into data/raw/
.venv/bin/python collector/collector.py --every 60     # poll every minute until stopped
.venv/bin/python tools/peek.py data/raw/<date>/tu/<time>.pb.gz
```

### Run on AWS (what we use for the real data)

A Lambda function polls both feeds every minute from 5:00 to 23:59 Ottawa time and writes `raw/<date>/<tu|vp>/<HHMMSS>.pb.gz` to an S3 bucket. Needs the AWS CLI and SAM CLI configured.

```bash
./deploy.sh
aws cloudformation describe-stacks --stack-name oc-transit-collector --query "Stacks[0].Outputs"
```

Download one day to process it:

```bash
aws s3 sync s3://<bucket>/raw/2026-11-19 data/raw/2026-11-19
```

## Repo layout

| Path | Purpose |
|---|---|
| `collector/collector.py` | Fetches the feeds and saves raw snapshots (local or S3) |
| `template.yaml`, `deploy.sh` | AWS Lambda + S3 + schedule for the collector |
| `tools/peek.py` | Summarizes one snapshot to check the collector works |

## Data sources and licence

OC Transpo GTFS and GTFS-RT data, used under the City of Ottawa Open Data Terms of Use. This project isn't affiliated with or endorsed by OC Transpo or the City of Ottawa.
