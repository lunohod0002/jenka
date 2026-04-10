# Начало
Я использовал Kuber в DockerDesktop поэтому мне достаточно было просто сбилдить image, чтобы он импортировался в kuber, если вы делаете иначе то вам возможно понадобиться импортировать в ручную

Сначала создаем image
```powershell
docker build -t quarkus-app -f .\ci-cd-exmaple-app-main\JavaDockerfile ci-cd-exmaple-app-main\
```
Если вы не в dockerDesktop, то можно импортировать так (Пример для kind):
```powershell
kind load docker-image quarkus-app
```

# Шаг 1: Подготовка окружения (Helm и Ingress)
```powershell
# Через Chocolatey
choco install kubernetes-helm
# ИЛИ через Winget
winget install Helm.Helm
```

## Установка Ingress-контроллера (Nginx):
В Docker Desktop Ingress не идет в комплекте, его нужно поставить командой:
```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.3/deploy/static/provider/aws/deploy.yaml
```
# Поднять локальный registry
docker run -d --restart=always -p 5000:5000 --name registry registry:2

# Проверить
curl http://localhost:5000/v2/_catalog

# {"repositories":[]}

# Запушить старый и будущий образ
docker tag georgezhironkin/quarkus-native:v3 localhost:5000/georgezhironkin/quarkus-native:v3
docker push localhost:5000/georgezhironkin/quarkus-native:v3
docker tag localhost:5000/georgezhironkin/quarkus-native:v3 localhost:5000/georgezhironkin/quarkus-native:v4
docker push localhost:5000/georgezhironkin/quarkus-native:v4

# Проверить, что образ в registry
curl http://localhost:5000/v2/georgezhironkin/quarkus-native/tags/list
После того как вы установили Ingress, нужно развернуть само приложение и настройку для ingress
```powershell
kubectl apply -f deploy.yml
kubectl apply -f ingress.yml
```

Для проверки развернутого приложения:
```powershell
kubectl get pods
```

Настройка файла hosts:
Чтобы work.local работал, добавьте строку в C:\Windows\System32\drivers\etc\hosts:
```text
127.0.0.1 work.local
```

После перейдите по ссылке http://work.local/q/swagger-ui/. У вас должна открыться документация приложения.

# Шаг 2: Установка Prometheus Stack и Адаптера
Установка основного стека (Prometheus + Grafana):
```powershell
helm install kube-stack-prometheus oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack --version 69.3.2 --namespace monitoring --create-namespace
```

## Установка Prometheus Adapter (с вашими правилами):
Используйте ваш файл values-adapter.yaml:
```powershell
helm install prometheus-adapter-custom oci://ghcr.io/prometheus-community/charts/prometheus-adapter --version 4.11.0 --namespace monitoring -f values-adapter.yaml
```

# Шаг 3: Деплой приложения и мониторинга
Примените ваши манифесты в указанном порядке:
```powershell
# 1. Мониторинг (ServiceMonitor)
kubectl apply -f monitor.yml

# 2. Автоскейлер (HPA)
kubectl apply -f hpa-quarkus.yml
```

# Шаг 4: Проверка доступности метрик
Подождите 1-2 минуты, пока Prometheus соберет первые данные, затем проверьте Custom Metrics API:
```powershell
# Проверка регистрации метрики
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"

# Получение текущего значения (должно быть 0 или больше)
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/work_active_requests_custom"
```

Если возникли проблемы, проверьте поднелись ли поды prometheus:
```powershell
kubectl get pods --namespace monitoring
```



Наблюдение за ростом реплик:
```powershell
# Следите за колонкой Replicas
kubectl get hpa quarkus-hpa -w
```

Проверка распределения:
Убедитесь, что запросы летят на разные поды (значения в value должны быть сопоставимы):
``` powershell
(kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/work_active_requests_custom" | ConvertFrom-Json).items | Select-Object { $_.describedObject.name }, value
```

## Этап 5. Jenkins
 
### 5.1 Запустить Jenkins в Docker
 
```bash
docker volume create jenkins-data



cat ~/.kube/config > kubeconfig
 ### Docker имеет специальный DNS-адрес, который ссылается на хост-машину (host.docker.internal)Возьмите ваш kubeconfig. и замените  адрес сервера  https://127.0.0.1:52694 на https://host.docker.internal:52694 (порт оставьте тот же). Это нужно, чтобы Jenkins обращался к кластеру
 

 ## В том же kubeconfig файле, под строчкой server:, нужно добавить строчку, отключающую проверку:
'''
    insecure-skip-tls-verify: true # <--- ОБЯЗАТЕЛЬНО ДОБАВЬТЕ ЭТО
'''
## Также уберите строчку certificate-authority-data, так как в доверенный адресах для kubernetes нет host.docker.internal
### Итоговый конфиг для jenkins:
'''
...
clusters:
- cluster:
    server: https://host.docker.internal:52694
    insecure-skip-tls-verify: true # <--- ОБЯЗАТЕЛЬНО ДОБАВЬТЕ ЭТО
  name: kind-kind
...
'''

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8081:8080 -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/kubeconfig:/var/jenkins_home/.kube/config \
  jenkins/jenkins:lts
 
# Начальный пароль
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
 
Открыть `http://localhost:8081`, ввести пароль, установить плагины.
 
### 5.2 Установить инструменты внутри Jenkins
 
```bash
docker exec -it -u root jenkins bash
 
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && mv kubectl /usr/local/bin/
kubectl version --client
 # Docker CLI (для проверки образа в registry)
apt-get update && apt-get install -y docker.io curl jq
 
exit
```
 # Шаг 6 - подготовка утилиты Go
 ## Шаг 6.1-собрать образ (docker build -t loadtester .)
 Проверка запуска:
 docker run --rm --network host loadtester \
  -url http://work.local/work \
  -rps 300 \
  -duration 30s | tee test_output.txt

cat test_output.txt
# Шаг 7
## Перед запуском pipeline необходимо обновить deployment до предудющего образа (v3 в примере)
kubectl apply -f deploy.yml

### Для pipeline Jenkins выложите ваш файл с Jenkins на github и настройте pipeline в интерфейсе Jenkins через git scm
