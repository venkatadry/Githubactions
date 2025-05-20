# Githubactions

Here are the structured notes based on the video transcript:

---

### **GitHub Actions CI/CD Pipeline with Live Project**  

#### **Agenda**  
1. **Set up GitHub Repository**  
2. **Configure GitHub Actions**  
3. **Add a VM as a Self-Hosted Runner**  
4. **Write CI/CD Pipeline (YAML) from Scratch**  
   - Security Scans (Trivy, GitLeaks)  
   - Unit Testing  
   - Build & Publish Artifacts  
   - Docker Image Build & Scan  
   - Kubernetes Deployment  

---

### **Key Concepts**  

#### **GitHub Actions vs Jenkins**  
- **Jenkins**:  
  - Requires dedicated resources (master/slave VMs).  
  - Manual setup and maintenance.  
  - Costs for infrastructure.  

- **GitHub Actions**:  
  - No setup overhead (hosted by GitHub).  
  - **Runners**:  
    - **Shared Runners**: Free, managed by GitHub (limited backend access).  
    - **Private Runners**: Self-hosted VMs (full control, isolated).  

---

### **Phase 1: Set Up GitHub Repo**  
1. Create a new repository (`GitHub-Actions-Project`).  
2. Clone locally, add code, and push:  
   ```bash
   git clone <repo-url>
   git add .
   git commit -m "Initial commit"
   git push origin main
   ```

---

### **Phase 2: Configure GitHub Actions**  
#### **Shared Runner Example**  
1. **Template Setup**:  
   - Use GitHub’s pre-built Java/Maven template.  
   - File: `.github/workflows/cicd.yml`.  
   ```yaml
   name: CI/CD Pipeline
   on: [push]
   jobs:
     compile:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Set up JDK 17
           uses: actions/setup-java@v3
           with: { java-version: '17' }
         - run: mvn package
   ```  
   - **Trigger**: Push to `main` branch.  

2. **Security Scans**:  
   - Add jobs for Trivy (vulnerability scan) and GitLeaks (secrets detection).  
   ```yaml
   security-checks:
     needs: compile
     runs-on: ubuntu-latest
     steps:
       - uses: actions/checkout@v4
       - name: Install Trivy
         run: sudo apt install -y trivy
       - name: Trivy FS Scan
         run: trivy fs --format table -o fs-report.json .
   ```  

---

### **Phase 3: Self-Hosted Runner Setup**  
1. **Add VM as Runner**:  
   - **Steps**:  
     - Launch Ubuntu VM (e.g., AWS EC2 `t2.medium`).  
     - In GitHub: **Settings → Actions → Runners → New Self-Hosted Runner**.  
     - Run provided commands on VM:  
       ```bash
       mkdir actions-runner && cd actions-runner
       curl -o runner.tar.gz -L https://github.com/actions/runner/releases/download/v2.XXX/actions-runner-linux-x64-2.XXX.tar.gz
       tar xzf runner.tar.gz
       ./config.sh --url <repo-url> --token <token> --labels self-hosted
       ./run.sh
       ```  
   - **Label**: Use `self-hosted` in workflow:  
     ```yaml
     runs-on: self-hosted
     ```  

2. **Install Tools on Runner**:  
   - Docker, Maven, Trivy:  
     ```bash
     sudo apt update
     sudo apt install -y maven trivy docker.io
     sudo usermod -aG docker $USER
     newgrp docker
     ```  

---

### **Phase 4: Pipeline Jobs**  

#### **1. SonarQube Analysis**  
1. **Set Up SonarQube Server**:  
   - Launch VM → Install Docker → Run SonarQube:  
     ```bash
     docker run -d -p 9000:9000 --name sonar sonarqube:community
     ```  
   - Generate token in SonarQube UI.  

2. **Add to Pipeline**:  
   ```yaml
   - name: SonarQube Scan
     uses: sonarsource/sonarqube-scan-action@master
     env:
       SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
       SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
   ```  

#### **2. Docker Build & Push**  
- **Steps**:  
  - Upload JAR artifact in `build` job.  
  - Download artifact in `docker` job.  
  - Build/push to Docker Hub:  
    ```yaml
    - name: Docker Build/Push
      uses: docker/build-push-action@v4
      with:
        push: true
        tags: username/bank-app:latest
        secrets: |
          ${{ secrets.DOCKER_USERNAME }}
          ${{ secrets.DOCKER_PASSWORD }}
    ```  

#### **3. Kubernetes Deployment**  
1. **Set Up EKS Cluster**:  
   - Use Terraform to create EKS (example repo: `terraform-aws-eks`).  
   - Configure `kubectl`:  
     ```bash
     aws eks --region <region> update-kubeconfig --name <cluster-name>
     ```  

2. **Deploy via Pipeline**:  
   ```yaml
   deploy:
     needs: docker
     runs-on: self-hosted
     steps:
       - uses: actions/checkout@v4
       - name: Install kubectl
         uses: azure/setup-kubectl@v3
       - name: Deploy
         run: |
           mkdir -p ~/.kube
           echo "${{ secrets.KUBE_CONFIG }}" > ~/.kube/config
           kubectl apply -f k8s-manifest.yaml
   ```  

---

### **Final Notes**  
- **Parallel Jobs**: Use `needs` to sequence jobs.  
- **Secrets Management**: Store credentials in GitHub Secrets.  
- **Debugging**: Check logs in **Actions → Workflow Runs**.  

**Batch 9 Enrollment**: Starts March 25th (Discord community, 20+ projects, interview prep).  

--- 

In GitHub Actions, the line `- uses: actions/checkout@v4` is a step in a workflow that uses the `actions/checkout` action (version `v4`) to check out your repository's code.

### Breakdown:
1. **`uses: actions/checkout@v4`**  
   - This tells GitHub Actions to use a pre-built action called `checkout`, hosted in the `actions` organization on GitHub.  
   - `@v4` specifies that it should use version 4 of this action (which is the latest stable version as of my last update).  

2. **What does `actions/checkout` do?**  
   - It checks out your repository's code into the workflow's workspace, allowing subsequent steps to access and work with the files.  
   - Without this step, your workflow wouldn't have access to your code.  

### Example Usage in a Workflow:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # Checks out the repo
      - run: echo "Now I can access the code!"
```

### Key Points:
- This action is almost always the first step in a workflow that needs to interact with the repository's code.  
- It supports features like checking out a specific branch, fetching all history, or using a personal access token for private repos.  

Would you like help configuring additional options for `actions/checkout`?
