Sales Forecast Using DevOps
▶️ Highly scalable Cloud-native Machine Learning system ◀️

Table of contents
Overview
Key Features
Tools / Technologies
Development environment
How things work
How to setup
With Docker Compose
With Kubernetes/Helm (Local cluster)
With Kubernetes/Helm (on GCP)
Cleanup steps
Important note on MLflow on Cloud
References / Useful resources
My notes
Overview
"Sales Forecast Using DevOps" delivers a full-stack, production-ready solution designed to streamline the entire sales forecasting system – from development and deployment to continuous improvement. It offers flexible deployment options, supporting both on-premises environments (Docker Compose, Kubernetes) and cloud-based setups (Kubernetes, Helm), ensuring adaptability to your infrastructure.



Key Features
Dual-Mode Inference: Supports both batch and online inference modes, providing adaptability to various use cases and real-time prediction needs.
Automated Forecast Generation: Airflow DAGs orchestrate weekly model training and batch predictions, with the ability for on-demand retraining based on the latest data.
Data-Driven Adaptability: Kafka handles real-time data streaming, enabling the system to incorporate the latest sales information into predictions. Models are retrained on demand to maintain accuracy.
Scalable Pipeline and Training: Leverages Spark and Ray for efficient data processing and distributed model training, ensuring the system can handle large-scale datasets and training.
Transparent Monitoring: Ray and Grafana provide visibility into training performance, while Prometheus enables system-wide monitoring.
User-Friendly Interface: Streamlit offers a clear view of predictions. MLflow tracks experiments and model versions, ensuring reproducibility and streamlined updates.
Best-Practices Serving: Robust serving stack with Nginx, Gunicorn, and FastAPI for reliable and performant model deployment.
CI/CD Automation: GitHub Actions streamline the build and deployment process, automatically pushing images to Docker Hub and GCP.
Cloud-native, Scalability and Flexibility: Kubernetes and Google Cloud Platform ensure adaptability to growing data and workloads. The open-source foundation (Docker, Ray, FastAPI, etc.) offers customization and extensibility.
Tools / Technologies
Note: Most of the service ports can be found and customized in the .env file at the root of this repository (or values.yaml and sfmlops-helm/templates/global-configmap.yaml for Kubernetes and Helm).

Platform: Docker, Kubernetes, Helm
Cloud platform: Google Cloud Platform
Experiment tracking / Model registry: MLflow
Pipeline orchestrator: Airflow
Model distributed training and scaling: Ray
Reverse proxy: Nginx and ingress-nginx (for Kubernetes)
Web Interface: Streamlit
Machine Learning service deployment: FastAPI, Uvicorn, Gunicorn
Databases: PostgreSQL, Prometheus
Database UI for Postgres: pgAdmin
Overall system monitoring & dashboard: Grafana
Distributed data streaming: Kafka
Forecast modeling framework: Prophet
Stream processing: Spark Streaming
CICD: GitHub Actions
Development environment
Docker (ref: Docker version 24.0.6, build ed223bc)
Kubernetes (ref: v1.27.2 (via Docker Desktop))
Helm (ref: v3.14.3)
How things work
After you start up the system, the data producer will read and store the data of the last 5 months from services/data-producer/datasets/rossman-store-sales/train_exclude_last_10d.csv to Postgres. It does this by modifying the last date of the data to be YESTERDAY. Afterward, it will keep publishing new messages (from train_only_last_19d.csv in the same directory), technically TODAY data, to a Kafka topic every 10 seconds (infinite loop).
There are two main DAGs in Airflow:
Daily DAG:
>> Ingest data from this Kafka topic
>> Process and transform with Spark Streaming
>> Store it in Postgres
Weekly DAG:
>> Pull the last four months of sales data from Postgres
>> Use it for training new Prophet models, with Ray (1,1115 models in total), which are tracked and registered by MLflow
>> Use these newly trained models to predict the forecast of the upcoming week (next 7 days)
>> Store the forecasts in Postgres (another table)
During training, you can monitor your system and infrastructure with Grafana and Prometheus.
By default, the data stream from topic sale_rossman_store gets stored in rossman_sales table and forecast results in forecast_results table, you can use pgAdmin to access it.
After the previous steps are executed successfully, you/users can now access the Streamlit website proxied by Nginx.
This website fetches the latest 7 predictions (technically, the next 7 days) for each store and each product and displays them in a good-looking line chart (thanks to Altair)
From the website, users can view sales forecast of any product from any store. Notice that the subtitle of the chart contains the model ID and version.
Since these forecasts are made weekly, whether users access this website on Monday or Wednesday, they will see the same chart. If, during the week, the users somehow feel like the forecast prediction is out of sync or outdated, they can trigger retraining for a specific model of that product and store.
When the users click a retrain button, the website will submit a model training job to the training service which then calls Ray to retrain this model. The retraining is pretty fast, usually done in under a minute, and it follows the same training strategy as the weekly training DAG (but of course, with the newest data possible).
Right after retraining is done, users can select a number of future days to predict and click a forecast button to request the forecasting service to use the latest model to make forecasts.
The result of new forecasts is then displayed in the line chart below. Notice that the model version number increased! Yoohoo! (note: For simplicity, this new forecast result won't be stored anywhere.)
How to setup
Prerequisites: Docker, Kubernetes, and Helm

With Docker Compose
(Optional) In case you want to build (not pulling images):
docker-compose build
docker-compose -f docker-compose.yml -f docker-compose-airflow.yml up -d
Sometimes it can freeze or fail the first time, especially if your machine is not that high in spec (like mine T_T). But you can wait a second, try the last command again and it should start up fine.
That's it!
Note: Most of the services' restart is left unspecified, so they won't restart on failures (because sometimes it's quite resource-consuming during development, you see I have a poor laptop lol).

With Kubernetes/Helm (Local cluster)
The system is quite large and heavy... I recommend running it locally just for setup testing purposes. Then if it works, just go off to the cloud if you want to play around longer OR stick with Docker Compose (it went smoother in my case)

Install Helm
bash install-helm.sh
Create airflow namespace:
kubectl create namespace airflow
Deploy the main chart:
Fetch all dependencies
cd sfmlops-helm
helm dependency build
helm -n mlops upgrade --install sfmlops-helm ./ --create-namespace -f values.yaml -f values-ray.yaml
Deploy Kafka:
(1st time only)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm -n kafka upgrade --install kafka-release oci://registry-1.docker.io/bitnamicharts/kafka --create-namespace --version 23.0.7 -f values-kafka.yaml
Deploy Airflow:
(1st time only)
helm repo add apache-airflow https://airflow.apache.org
helm -n airflow upgrade --install airflow apache-airflow/airflow --create-namespace --version 1.13.1 -f values-airflow.yaml
Sometimes, you might get a timeout error from this command (if you do, it means your machine spec is too poor for this system (like mine lol)). It's totally fine. Just keep checking the status with kubectl, if all resources start up correctly, go with it otherwise try running the command again.
Deploy Prometheus and Grafana:
(1st time only)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm -n monitoring upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack  --create-namespace --version 57.2.0 -f values-kube-prometheus.yaml
Forward port for Grafana:
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
OR assign grafana.service.type: LoadBalancer in values-kube-prometheus.yaml
One of the good things about kube-prometheus-stack is that it comes with many pre-installed/pre-configured dashboards for Kubernetes. Feel free to explore!
That's it! Enjoy your highly scalable Machine Learning system for Sales forecasting! ;)
Note: If you want to change namespace kafka and/or release name kafka-release of Kafka, please also change them in values.yaml and KAFKA_BOOTSTRAP_SERVER env var in values-airflow.yaml. They are also used in templating.

Note 2: In Docker Compose, Ray has already been configured to pull the embedded dashboards from Grafana. But in Kubernetes, this process involves a lot more manual steps. So, I intentionally left it undone for ease of setup of this project. You can follow the guide here if you want to anyway.

With Kubernetes/Helm (on GCP)
Prerequisites: GKE Cluster (Standard cluster, NOT Autopilot), Artifact Registry, Service Usage API, gcloud cli

Follow this Medium blog. Instead of using the default Service Account (as done in the blog), I recommend creating a new Service Account with Owner role for a quick and dirty run (but of course, please consult your cloud engineer if you have security concerns).
Download your Service Account's JSON key
Activate your Service Account:
gcloud auth activate-service-account --key-file=<PATH_TO_JSON_KEY>
Connect local kubectl to cloud:
gcloud container clusters get-credentials <GKE_CLUSTER_NAME> --zone <GKE_ZONE> --project <PROJECT_NAME>
Now kubectl (and helm) will work in the context of the GKE environment.
Follow the steps in With Kubernetes/Helm (Local cluster) section
If you face a timeout error when running helm commands for airflow or the system struggles to set up and work correctly, I recommend trying to upgrade your machine type in the cluster.
Note: For the machine type of node pool in the GKE cluster, from experiments, e2-medium (default) is not quite enough, especially for Airflow and Ray. In my case, I went for e2-standard-8 with 1 node (explanation on why only 1 node is in Important note on MLflow on Cloud section). I also found myself the need to increase the quota for PVC in IAM too.

Cleanup steps
helm uninstall sfmlops-helm -n mlops
helm uninstall kafka-release -n kafka
helm uninstall airflow -n airflow
helm uninstall kube-prometheus-stack -n monitoring
Important note on MLflow on Cloud
In this setting, I set the MLflow's artifact path to point to a local path. Internally, MLflow expects this path to be accessible from both MLflow client and server (honestly, I'm not a fan of this model either). It is meant to be an object storage path like S3 (AWS) or Cloud Storage (GCP). For a full on-premises experience, we can create a Docker volume and mount it to the EXACT same path on both client and server to address this. In a local Kubernetes cluster, we can do the same thing by creating a PVC with accessModes: ReadWriteOnce (in sfmlops-helm/templates/mlflow-pvc.yaml).

However for on-cloud Kubernetes with a typical multi-node cluster, if we want the PVC to be able to read and write across nodes, we need to set accessModes: ReadWriteMany. Most cloud providers DO NOT support this type of PVC and recommend using centralized storage instead. Therefore, if you want to just try it out and run for fun, you can use this exact setting and create a single-node cluster (which will behave similarly to a local Kubernetes cluster, just on the cloud). For a real production environment, please create a cloud storage bucket, remove mlflow-pvc.yaml and its mount paths, and change the artifact path variable MLFLOW_ARTIFACT_ROOT in sfmlops-helm/templates/global-configmap.yaml to the cloud storage path. Here's the official doc for more information.

References / Useful resources
Ray sample config: https://github.com/ray-project/kuberay/tree/master/ray-operator/config/samples
Bitnami Kafka Helm: https://github.com/bitnami/charts/tree/main/bitnami/kafka
Airflow Helm: https://airflow.apache.org/docs/helm-chart/stable/index.html
Airflow Helm default values.yaml: https://github.com/apache/airflow/blob/main/chart/values.yaml
dataset: https://www.kaggle.com/datasets/pratyushakar/rossmann-store-sales
Original Airflow's docker-compose file: https://airflow.apache.org/docs/apache-airflow/2.8.3/docker-compose.yaml
My notes
If you have any comments, questions, or want to learn more, check out notes/README.md. I have included a lot of useful notes about how and why I made certain choices during development. Mostly, they cover tool selection, design choices, and some caveats.
