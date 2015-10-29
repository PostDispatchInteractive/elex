![](https://cloud.githubusercontent.com/assets/109988/10737959/635bfb56-7beb-11e5-9ee5-102eb1582718.png)

## Usage
### Demo app
```bash
python -m elex.demo
```

### Modules
Use the election loader manually from within your project.

#### Elections
```python
from elex.parser import api

# Show all elections available.
# Note: Some elections may be in the past.
elections = api.Election.get_elections()

# Get the next election.
election = api.Election.get_next_election()
```

#### Races, Candidates and results
```python
import datetime

from elex.parser import api
from elex import loader
from elex.loader import postgres


TABLE_LIST = [
    postgres.Candidate,
    postgres.CandidateResult,
    postgres.Race,
    postgres.ReportingUnit,
    postgres.BallotPosition
]

start = datetime.datetime.now()

candidate_results = []
reportingunits = []
races = []

# Load races, reporting units and candidates into lists.
for race in api.Election.get_races('2015-11-03', omitResults=False, level="ru", test=True):
    for reporting_unit in race.reportingunits:
        reportingunits.append(reporting_unit)
        candidate_results += [c for c in reporting_unit.candidates]
        del reporting_unit.candidates
    del race.candidates
    del race.reportingunits
    races.append(race)


unique_candidates = {}
unique_ballotpositions = {}

for c in candidate_results:
    if c.is_ballot_position:
        if not unique_ballotpositions.get(c.candidateid, None):
            unique_ballotpositions[c.candidateid] = {"last": c.last, "candidateid": c.candidateid, "polid": c.polid, "ballotorder": c.ballotorder, "polnum": c.polnum, "seatname": c.seatname, "description": c.description}
    else:
        if not unique_candidates.get(c.candidateid, None):
            unique_candidates[c.candidateid] = {"first": c.first, "last": c.last, "candidateid": c.candidateid, "polid": c.polid, "ballotorder": c.ballotorder, "polnum": c.polnum, "party": c.party}

candidates = [postgres.Candidate(**v) for v in unique_candidates.values()]
ballotpositions = [postgres.BallotPosition(**v) for v in unique_ballotpositions.values()]

print "Parsed %s candidate results." % len(candidate_results)
print "Parsed %s candidates." % len(candidates)
print "Parsed %s ballot positions." % len(ballotpositions)
print "Parsed %s reporting units." % len(reportingunits)
print "Parsed %s races.\n" % len(races)

parse_end = datetime.datetime.now()

# Connect to the database.
# Drop and recreate tables, as we're bulk-loading.
loader.ELEX_PG_CONNEX.connect()
loader.ELEX_PG_CONNEX.drop_tables(TABLE_LIST, safe=True)
loader.ELEX_PG_CONNEX.create_tables(TABLE_LIST, safe=True)

# Do the bulk loads with atomic transactions.
with loader.ELEX_PG_CONNEX.atomic():
    for idx in range(0, len(candidates), 1000):
        postgres.Candidate.insert_many([c.__dict__['_data'] for c in candidates[idx:idx+1000]]).execute()

with loader.ELEX_PG_CONNEX.atomic():
    for idx in range(0, len(ballotpositions), 1000):
        postgres.BallotPosition.insert_many([c.__dict__['_data'] for c in ballotpositions[idx:idx+1000]]).execute()

with loader.ELEX_PG_CONNEX.atomic():
    for idx in range(0, len(candidate_results), 1000):
        postgres.CandidateResult.insert_many([c.__dict__ for c in candidate_results[idx:idx+1000]]).execute()

with loader.ELEX_PG_CONNEX.atomic():
    for idx in range(0, len(reportingunits), 1000):
        postgres.ReportingUnit.insert_many([c.__dict__ for c in reportingunits[idx:idx+1000]]).execute()

with loader.ELEX_PG_CONNEX.atomic():
    for idx in range(0, len(races), 1000):
        postgres.Race.insert_many([c.__dict__ for c in races[idx:idx+1000]]).execute()

print "Inserted %s candidate results." % len(candidate_results)
print "Inserted %s candidates." % len(candidates)
print "Inserted %s ballot positions." % len(ballotpositions)
print "Inserted %s reporting units." % len(reportingunits)
print "Inserted %s races.\n" % len(races)

end = datetime.datetime.now()

print "Overall: %s seconds." % float(str(end - start).split(':')[-1])
print "  Parsing: %s seconds." % float(str(parse_end - start).split(':')[-1])
print "  Loading: %s seconds." % float(str(end - parse_end).split(':')[-1])
```

## Options
### Recording
#### Flat files
Will record timestamped and namespaced files to the `ELEX_RECORDING_DIR` before parsing.

```bash
export ELEX_RECORDING=flat
export ELEX_RECORDING_DIR=/tmp
```

#### MongoDB
Will record a timestamped record to MongoDB, connecting via `ELEX_RECORDING_MONGO_URL` and writing to the `ELEX_RECORDING_MONGO_DB` database.

```bash
export ELEX_RECORDING=mongodb
export ELEX_RECORDING_MONGO_URL=mongodb://localhost:27017/  # Or your own connection string.
export ELEX_RECORDING_MONGO_DB=ap_elections_loader
```