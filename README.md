
# Mini-Lab: Building, Testing, and Deploying a Healthcare FastAPI Application with CI/CD

## Overview
In this mini-lab, you will build a FastAPI application to manage a healthcare patient dataset, test it with pytest, set up a CI/CD pipeline using GitHub Actions, and deploy it to Google Cloud Run. You will also create an Ubuntu VM with Chrome Remote Desktop on GCP to replicate the process. The dataset includes patient records with fields like ID, name, age, condition, and admission date. A rubric is provided for grading your submission on Canvas.

**Learning Objectives**:
- Build a REST API with FastAPI and SQLite.
- Test the API using pytest.
- Set up CI/CD with GitHub Actions.
- Deploy to Google Cloud Run.
- Configure an Ubuntu VM with Chrome Remote Desktop on GCP.
- Document the process with screenshots and a report.

**Prerequisites**:
- GitHub account.
- Google Cloud Platform (GCP) account with billing enabled (free tier available).
- Basic Python knowledge.
- Familiarity with terminal commands.

## Step-by-Step Instructions

### Part 1: Set Up the Healthcare Dataset and FastAPI Application
1. **Create Project Directory**:
   ```bash
   mkdir healthcare-api && cd healthcare-api
   mkdir tests
   touch main.py database.py requirements.txt README.md tests/__init__.py tests/test_api.py
   ```

2. **Create the Healthcare Dataset**:
   - Create a CSV file `patients.csv` with sample data:
     ```csv
     id,name,age,condition,admission_date
     1,John Doe,45,Hypertension,2025-01-15
     2,Jane Smith,32,Diabetes,2025-02-20
     3,Alice Brown,60,Heart Disease,2025-03-10
     ```

3. **Set Up SQLite Database** (`database.py`):
   ```python
   from sqlalchemy import create_engine, Column, Integer, String
   from sqlalchemy.ext.declarative import declarative_base
   from sqlalchemy.orm import sessionmaker
   import csv

   Base = declarative_base()
   engine = create_engine('sqlite:///patients.db')

   class Patient(Base):
       __tablename__ = 'patients'
       id = Column(Integer, primary_key=True)
       name = Column(String)
       age = Column(Integer)
       condition = Column(String)
       admission_date = Column(String)

   Base.metadata.create_all(engine)

   def init_db():
       with open('patients.csv', 'r') as f:
           csv_reader = csv.DictReader(f)
           with sessionmaker(bind=engine)() as session:
               for row in csv_reader:
                   patient = Patient(**row)
                   session.add(patient)
               session.commit()

   def get_db():
       db = sessionmaker(bind=engine)()
       try:
           yield db
       finally:
           db.close()
   ```

4. **Create FastAPI Application** (`main.py`):
   ```python
   from fastapi import FastAPI, Depends, HTTPException
   from sqlalchemy.orm import Session
   from pydantic import BaseModel
   from database import Patient, get_db, init_db

   app = FastAPI(title="Healthcare API")

   class PatientModel(BaseModel):
       name: str
       age: int
       condition: str
       admission_date: str

   @app.on_event("startup")
   def startup_event():
       init_db()

   @app.post("/patients/")
   def create_patient(patient: PatientModel, db: Session = Depends(get_db)):
       if patient.age < 0 or patient.age > 120:
           raise HTTPException(status_code=422, detail="Age must be between 0 and 120")
       db_patient = Patient(**patient.dict())
       db.add(db_patient)
       db.commit()
       db.refresh(db_patient)
       return db_patient

   @app.get("/patients/")
   def get_patients(db: Session = Depends(get_db)):
       return db.query(Patient).all()

   @app.get("/patients/{patient_id}")
   def get_patient(patient_id: int, db: Session = Depends(get_db)):
       patient = db.query(Patient).filter(Patient.id == patient_id).first()
       if not patient:
           raise HTTPException(status_code=404, detail="Patient not found")
       return patient
   ```

5. **Create Requirements File** (`requirements.txt`):
   ```
   fastapi==0.115.0
   uvicorn==0.30.6
   sqlalchemy==2.0.31
   pytest==8.3.2
   pytest-httpx==0.30.0
   ```

6. **Install Dependencies and Run Locally**:
   ```bash
   pip install -r requirements.txt
   uvicorn main:app --reload
   ```
   - Access API at `http://127.0.0.1:8000/docs`.

7. **Test Locally**:
   - Use the FastAPI docs UI to create a patient (e.g., `POST /patients/` with `{"name": "Bob Wilson", "age": 50, "condition": "Asthma", "admission_date": "2025-04-01"}`).
   - Verify patients are listed at `GET /patients/`.

### Part 2: Test the Application with pytest
1. **Create Test File** (`tests/test_api.py`):
   ```python
   from fastapi.testclient import TestClient
   from main import app
   from sqlalchemy import create_engine
   from sqlalchemy.orm import sessionmaker
   from database import Base, Patient
   import pytest

   @pytest.fixture
   def client():
       return TestClient(app)

   @pytest.fixture
   def test_db():
       engine = create_engine("sqlite:///:memory:")
       Base.metadata.create_all(engine)
       SessionLocal = sessionmaker(bind=engine)
       db = SessionLocal()
       yield db
       db.close()

   def test_create_patient(client, test_db):
       response = client.post("/patients/", json={"name": "Bob Wilson", "age": 50, "condition": "Asthma", "admission_date": "2025-04-01"})
       assert response.status_code == 200
       assert response.json()["name"] == "Bob Wilson"

   def test_create_patient_invalid_age(client):
       response = client.post("/patients/", json={"name": "Bob Wilson", "age": 150, "condition": "Asthma", "admission_date": "2025-04-01"})
       assert response.status_code == 422

   def test_get_patients(client, test_db):
       client.post("/patients/", json={"name": "Bob Wilson", "age": 50, "condition": "Asthma", "admission_date": "2025-04-01"})
       response = client.get("/patients/")
       assert response.status_code == 200
       assert len(response.json()) >= 1
   ```

2. **Run Tests**:
   ```bash
   pytest tests/test_api.py -v
   ```
   - Take a screenshot of the test output.

### Part 3: Set Up GitHub Repository and CI/CD
1. **Clone,Commit and Push to GitHub**:
   ```bash
   git clone your_repo
   git add .
   git commit -m "Initial commit for Healthcare API"
   git branch -M main
   git push origin main
   ```

2. **Create Dockerfile**:
   ```dockerfile
   FROM python:3.8-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
   ```

3. **Create CI/CD Workflow** (`.github/workflows/ci-cd.yml`):
   ```yaml
   name: CI/CD
   on:
     push:
       branches: [main]
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Set up Python
           uses: actions/setup-python@v4
           with:
             python-version: '3.8'
         - name: Install dependencies
           run: pip install -r requirements.txt
         - name: Run tests
           run: pytest tests/test_api.py -v
     deploy:
       needs: test
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Set up Google Cloud SDK
           uses: google-github-actions/setup-gcloud@v1
           with:
             project_id: ${{ secrets.GCP_PROJECT_ID }}
             service_account_key: ${{ secrets.GCP_SA_KEY }}
         - name: Configure Docker
           run: gcloud auth configure-docker
         - name: Build and Push Docker Image
           run: |
             docker build -t gcr.io/${{ secrets.GCP_PROJECT_ID }}/healthcare-api:latest .
             docker push gcr.io/${{ secrets.GCP_PROJECT_ID }}/healthcare-api:latest
         - name: Deploy to Cloud Run
           run: |
             gcloud run deploy healthcare-api \
               --image gcr.io/${{ secrets.GCP_PROJECT_ID }}/healthcare-api:latest \
               --platform managed \
               --region us-central1 \
               --allow-unauthenticated
   ```

4. **Set Up GCP Credentials**:
   - Create a GCP project and enable Cloud Run and Container Registry APIs.
   - Create a service account with roles: Cloud Run Admin, Storage Admin, Service Account User.
   - Download the JSON key and add it as `GCP_SA_KEY` in GitHub Secrets.
   - Add `GCP_PROJECT_ID` as a GitHub Secret.

5. **Push Changes to Trigger CI/CD**:
   ```bash
   git add .
   git commit -m "Added CI/CD workflow"
   git push origin main
   ```
   - Monitor the GitHub Actions workflow in the Actions tab and take a screenshot of the successful run.

### Part 4: Set Up Ubuntu VM with Chrome Remote Desktop on GCP
1. **Create Ubuntu VM**:
   - In GCP Console, go to Compute Engine > VM Instances > Create Instance.
   - Name: `healthcare-vm`, OS: Ubuntu 20.04 LTS, Machine type: e2-medium.
   - Allow HTTP/HTTPS traffic.
   - Create and note the external IP.

2. **Install Chrome Remote Desktop**:
   - SSH into the VM:
     ```bash
     gcloud compute ssh healthcare-vm
     ```
   - Install Chrome Remote Desktop:
     ```bash
     sudo apt-get update
     sudo apt-get install -y xvfb xfce4 xfce4-goodies
     sudo apt-get install -y chrome-remote-desktop
     ```
   - Install Chrome:
     ```bash
     wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
     sudo dpkg -i google-chrome-stable_current_amd64.deb
     sudo apt-get install -f
     ```
   - Set up Chrome Remote Desktop:
     ```bash
     /usr/lib/chromium-browser/chrome-remote-desktop --start
     ```
   - Follow prompts to set up a PIN and authorize at `remotedesktop.google.com`.

3. **Clone Repository and Repeat Process**:
   - Install dependencies:
     ```bash
     sudo apt-get install -y python3-pip git docker.io
     sudo pip3 install -r requirements.txt
     ```
   - Clone your GitHub repository:
     ```bash
     git clone <your-github-repo-url>
     cd healthcare-api
     ```
   - Run tests:
     ```bash
     pytest tests/test_api.py -v
     ```
   - Build and test Docker image:
     ```bash
     sudo docker build -t healthcare-api .
     sudo docker run -p 8080:8080 healthcare-api
     ```
   - Access at `http://<VM-EXTERNAL-IP>:8080/docs` and take a screenshot.

4. **Deploy to Cloud Run from VM**:
   - Install gcloud CLI:
     ```bash
     curl https://sdk.cloud.google.com | bash
     exec -l $SHELL
     gcloud init
     ```
   - Authenticate and set project:
     ```bash
     gcloud auth login
     gcloud config set project <your-project-id>
     ```
   - Build and push Docker image:
     ```bash
     gcloud auth configure-docker
     docker tag healthcare-api gcr.io/<your-project-id>/healthcare-api:latest
     docker push gcr.io/<your-project-id>/healthcare-api:latest
     ```
   - Deploy to Cloud Run:
     ```bash
     gcloud run deploy healthcare-api \
       --image gcr.io/<your-project-id>/healthcare-api:latest \
       --platform managed \
       --region us-central1 \
       --allow-unauthenticated
     ```
   - Take a screenshot of the deployed URL.

### Part 5: Submission Requirements
1. **Report**:
   - Create a Word document (`report.docx`) with:
     - Introduction: Briefly describe the lab and its objectives.
     - Screenshots: Include screenshots of:
       - FastAPI docs UI with a successful `POST /patients/` request.
       - pytest test output.
       - GitHub Actions CI/CD run (successful test and deploy).
       - Chrome Remote Desktop session showing VM setup.
       - VM local API test (`http://<VM-EXTERNAL-IP>:8080/docs`).
       - Cloud Run deployed URL.
     - Descriptions: For each screenshot, explain what it shows and its significance.
     - Conclusion: Reflect on what you learned about CI/CD and deployment.
2. **GitHub Repository**:
   - Push all code to a public GitHub repository.
   - Include a `README.md` with setup and run instructions.
3. **Canvas Submission**:
   - Submit `report.docx` and the GitHub repository URL via Canvas.

### Rubric (50 Points)
| **Criteria** | **Description** | **Points** |
|--------------|-----------------|------------|
| **FastAPI Application (10 pts)** | Correctly implements `main.py`, `database.py`, and `patients.csv` with all endpoints functioning. | 10 |
| **pytest Tests (10 pts)** | Implements at least three tests in `test_api.py` covering `POST` and `GET` endpoints with valid and invalid inputs. | 10 |
| **CI/CD Pipeline (10 pts)** | Sets up GitHub Actions workflow (`ci-cd.yml`) that runs tests and deploys to Cloud Run successfully. | 10 |
| **GCP VM Setup (10 pts)** | Configures Ubuntu VM with Chrome Remote Desktop, clones repo, and runs tests and local API successfully. | 10 |
| **Report Quality (10 pts)** | Includes all required screenshots with clear descriptions and a reflective conclusion in `report.docx`. | 10 |

**Total**: 50 points

## Tips
- **GCP Free Tier**: Use the $300 free credits for new GCP accounts.
- **GitHub Secrets**: Ensure `GCP_SA_KEY` and `GCP_PROJECT_ID` are correctly set to avoid deployment errors.
- **Screenshots**: Use Chrome Remote Desktop to capture VM activities clearly.
- **Documentation**: Keep your `README.md` concise and professional for portfolio use.

## Submission Checklist
- [ ] `patients.csv`, `main.py`, `database.py`, `requirements.txt`, `Dockerfile`, `tests/test_api.py`.
- [ ] GitHub repository with CI/CD workflow.
- [ ] Ubuntu VM with Chrome Remote Desktop set up and tested.
- [ ] `report.docx` with screenshots and descriptions.
- [ ] Canvas submission with report and repo URL.

**References**:
- Deploying to Cloud Run: https://cloud.google.com/run/docs[](https://cloud.google.com/run/docs/quickstarts/deploy-continuously)
- GitHub Actions: https://docs.github.com/en/actions[](https://github.com/features/actions)

