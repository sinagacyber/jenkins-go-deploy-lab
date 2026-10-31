# Jenkins Go Binary Deployment Lab

Mini project to learn a realistic **manual Jenkins deployment** on Windows + Docker Desktop.

## What this lab does

1. Jenkins checks out the `main` branch from Git.
2. Jenkins runs `go test ./...`.
3. Jenkins builds a Linux AMD64 binary.
4. Jenkins backs up the previously deployed binary.
5. Jenkins replaces the binary and starts it as a background process.
6. Jenkins calls `/health` and `/version` to verify the deployment.

Jenkins and the demo API run inside the same Linux container in this lab. This keeps the exercise close to the selected same-server binary deployment model. It is a learning setup, not a production container design. In production, the app should normally run as its own systemd service, and Jenkins should use a dedicated deployment account with least-privilege permissions.

## Requirements

- Windows 10/11
- Docker Desktop, using Linux containers
- Git for Windows
- A GitHub account (or another Git server)

## 1. Create a remote Git repository

Create an **empty public repository** named `jenkins-go-deploy-lab` for this learning exercise. Do not add a README, `.gitignore`, or license on the Git hosting website because those files already exist here.

Open PowerShell in this folder and run:

```powershell
git init
git add .
git commit -m "Initial Go deployment lab"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/jenkins-go-deploy-lab.git
git push -u origin main
```

Replace `YOUR-USERNAME` with your GitHub username. The repository is public only to make the Jenkins learning setup simpler; never put passwords, tokens, private keys, or production configuration in this demo repository.

## 2. Build and start Jenkins

From this folder in PowerShell:

```powershell
docker compose build
docker compose up -d
```

The first build downloads Jenkins, Go, and Jenkins plugins, so it needs internet access and may take a few minutes.

Check the container:

```powershell
docker compose ps
```

Read the initial Jenkins password:

```powershell
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open http://localhost:8080, enter the password, install the suggested plugins if prompted, and create your administrator account. The Git and Pipeline plugins are already included in the custom image.

## 3. Create the Jenkins Pipeline job

1. In Jenkins, choose **New Item**.
2. Name it `go-binary-deploy-lab`.
3. Select **Pipeline**, then click **OK**.
4. Under **Pipeline**, set **Definition** to **Pipeline script from SCM**.
5. Set **SCM** to **Git**.
6. Set **Repository URL** to your public GitHub repository URL.
7. Set **Branch Specifier** to `*/main`.
8. Set **Script Path** to `Jenkinsfile`.
9. Save.

No Git credentials are needed for a public repository. For a private repository, add credentials in Jenkins rather than putting a token in the repository URL or Jenkinsfile.

## 4. Run a manual deployment

Open the job and click **Build Now**. Open the build number, then **Console Output**. The expected stages are:

- Checkout
- Test
- Build Linux Binary
- Deploy to Local Linux Runtime
- Verify Deployment

When the build succeeds, open:

- http://localhost:8081/ — application greeting
- http://localhost:8081/health — health status
- http://localhost:8081/version — build number, Git commit, and build time

## 5. Inspect the deployed files

The deployment files live in a Docker named volume. Inspect them with:

```powershell
docker compose exec jenkins sh -c "ls -lah /opt/demo-app && ls -lah /opt/demo-app/backup"
```

View the application log:

```powershell
docker compose exec jenkins sh -c "tail -n 50 /opt/demo-app/demo-api.log"
```

## 6. Try a second deployment

Edit the message in `main.go`, for example change `Hello! This Go app was deployed by Jenkins.` to `Hello from deployment number two!`.

Then run:

```powershell
git add .
git commit -m "Change demo greeting"
git push origin main
```

Return to Jenkins and click **Build Now** again. Refresh http://localhost:8081/. You should see the new message. Jenkins should also have created a backup of the previous binary.

## 7. Stop the lab

```powershell
docker compose down
```

This stops the Jenkins container but preserves its data and the deployment volume. To remove the containers **and all Jenkins/deployment volumes** and start from scratch:

```powershell
docker compose down -v
```

Be careful: `-v` deletes your Jenkins configuration, job history, and deployed demo binary/backups.

## Important notes

- This lab builds `linux/amd64`. That is appropriate for a typical Intel/AMD x86-64 Linux server. Confirm the production server architecture before using the same setting.
- The demo app listens on port `8081`; Jenkins UI uses port `8080`.
- This lab uses a background process and PID file to make the restart visible and easy to learn. On the actual Linux server, we should replace this part with a `systemd` service (`systemctl restart ...`) and add an explicit rollback/health-check strategy.
- The Jenkins controller and app share one container only for local learning. Do not copy this single-container process layout as a production design.
