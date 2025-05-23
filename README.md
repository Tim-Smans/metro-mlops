# MLOps Architecture Setup Script
This script automates the setup of a complete MLOps architecture including Kubeflow Pipelines, Istio, MLflow, MinIO, Prometheus, and Grafana on your local Kubernetes cluster (Minikube).

![](https://i.imgur.com/EOMytoF.png)

---

## ✅ Prerequisites


Before running the script, ensure the following dependencies are installed:

    Minikube
    Kubectl
    Helm
    yq
    Conda (Recommended for virtual environments)

If you are using conda, make sure you run the setup script in an activated environment

Also make sure your Minikube cluster is running with at least 32GB RAM and at least 4 CPUs

---
## ⚙️ Configuration

Edit the provided `config.yaml` file to customize your setup.

---

## 🛠️ How to Run the Script

Step 1: Activate your Conda environment
`
conda activate your_conda_env
`

Step 2: Make the script executable
`
chmod +x setup.sh
`

Step 3: Run the script
`
./setup_mlops.sh
`

While running the script:
1. Installing Kubeflow is going to take a while. If it times out for you, change the timeout limit in the `config.yaml`
2. When it is creating the minio buckets you might need to input your `sudo password`.


The script will automatically:

- Install Istio Service Mesh.
- Create namespaces and enable Istio sidecar injection.
- Deploy Kubeflow Pipelines.
- Deploy MLflow tracking server.
- Set up MinIO buckets automatically
- Deploy and configure Prometheus and Grafana with your specified password.
- Deploy MLFlow Serving, there is a seperate section about this component.

---

## 📊 Accessing Your Components
Before accessing your components you have to get your external ip

If you are using `Minikube` make sure to run `minikube tunnel` to get access to your components.

If you are cloud based the external ip should be generated automaticly. Get it by doing:
`kubectl get svc -n istio-system 
`
The `istio-ingressgateway` should have an external ip you can use.

| Component | URL | Default Credentials |
|-----------|-----|---------------------|
| Kubeflow Pipelines UI | `http://<ingress-ip>/pipeline` | - |
| MLflow UI | `http://<ingress-ip>/mlflow` | - |
| MinIO | `http://<ingress-ip>/` | User: `minio`, Password: `minio123` |
| Grafana | `http://<ingress-ip>/grafana` | User: `admin`, Password: Configured in `config.yaml` |
| Prometheus | `http://<ingress-ip>/prometheus` | - |

---


## 🔑 Grafana

Grafana password is configured via `config.yaml`. Default username is `admin`.

If you did not change the default configuration, the password should aso be `admin` 

If this for some reason doesn't work, retrieve Grafana's autogenerated password:

`kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
`

### Enabling Prometheus in Grafana

To use prometheus in grafana you first have to add it as a data source.
Follow these steps:
- In grafana go to `Connections` -> `Prometheus`
- Under `Prometheus server URL` put: `http://istio-ingressgateway.istio-system.svc.cluster.local/prometheus`
- Click `Save & test` this should validate that it's working.

---
## 🖨️ Serving your model
Serving automaticly goes through MLFlow. If you check your pods when first installing you will see a pod called:
`mlflow-model-deployment` that will not be able to start up when first creating, this is because an MLFlow model has not been trained yet.
After training your first model and saving it in the MLFlow you have to make sure that your MLFlow model that you want to serve is saved in:
`s3//ml-models/latest`
Once you have a model here you can restart the deployment using:
`kubectl rollout restart  deployment mlflow-model-deployment -n kubeflow
`

If your model has been saved correctly it should now be served


---
## Using your served model

```
url = "http://<external-ip>/mlflow-model/invocations"
headers = {"Content-Type": "application/json", "Host": "mlflow-model.local" }
response = requests.post(url, data=payload, headers=headers)

Print response
print("Model Prediction:", response.json())
```

The most important parts when using our served model is:
- Using the right url, this should be  `http://<external-ip>/mlflow-model/invocations`
- Setting the `Host` header to `mlflow-model.local` If you don't do this, you will not be able to access your model.
- If you are using a Minikube cluster make sure to use `minikube tunnel`


---

## ⚠️ Important Notes

- Ensure your Minikube VM has at least 32 GB RAM and 4 CPUs for a smooth experience.
- It's recommended to always run this script in a clean Minikube cluster to avoid conflicts.
