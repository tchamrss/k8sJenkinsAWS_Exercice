EKS-Cluster erstellen

Jetzt erstellen wir den neuen Cluster mit genau 3 EC2-Nodes.

eksctl create cluster \
  --name eks-company-cluster \
  --region eu-central-1 \
  --version 1.34 \
  --nodegroup-name company-nodes \
  --node-type t3.small \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 3


Was passiert dabei?
1. AWS erstellt den EKS-Control-Plane.
2. eksctl erstellt die benötigte Netzwerkinfrastruktur.
3. AWS erstellt eine Managed Nodegroup mit 3 EC2-Instanzen.
4. Die Nodes werden mit deinem Kubernetes-Cluster verbunden.
5. Deine lokale kubeconfig wird aktualisiert.

Cluster überprüfen

eksctl get cluster --region eu-central-1

kubectl get nodes

Fargate Profile erstellen


eksctl create fargateprofile \
  --cluster eks-company-cluster \
  --region eu-central-1 \
  --name company-fargate-profile \
  --namespace fargate


Namespace erstellen

kubectl create namespace fargate

Fargate Profile überprüfen:

eksctl get fargateprofile \
  --cluster eks-company-cluster \
  --region eu-central-1


Exercise 2: Deploy MySQL and phpMyAdmin

Exercise 2 – Schritt 1: Richtigen Cluster auswählen
Zuerst stellen wir sicher, dass wir auf dem neuen Cluster arbeiten und nicht versehentlich auf demo-cluster.

kubectl config current-context

Falls dort nicht eks-company-cluster steht, wechseln wir mit:
aws eks update-kubeconfig \
  --region eu-central-1 \
  --name eks-company-cluster



Schritt 2: Namespace für MySQL und phpMyAdmin
Wir verwenden einen eigenen Namespace namens database.

kubectl create namespace database

Schritt 3: Helm überprüfen
helm version

Exercise 2 – Schritt 4: MySQL Helm Repository hinzufügen
Wir verwenden das Bitnami-Repository für das Helm Chart. Wichtig: Die Images wurden in das Repository bitnamilegacy verschoben. Deshalb müssen wir das Image in unserer values.yaml anpassen.

Zuerst fügen wir das Helm-Repository hinzu:

helm repo add bitnami https://charts.bitnami.com/bitnami

Danach aktualisieren wir die Chart-Liste:

helm repo update

Prüfen, ob das MySQL Chart verfügbar ist
helm search repo bitnami/mysql --versions

Schritt 6: MySQL Values-Datei erstellen
Wir verwenden zunächst Chart-Version 14.0.3.

Erstelle auf deinem Mac im Projektverzeichnis den Ordner und die Datei:
mkdir -p mysql
touch mysql/values.yaml

Schritt 6: MySQL-Konfiguration prüfen

Bevor wir MySQL installieren, prüfen wir, ob Helm die Konfiguration akzeptiert.

Führe bitte diesen Befehl aus:
helm template mysql bitnami/mysql \
  --version 14.0.3 \
  --namespace database \
  --values mysql/values.yaml

Schritt 7: MySQL im EKS-Cluster installieren

Führe jetzt diesen Befehl aus:
helm install mysql bitnami/mysql \
  --version 14.0.3 \
  --namespace database \
  --values mysql/values.yaml



phpMyAdmin vorbereiten
Als Nächstes installieren wir phpMyAdmin im Namespace database, damit du MySQL über eine Weboberfläche verwalten kannst.

Zuerst prüfen wir, ob Helm das phpMyAdmin-Chart im Bitnami-Repository findet:
helm search repo bitnami/phpmyadmin


Konfigurationsdatei erstellen
Erstelle zunächst einen Ordner für phpMyAdmin:
mkdir -p phpmyadmin

Helm-Chart-Konfiguration prüfen
Führe aus:
helm show values bitnami/phpmyadmin --version 20.0.0 | grep -A 8 -E '^db:|^service:|^image:'

phpMyAdmin mit Helm installieren
Führe jetzt diesen Befehl aus:
helm install phpmyadmin bitnami/phpmyadmin \
  --version 20.0.0 \
  --namespace database \
  --values phpmyadmin/values.yaml

 Deploy your Java application

erstelle deployment.yaml and service.yaml


Im Namespace fargate existiert noch kein Secret. Wir erstellen jetzt das Docker-Hub-Secret mit deinem gewünschten Namen aws_fargate-registry-key.
Schritt 6: Docker-Hub-Zugangsdaten als Kubernetes-Secret anlegen
Führe diesen Befehl aus und ersetze die Platzhalter durch deine Docker-Hub-Zugangsdaten:
kubectl create secret docker-registry aws-fargate-registry-key \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=tchamrss \
  --docker-password=dockerhubpw \
  --namespace=fargate


ava-Anwendung auf EKS deployen
Jetzt wenden wir beide Kubernetes-Dateien auf deinen neuen EKS-Cluster an.

Führe diesen Befehl aus:
kubectl apply -f java-maven-app/kubernetes/


Damit werden erstellt:
1. Das Deployment java-maven-app mit 3 Replicas.
2. Der Service java-maven-app im Namespace fargate.

 Use ECR as Docker repository and Automate deployment

 aws ecr create-repository \
  --repository-name java-maven-app \
  --region eu-central-1


Schritt 4: ECR-Berechtigungen für Fargate prüfen
Fargate benötigt Leserechte auf ECR, damit es die Images herunterladen kann.

Führe diesen Befehl aus:
aws iam list-attached-role-policies \
  --role-name eksctl-eks-company-cluster--FargatePodExecutionRole-aDIXgc0a5ilM




