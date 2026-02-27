---
name: training-records
description: >
  Track staff competence, training completion, and DARE authorization eligibility.
  Use when: recording training, checking who is trained, flagging retraining needs,
  verifying DARE eligibility, or building a competency matrix.

# === TU-POC Extensions ===
suppliers:
  - source: directory_skill
    type: internal
    description: "Provides person registry"
  - source: skill_creator
    type: internal
    description: "When skills update version, triggers retraining flags"

customers:
  - recipient: dare_framework
    type: internal
    receives: "competence verification for authorization"
  - recipient: management
    type: internal
    receives: "competency matrix and gap reports"
  - recipient: auditor
    type: regulatory
    receives: "ISO 7.2 compliance evidence"

with_who:
  executor:
    model: gemini-2.5-flash
    capabilities: [reasoning, tool_use]

risk_profile:
  actions:
    - name: record_training
      rpn: 2
      autonomy: autonomous
    - name: flag_retraining
      rpn: 2
      autonomy: autonomous
    - name: verify_dare_eligibility
      rpn: 4
      autonomy: autonomous
    - name: override_competence
      rpn: 12
      autonomy: dare
      mitigations: [require_justification, log_override]
    - name: delete_training_record
      rpn: 15
      autonomy: dare
      mitigations: [require_owner_approval, backup_first]

measures:
  - kpi: training_coverage
    target: ">90% of staff trained on assigned skills"
  - kpi: retraining_response
    target: "<7 days from flag to completion"

environmental:
  compute_cost: low
  data_retention: permanent

next_review: 2026-08-11
---

# Training Records

Track who is trained on what, flag retraining when skills change, verify DARE eligibility per ISO 9001 Clause 7.2.

## Storage

```bash
mkdir -p data/training
```

Records stored as JSON at `data/training/`.

## Record Training

```bash
python3 -c "
import json, os
from datetime import datetime
rec = {
    'person_name': 'NAME',
    'skill_name': 'SKILL',
    'skill_version': 'VERSION',
    'method': 'METHOD',  # classroom|shadowing|self-study|on-the-job
    'completion_date': datetime.utcnow().isoformat()+'Z',
    'assessed_by': 'ASSESSOR',
    'result': 'RESULT',  # competent|needs-practice|not-competent
    'status': 'active'
}
path = f'data/training/{rec[\"person_name\"].lower().replace(\" \",\"-\")}-{rec[\"skill_name\"]}.json'
json.dump(rec, open(path, 'w'), indent=2)
print(f'Training recorded: {path}')
"
```

## Flag Retraining

When a SKILL.md version changes, find affected training records:

```bash
python3 -c "
import json, glob
skill = 'SKILL_NAME'
current_version = 'NEW_VERSION'
for f in glob.glob('data/training/*.json'):
    rec = json.load(open(f))
    if rec.get('skill_name') == skill and rec.get('skill_version') != current_version:
        rec['status'] = 'retraining-required'
        rec['retraining_reason'] = f'Skill updated to v{current_version}'
        json.dump(rec, open(f, 'w'), indent=2)
        print(f'Flagged: {rec[\"person_name\"]} needs retraining on {skill}')
"
```

## Check DARE Eligibility

Before granting DARE authority: person must have active `competent` record on current skill version.

## Competency Matrix

```bash
python3 -c "
import json, glob
records = [json.load(open(f)) for f in glob.glob('data/training/*.json')]
people = sorted(set(r['person_name'] for r in records))
skills = sorted(set(r['skill_name'] for r in records))
print(f'{'Person':30s}', '  '.join(s[:8] for s in skills))
for p in people:
    row = []
    for s in skills:
        match = [r for r in records if r['person_name']==p and r['skill_name']==s]
        if not match: row.append('❌')
        elif match[-1]['status']=='retraining-required': row.append('⚠️ ')
        elif match[-1]['result']=='competent': row.append('✅')
        else: row.append('🔄')
    print(f'{p:30s}', '  '.join(row))
"
```

## ISO 9001 Compliance

| Clause | Requirement            | Implementation                  |
| ------ | ---------------------- | ------------------------------- |
| 7.2(a) | Determine competence   | Skill-to-training mapping       |
| 7.2(b) | Ensure competence      | Training records + assessment   |
| 7.2(c) | Evaluate effectiveness | Result field + retraining flags |
| 7.2(d) | Retain documented info | All records stored as JSON      |
