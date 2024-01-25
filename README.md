# Deploying a Netflix CLone on EKS using ArgoCD

## Launching an t2.large Instance:

I decided to launch a t2.large instance since we install Jenkins,docker,SonarQube and Trivy on this.

## Installing Docker:
```
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
```
Here we gave executable permissions for docker.sock for all users, so that processes can talk to docker deamon. But generally not recommended.

## API Key:
Next we use TMDB (the movie database) API key to fetch information about movies form their data base and integrate this with our clone application to show movies and other infromation on our clone application. The API key is used for authentication when our clone tries to fetch info from their DB.

## Install SonarQube and Trivy:
### SonarQube:
It checks our program to find any mistakes or bugs. It helps us make sure our code is clean, efficient, and doesn't have any issues that could cause problems.
```
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
(By default username & password is admin)

### Trivy:
It looks at the files and images we use and checks if there are any security threats. It helps us know if there are any weaknesses that could be exploited.
```
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy        

```
To scan our image using trivy
```
trivy image <imageid>
```
## Install Jenkins for Automation:
Jenkins need Java as run time so first we will install java

```
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
openjdk version "17.0.8" 2023-07-18
OpenJDK Runtime Environment (build 17.0.8+7-Debian-1deb12u1)
OpenJDK 64-Bit Server VM (build 17.0.8+7-Debian-1deb12u1, mixed mode, sharing)

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
Jenkins  runs on localhost 8080.

### Install Necessary Plugins in Jenkins:
Goto Manage Jenkins →Plugins → Available Plugins →Install below plugins

* Eclipse Temurin Installer 
* SonarQube Scanner 
* NodeJs Plugin

The Eclipse Temurin Installer in Jenkins ensures a smooth, automated, and consistent Java environment for Jenkins instances.

### Configure Java and Nodejs in Global Tool Configuration
Goto Manage Jenkins → Tools → Install JDK(17) and NodeJs(16)→ Click on Apply and Save
### Need for Node.js:
To run JavaScript code for our front-end application: Node.js provides the JavaScript runtime that allows us to execute JS code to build and run our front-end app and see it functioning.
To install front-end packages from npm: Node.js comes bundled with the npm package manager. This allows our front-end code to declare dependencies and install hundreds of reusable JS packages like React, Vue, lodash etc. from the npm registry. These packages provide critical functionality that we can rely on instead of building everything from scratch. 

### Next Install Dependency-Check and Docker Tools in Jenkins

We need to install and configure Dependencey-check tool OWASP as it scan all the dependencies which We downloaded from npm are safe and not vulnerable.


## Monitoring Tools
Install Prometheus and Grafana to collect metrics and display it in Grafana. To do this launch a new t2.medium instance

### Installing Prometheus:

```
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz

```
Extract Prometheus files, move them, and create directories
```
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

```
The ownership is set so that the prometheus can acces the directories using the user prometheus which we created  earlier. By doing this we are not giving explicit permissions to other users and groups to access them.

* Even though we installed prometheus the service is not yet identified. fo that we have to create the servuce file in systemd

```
sudo nano /etc/systemd/system/prometheus.service
```
In the above created file add

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```
Now the proometheus will be running on 
```
http://<your-server-ip>:9090
```
## Node exporter

In general Prometheus is not able to collect metrics directly. Node exporter is needed for that purpose. Node exporter is such an application that collects metrics form os level. Prometheus then scrapes/collects the metrics from the node exporter in a timely pull based manner and displays the metrics.

```
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```
```
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

```
sudo nano /etc/systemd/system/node_exporter.service
```
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```
## Configure prometheus
To configure Prometheus to scrape metrics from Node Exporter and Jenkins, you need to modify the prometheus.yml.

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

```


Check the validity of the configuration file:
```
promtool check config /etc/prometheus/prometheus.yml
```

Reload the Prometheus configuration without restarting:
```
curl -X POST http://localhost:9090/-/reload
```

You can access Prometheus targets at:

```http://<your-prometheus-ip>:9090/targets```

## Install Grafana on Ubuntu 22.04 and Set it up to Work with Prometheus

First, ensure that all necessary dependencies are installed
```
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
```

Add the GPG key for Grafana
```
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

Add the repository for Grafana stable releases:

```
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```
Update the package list and install Grafana:
```
sudo systemctl enable grafana-server
sudo systemct  start grafana-server
sudo systemctl status grafana-server
```
```http://<your-server-ip>:3000```

### Change the Default Password
Upon your initial login to Grafana, you'll be prompted to change the default password for security. Follow the prompts to set a new password.
### Add Prometheus Data Source
To visualize metrics, follow these steps:

Click on the gear icon in the left sidebar for the "Configuration" menu.
Select "Data Sources."
Click "Add data source."
Choose "Prometheus" as the data source type.
In the "HTTP" section, set the "URL" to http://localhost:9090 .
Click "Save & Test" to ensure the data source is working.


### Import a Dashboard

For easier metric visualization, import a pre-configured dashboard:

Click the "+" (plus) icon in the left sidebar to open the "Create" menu.
Select "Dashboard."
Choose "Import" dashboard.
Enter the dashboard code 1860.
Click "Load."
Select the added data source (Prometheus) from the dropdown.
Click "Import."

Now, your Grafana dashboard is set up to display metrics from Prometheus. Grafana offers powerful customization for your monitoring needs.

### ArgoCD on EKS

Create a eks cluster and once the nodegroup is up install argocd inside that cluster using

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml

```
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
After installing ArgoCD, you must configure your GitHub repository as a source for your application deployment. This usually entails configuring the connection to your repository and specifying the source for your ArgoCD application. The specific steps will depend on your setup and needs.


Create an ArgoCD Application:

name: Provide a name for your application.
destination: Determine where your application should be deployed.
project: Indicate which project the application belongs to.
source: Configure the source of your application, including the GitHub repository URL, revision, and path to the application within the repository.
SyncPolicy: Set the sync policy, which includes automatic syncing, pruning, and self-healing.

To access the app, make sure port 30007 is open in your security group, then open a new tab and paste your NodeIP:30007. Your app should now be running.

Once everything is done delete the instances and delete the cluster.


