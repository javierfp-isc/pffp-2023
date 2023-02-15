# pffp-2023

## Despliegue de gitlab

En gitlab tenemos un despliegue docker-compose para levantar un gitlab de pruebas:

```
docker-compose up -d
```

Para acceder por primera vez con root la password está en **/etc/gitlab/initial_root_password**

## Despliegue de k3d

Para simular un cluster Kubernetes en docker usaremos k3d. Para desplegarlo:

Necesamos:
* docker
* kubectl (se puede instalar con apt)

Instalación:

```
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Creación de clúster con 3 nodos

```
k3d cluster create mi-cluster --agents 3
```

Parar, destruir, iniciar podemos usar comandos del tipo **k3d cluster** (stop, start, delete...)

## Integración de Gitlab con el cluster Kubernetes

### Instalación de Helm

Necesitamos disponer de helm en el equipo de trabajo para poder instalar apps en el cluster. Para instalarlo:

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Instalación de Jupytherhub en el clúster con Helm

Para la instalación añadimos el repositorio

```
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
```

Actualizamos e instalamos el chart en el namespace **default**:

```
helm install jupyterhub/jupyterhub --generate-name  --namespace=default
```

Si no tenemos Ingress, activamos una redirección con a través de localhost:8080:

```
kubectl --namespace=default port-forward service/proxy-public 8080:http &
```

Para acceder:

* user: jovyan
* password: jupyter

### Creación del agent

Editamos en gitlab el archivo /etc/gitlab/gitlab.rb añadiendo las líneas:

```
gitlab_kas['enable'] = true
gitlab_kas['listen_address'] = '0.0.0.0:8150'                                                                        
gitlab_kas['listen_network'] = 'tcp'                         
gitlab_kas['listen_websocket'] = false
```

Ejecutamos:

```
gitlab-ctl reconfigure
```

Para la integración:

* Creamos proyecto en gitlab. Vamos a Infraestructure->Kubernetes clusters->Connect a cluster
* Creamos el archivo de configuración del agente en la raíz del repositorio en **.gitlab/agents/agent/config.yaml** (en este caso el nombre del agente es "agent")
* Creamos un agent (de nombre agent) y lo registramos (Register). Desde la ventana emergente copiamos el comando del tipo:

```
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install agent gitlab/gitlab-agent \
    --namespace default \
    --set image.tag=v15.8.0 \
    --set config.token=DhDy1tXxgxEZyeGgTQcon4ux9myji9eS32m7NxnTyfWKs_Tfwg \
    --set config.kasAddress=grpc://172.17.0.1:8150/-/kubernetes-agent
```

El comando anterior añade el repo de gitlab a Helm, lo actualiza e instala el agente.

Para ver los logs del pod del agente:

```
kubectl logs -f -l=app=gitlab-agent -n default
```
