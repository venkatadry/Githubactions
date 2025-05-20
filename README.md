https://medium.com/@inderjotsingh141/github-actions-to-ecs-container-b9a492fc95bc

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

if you have to run multiple commands
     run: |
        sudo apt-get install -y wget apt-transport-https gnupg lsb-release
        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
        echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
        sudo apt-get update -y
        sudo apt-get install -y trivy



        test:
    runs-on: self-hosted
    needs: security-check
    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    - name: Unit Test Cases
      run: mvn test

  build_project_and_sonar_scan:
    runs-on: self-hosted
    needs: test **Here one the above test job is run it will run**
    steps:
----
In GitHub Actions, **labels** are used to categorize and organize workflows, jobs, or steps. They can be applied in different contexts:

### 1. **Self-Hosted Runner Labels**
   - Used to direct jobs to specific self-hosted runners.
   - Example:
     ```yaml
     jobs:
       deploy:
         runs-on: [self-hosted, linux, production]
     ```
   - Here, `linux` and `production` are labels assigned to a self-hosted runner.

### 2. **Matrix Strategy Labels**
   - Used in matrix strategies to define different configurations.
   - Example:
     ```yaml
     jobs:
       test:
         strategy:
           matrix:
             os: [ubuntu-latest, windows-latest]
             node: [14, 16]
         runs-on: ${{ matrix.os }}
     ```
   - Here, `ubuntu-latest` and `windows-latest` are labels for different OS environments.

### 3. **Labeling Issues & PRs via Workflows**
   - GitHub Actions can automatically add/remove labels to issues or PRs using `actions/github-script` or `actions-ecosystem/action-add-labels`.
   - Example:
     ```yaml
     - name: Add Label
       uses: actions-ecosystem/action-add-labels@v1
       with:
         labels: bug, high-priority
     ```

### 4. **Filtering Workflows Based on PR/Issue Labels**
   - You can trigger workflows conditionally based on labels.
   - Example:
     ```yaml
     on:
       pull_request:
         types: [labeled]
     jobs:
       build:
         if: contains(github.event.pull_request.labels.*.name, 'urgent')
         runs-on: ubuntu-latest
     ```

Would you like more details on a specific use case? 🚀
**The runnder status should be green and Idle***

In Ubuntu, when you install Docker, it creates a `docker` group to manage access to the Docker daemon. By default, only the **root** user or members of the **docker** group can run Docker commands.

### How to Add a User to the Docker Group in Ubuntu

1. **Check if the `docker` group exists:**
   ```sh
   sudo groupadd docker
   ```
   (This command will create the group if it doesn't exist.)

2. **Add your user to the `docker` group:**
   ```sh
   sudo usermod -aG docker $USER
   ```
   - `-aG` means "append to group."
   - Replace `$USER` with the username if needed.

3. **Apply the changes:**
   - Log out and log back in, or run:
     ```sh
     newgrp docker
     ```
   - This refreshes group permissions without requiring a logout.

4. **Verify Docker access:**
   ```sh
   docker run hello-world
   ```
   - If it runs successfully, the user has Docker access.

### Important Security Note:
- Being in the `docker` group grants **root-equivalent privileges** because Docker allows container manipulation that can affect the host system.
- Only trusted users should be added to this group.

### Alternative (Not Recommended):
- Some users run Docker with `sudo` (e.g., `sudo docker ...`), but this is less secure and more cumbersome.

Would you like help troubleshooting any specific Docker permission issues?

----
git repo
GitHub actions
add vm runner
cicd yaml--:a)security check
  b)test cases
c)build and publish artificat
d)build scan docker image and publish
e)deploy to k8s

like jenkins maste ro slave we have runner which runs cicd pipeline

https://github.com/jaiswaladi246/Github-Actions-Project/blob/main/.github/workflows/cicd.ymlinside jobs you can have multiple jobs
Each job will be running in a isolated environment

how we can run jobs paralley?/one afte the ther?

how do you check the deploy jobs in runner server?

add docker user to group to run docker images

secrets and tokens


runner
-shared runner- frree cost
- private runner

shared runner - if you want to run in ubuntu
 you can use ubuntu-latest
you cannot have acces to it

- private runner

phase

