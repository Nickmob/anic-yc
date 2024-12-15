# anic-yc
Запуск ANIC в качестве Ingress для Яндекс.Облака

## Установка локальных инструментов для kubernetes и Яндекс.Облака

Пример для Ubuntu 24/04.

```bash
sudo snap install --classic helm
sudo snap install --classic kubectl
sudo snap install docker
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
yc init
```
Токен для доступа в Яндекс.Облако можно получить здесь:  https://oauth.yandex.ru/verification_code

После создания кластера добавляем в конфиг параметры доступа, они будут храниться в **~/.kube/config**. Здесь название кластера "оtus"

```bash
yc managed-kubernetes cluster get-credentials otus --external
```
Посмотрим ноды и конфигурацию кластера

```bash
kubectl config view
kubectl get nodes
kubectl get svc
```

## Получаем доступ к образу ANIC

Создаём docker-секрет. Сохраняем путь к файлу с секретом для следующей команды.

```bash
docker login -u=<login> -p=<password> anic.docker.angie.software
```
Создаём секрет для kubernetes из секрета docker

```
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=$HOME/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
```
## Получаем и готовим helm-чарты

Получем helm-чарт из репозитоирия

```bash
git clone https://git.angie.software/web-server/anic-helm-charts.git
cd anic-helm-charts/anic/

```

Редактируем values.yaml. Ставим:

```yaml
...
imagePullSecretName: "regcred"
...
enableCertManager: true
...

```
Собираем чарт и устанавливаем.

```
helm package .
helm install anic anic-0.5.0.tgz
```

## Запускаем менеджер сертификатов


```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.1/cert-manager.yaml
```

Должно быть три пода с готовностью 1/1 и Running

```bash
kubectl get pods -n cert-manager --watch
kubectl apply -f http01-clusterissuer.yaml
```

Создаём объект ClusterIssuer для выпуска сертификатов:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: http01-clusterissuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: test@test.ru
    privateKeySecretRef:
      name: http01-clusterissuer-secret
    solvers:
    - http01:
        ingress:
          class: angie
```

```bash
kubectl apply -f http01-clusterissuer.yaml
```

## Запускаем Ingress и тестовое приложение

Нужно прописать действующий домен, поставить его вместо otus-kube.mtdlb.ru. Внешний IP находим в панели Облака: Managed Service for Kubernetes/Кластеры/otus/Сеть (внешний IP-адрес). 

```bash
kubectl apply -f http01-clusterissuer.yaml
```

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    cert-manager.io/cluster-issuer: "http01-clusterissuer"
    acme.cert-manager.io/http01-edit-in-place: "true"
spec:
  ingressClassName: angie
  tls:
    - hosts:
      - otus-kube.mtdlb.ru
      secretName: domain-name-secret
  rules:
    - host: otus-kube.mtdlb.ru
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: app
              port:
                number: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
```

Запускаем всё сразу:
```bash
kubectl apply -f app-angie.yaml
```

Проверяем статус сертификата и сектера (в котором он содержится)

```bash
kubectl describe certificate domain-name-secret
kubectl describe secret domain-name-secret
```

## Проверка работы

Заходим по https://otus-kube.mtdlb.ru

Если есть проблемы, смотрим логи подов.
Просмотр конфигурации Angie (подставляем в angic-.. имя пода с ANIC):

```bash
kubectl exec anic-64ff957589-jlg7s -- angie -T

```

## Удаление

```bash
kubectl delete -f app-angie.yaml
```
