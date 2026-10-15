# AttendEase — Student Attendance Analytics System
### CCA 2: CI/CD Pipeline with Jenkins, Docker and Kubernetes (Minikube)

AttendEase lets a student record attendance per subject and instantly see whether
they are safely above the required percentage, instead of calculating it by hand.

## 1. Project chosen

AttendEase is an existing academic project (Node.js + Express + MongoDB Atlas, vanilla JS frontend).

- Student registration/login (JWT, bcrypt)
- Subjects with a custom required-attendance threshold
- Present/Absent marking, dashboard with Safe / Warning / Critical status
- "Can I miss the next lecture?" predictor

Attendance % = (lectures attended / total lectures held) × 100.

## 2. Repository layout

```
attendease/
├── public/               # frontend
├── server/               # Express API, models, routes, utils
├── tests/                # Jest unit tests
├── k8s/
│   ├── deployment.yaml   # Deployment (2 replicas, secrets via env)
│   ├── service.yaml      # NodePort service
│   └── secret.example.yaml
├── Dockerfile
├── Jenkinsfile
└── package.json
```

## 3. Pipeline stages (Jenkinsfile)

| # | Stage | What it does |
|---|-------|--------------|
| 1 | Checkout | Jenkins pulls the latest code from GitHub |
| 2 | Build & Test | Inside a `node:18-alpine` container: `npm ci`, `node --check` (syntax check), `npm test` (Jest) |
| 3 | Docker Build | Builds `<dockerhub-user>/attendease:<BUILD_NUMBER>` and `:latest` |
| 4 | Push Image | Logs in with Jenkins credentials and pushes both tags to Docker Hub |
| 5 | Deploy to Minikube | Creates the K8s Secret from Jenkins credentials, applies Deployment (with the new tag) and Service, waits for rollout |
| 6 | Verify | Lists pods, deployment, service; gets the URL and curls the app |

The pipeline also polls GitHub every ~2 minutes (`pollSCM`), so a push triggers a run automatically.

## 4. Setting up Jenkins on AWS

1. Launch an **EC2** instance: Ubuntu 22.04, at least 2 vCPU / 4 GB RAM (t3.medium), 20 GB disk.
2. Security group inbound: **22** (SSH), **8080** (Jenkins), **30080** (app NodePort / or use SSH tunnel).
3. Install Java, Jenkins, Docker, kubectl, Minikube:

```bash
sudo apt update && sudo apt install -y fontconfig openjdk-17-jre docker.io curl git conntrack

# Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update && sudo apt install -y jenkins

# kubectl + minikube
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# let jenkins use docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker $USER
sudo systemctl restart jenkins
```

4. Start Minikube **as the jenkins user** so its kubeconfig is usable by the pipeline:

```bash
sudo su - jenkins
minikube start --driver=docker
kubectl get nodes
exit
```

5. Open `http://<EC2-public-IP>:8080`, unlock Jenkins (`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`), install suggested plugins plus **Docker Pipeline**.

### Jenkins credentials (Manage Jenkins → Credentials → Global)

| ID | Type | Value |
|----|------|-------|
| `dockerhub-creds` | Username with password | Docker Hub username + access token |
| `mongodb-uri` | Secret text | MongoDB Atlas connection string |
| `jwt-secret` | Secret text | Long random string |

MongoDB Atlas → Network Access must allow the EC2 instance's public IP.

### Pipeline job

New Item → **Pipeline** → *Pipeline script from SCM* → Git → repository URL → branch `main` → Script Path `Jenkinsfile`.
Edit `DOCKERHUB_USER` at the top of the `Jenkinsfile` to your Docker Hub username before the first run.

## 5. How the image is pushed

The Jenkins `Push Image` stage uses `withCredentials` to read `dockerhub-creds`, runs
`docker login --password-stdin`, then pushes `attendease:<build number>` (traceable tag) and `attendease:latest`.
Credentials are never written in the repo or printed in logs.

## 6. Deployment and access

Kubernetes pulls the image from Docker Hub. MongoDB URI and JWT secret are injected from a
Kubernetes Secret created by the pipeline. The Service is `NodePort` (30080).

```bash
kubectl get pods,deployment,svc
minikube service attendease-service --url
```

To open from your laptop, either use an SSH tunnel to the URL printed above, or on the EC2 host run:

```bash
kubectl port-forward --address 0.0.0.0 svc/attendease-service 3000:80
# then browse http://<EC2-public-IP>:3000  (open port 3000 in the security group)
```

## 7. Second run (code change demo)

Changed: *(fill in, e.g. "dashboard heading text changed from 'Dashboard' to 'My Attendance Dashboard'")*.
Committed and pushed to `main` → Jenkins ran again → new image tag `<N+1>` pushed →
Deployment rolled out new pods → refreshed browser shows the change.

## 8. Screenshots

Add to a `screenshots/` folder: Jenkins stage view (run 1 and run 2), credentials page,
Docker Hub tags, `kubectl get pods,deploy,svc`, app in browser (before/after change).

## 9. Local run

```bash
npm install
cp .env.example .env     # fill in MONGODB_URI and JWT_SECRET
npm run dev
npm test
```

## 10. Viva notes

- **CI**: automatically build and test on every push. **CD**: automatically ship the passing build.
- **Why Docker?** Same environment everywhere; Kubernetes needs an image.
- **Why a registry?** Minikube pulls the image from Docker Hub; Jenkins pushes it there.
- **Why a Secret?** Keeps DB URI and JWT key out of Git and the image.
- **NodePort**: exposes the service on a port of the node so it is reachable from outside the cluster.
- **Tag = BUILD_NUMBER**: every deployment is traceable and rollbacks are possible (`kubectl rollout undo`).
- **If tests fail**: the pipeline stops, so no image is pushed or deployed.
- **No build step in Node**: the "build" is `npm ci` plus a syntax check, then the Docker build.
