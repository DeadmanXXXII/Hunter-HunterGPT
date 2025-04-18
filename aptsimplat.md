# Red Horizon: APT-Centric Autonomous Training Platform

## Root Directory Structure
```
/red_horizon_labs
├── backend
│   ├── api
│   │   ├── views
│   │   │   ├── auth.py
│   │   │   ├── simulations.py
│   │   │   ├── apt_generator.py
│   │   ├── models
│   │   │   ├── user.py
│   │   │   ├── simulation.py
│   │   │   ├── leaderboard.py
│   │   │   ├── scenario_template.py
│   │   │   └── apt_profile.py
│   ├── workers
│   │   └── celery_worker.py
│   ├── engine
│   │   └── auto_generator.py
│   │   └── docker_builder.py
│   │   └── nvd_crawler.py
│   └── app.py
├── frontend
│   └── react-app
│       └── src
│           └── components
│           └── pages
│           └── App.js
├── simulator_vms
│   ├── konti_ransomware_lab
│   ├── koobface_smartphone_lab
│   ├── rhysida_lab
│   ├── kioptrix_lvl1
│   └── dev_butler_blackperl_blue
├── huntergpt
│   ├── gpt_assist.py
│   └── integration.py
├── dockerfiles
│   ├── simulation_base.Dockerfile
│   └── nginx.Dockerfile
├── config
│   ├── settings.yaml
│   └── secrets.env
├── database
│   └── init_db.sql
├── scripts
│   ├── init_db.py
│   └── seed_data.py
├── README.md
├── docker-compose.yml
└── requirements.txt
```

## Key Code Snippets

### backend/api/views/apt_generator.py
```python
from engine.auto_generator import generate_apt_scenario
from flask import Blueprint, jsonify, request

apt_generator = Blueprint('apt_generator', __name__)

@apt_generator.route('/generate/apt', methods=['POST'])
def auto_generate():
    keyword = request.json.get('keyword', 'APT')
    new_scenario = generate_apt_scenario(keyword)
    return jsonify(new_scenario)
```

### backend/engine/auto_generator.py
```python
import requests, os, json
from .docker_builder import build_lab
from .nvd_crawler import fetch_latest_apt

SCENARIO_BASE = '/simulator_vms/'

def generate_apt_scenario(keyword):
    apt_report = fetch_latest_apt(keyword)
    title = apt_report['name']
    desc = apt_report['description']
    cves = apt_report['cves']

    build_path = os.path.join(SCENARIO_BASE, title.replace(' ', '_'))
    build_lab(title, desc, cves, build_path)
    return {
        'title': title,
        'description': desc,
        'path': build_path,
        'cves': cves
    }
```

### backend/engine/nvd_crawler.py
```python
import requests

def fetch_latest_apt(keyword):
    url = f"https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch={keyword}&resultsPerPage=1"
    res = requests.get(url)
    data = res.json()
    first = data['vulnerabilities'][0]['cve']
    return {
        'name': first['id'],
        'description': first['descriptions'][0]['value'],
        'cves': [first['id']]
    }
```

### backend/engine/docker_builder.py
```python
import os

def build_lab(title, desc, cves, path):
    os.makedirs(path, exist_ok=True)
    dockerfile_path = os.path.join(path, 'Dockerfile')
    with open(dockerfile_path, 'w') as f:
        f.write(f"""
        FROM ubuntu:20.04
        RUN apt update && apt install -y netcat curl
        RUN echo '{desc}' > /info.txt
        LABEL CVE="{','.join(cves)}"
        CMD [\"/bin/bash\"]
        """)
```

### docker-compose.yml
```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - FLASK_APP=app.py
      - FLASK_ENV=development
  frontend:
    build: ./frontend/react-app
    ports:
      - "3000:3000"
  database:
    image: postgres
    restart: always
    environment:
      POSTGRES_DB: redhorizon
      POSTGRES_USER: red
      POSTGRES_PASSWORD: secure
  redis:
    image: redis
```

### backend/app.py
```python
from flask import Flask
from api.views.apt_generator import apt_generator

app = Flask(__name__)
app.register_blueprint(apt_generator)

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```

## Next Steps
- 🧠 Add GPT-4 Turbo hook into auto_generator for scenario enrichment
- 💾 Store generated scenarios into `scenario_template` table with difficulty tagging
- 🧪 Wrap pre-built labs (like Rhysida, Conti, etc.) with metadata + GUI boot options
- 🌐 Deploy this backend with a React dashboard

