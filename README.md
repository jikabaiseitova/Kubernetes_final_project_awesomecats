Final Project Steps
1. Create RDS or CloudSQL instance, connect to your database and create tables inside postgres database
2. Create a secret with your hostname, username, password and database name.
3. Clone this repos locally:
https://github.com/AntTechLabs/awesome_cats_backend.git
https://github.com/AntTechLabs/awesome_cats_frontend.git
4. Write Dockerfile for frontend and backend images
5. Push your images to private repos in ECR or GCR
6. Write yaml files for backend and frontend deployments with 2 replicas, show database credentials as env variable
7. Create clusterIP services for your deployments
8. Install nginx controller and create ingress to access your application with load balancer.
9. Configure your domain name with Cloud DNS or Route53 and ExternalDNS, access awesome cats application from web browser with your domain name
10. Get certificate to your domain name with cert-manager


IAM&Admin in GCR:
ADD Role: Compute Storage Admin
DNS Administrator
(json format)

Step 1)
create SCL: GCloud

cli:
gcloud sql instances list
gcloud sql connect postgres --user=postgres
gcloud sql instances describe postgres

gcloud sql instances patch postgres --authorized-networks=$(curl ifconfig.me)

C:\Program Files\PostgreSQL\15\bin

C:\Users\jikab>psql --version
psql (PostgreSQL) 15.4

psql -h 34.135.232.164 -U postgres -d postgres -p 5432
psql --host=<hostname> --username=<username> --dbname=<database_name>

Hostname 34.135.232.164
username postgres
database postgres
password 1234
port 5432

Step 2)
Secret

echo -n "your_hostname" | base64
echo -n "your_username" | base64
echo -n "your_password" | base64
echo -n "your_database_name" | base64

echo -n "34.135.232.164" | base64
MzQuMTM1LjIzMi4xNjQ=
echo -n "postgres" | base64
cG9zdGdyZXM=
echo -n "1234" | base64
MTIzNA==
echo -n "postgres" | base64
cG9zdGdyZXM=

database-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: database-secret
  namespace: default
type: Opaque
data:
  hostname: MzQuMTM1LjIzMi4xNjQ=
  username: cG9zdGdyZXM=
  password: MTIzNA==
  database: cG9zdGdyZXM=


kubectl apply -f database-secret.yaml

Step 3)
git clone https://github.com/AntTechLabs/awesome_cats_backend.git
git clone https://github.com/AntTechLabs/awesome_cats_frontend.git

Step 4)
Dockerfile backend
-->
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
-->

Dockerfile frontend
-->
FROM node:14 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
-->

Step 5)
In GCR: Name - ID
New Project 	new-project-426217


cd ..
cd awesome_cats_backend 
docker build -t gcr.io/new-project-426217/backend:latest .
docker push gcr.io/new-project-426217/backend:latest

cd ..
cd awesome_cats_frontend
docker build -t gcr.io/new-project-426217/frontend:latest .
docker push gcr.io/new-project-426217/frontend:latest

Step 6)
Backend-deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: gcr.io/new-project-426217/backend:latest
        ports:
        - containerPort: 3000
        env:
        - name: PGHOST
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: hostname
        - name: PGUSER
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: username
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: password
        - name: PGDATABASE
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: database

Frontend-deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: gcr.io/new-project-426217/frontend:latest
        ports:
        - containerPort: 80


kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

Step 7)
Backend-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000

Frontend-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80


kubectl apply -f backend-service.yaml
kubectl apply -f frontend-service.yaml
Kubectl get pods
kubectl get svc

Step 8)
Ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awesome-cats-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    external-dns.alpha.kubernetes.io/hostname: "awesome.zhyldyz.site"
spec:
  tls:
  - hosts: 
    - awesome.zhyldyz.site
    secretName: tls-secret
  ingressClassName: nginx
  rules:
  - host: awesome.zhyldyz.site
    http:
      paths:
      - path: /api/?(.*)
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 3000
      - path: /?(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

kubectl apply -f ingress.yaml
kubectl get svc
kubectl get svc -n ingress-nginx  (External IP)

Проверка backenda:
<external ip kubectl get svc>:<port>
34.118.236.178:3000
34.148.126.148:80    ( frontend )
