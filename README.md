Mind Track Codes

Clone repo:
cd ~/
mkdir -p mindtrack-project
cd mindtrack-project
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
ls -la

Check Details:
cat package.json 2>/dev/null || echo "No package.json"
ls -la
ls dist/ 2>/dev/null || ls build/ 2>/dev/null || echo "No build folder"

Create Files:
cd ~/mindtrack-project/Brain-Tasks-App

# Dockerfile (nginx to serve pre-built dist/)
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# .dockerignore
cat > .dockerignore << 'EOF'
node_modules
.git
.env
README.md
EOF

# .gitignore
cat > .gitignore << 'EOF'
node_modules/
.env
.DS_Store
EOF

cat Dockerfile
ls -la

docker build -t brain-tasks-app:latest .
docker run -d -p 3000:80 --name brain-test brain-tasks-app:latest
docker ps

K8s files created:
# Remove old brain-test container and restart
docker rm -f brain-test 2>/dev/null
docker rm -f trend-app 2>/dev/null
docker ps -a

# Run fresh
docker run -d -p 3000:80 --name brain-test brain-tasks-app:latest
docker ps

Creating AWS ECR repo 

aws ecr create-repository \
  --repository-name brain-tasks-app \
  --region us-east-1

aws ecr describe-repositories \
  --repository-names brain-tasks-app \
  --query "repositories[0].repositoryUri" \
  --output text

aws ecr get-login-password --region us-east-1 | \ docker login --username AWS --password-stdin \ 562395968623.dkr.ecr.us-east-1.amazonaws.com


docker tag brain-tasks-app:latest \ 562395968623.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest docker push 562395968623.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest


cd ~/mindtrack-project/Brain-Tasks-App
mkdir -p k8s

ECR_URI="562395968623.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app"

cat > k8s/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: brain-tasks-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: brain-tasks-app
  template:
    metadata:
      labels:
        app: brain-tasks-app
    spec:
      containers:
      - name: brain-tasks-app
        image: $ECR_URI:latest
        ports:
        - containerPort: 80
EOF

cat > k8s/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: brain-tasks-service
spec:
  selector:
    app: brain-tasks-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
EOF

ls k8s/
cat k8s/deployment.yaml


Create buildspec.yml for CodeBuild:
cat > buildspec.yml << 'EOF'
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 562395968623.dkr.ecr.us-east-1.amazonaws.com
      - aws eks update-kubeconfig --name trend-eks --region us-east-1
  build:
    commands:
      - echo Building Docker image...
      - docker build -t brain-tasks-app .
      - docker tag brain-tasks-app:latest 562395968623.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest
  post_build:
    commands:
      - echo Pushing image to ECR...
      - docker push 562395968623.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest
      - echo Deploying to EKS...
      - kubectl apply -f k8s/deployment.yaml
      - kubectl apply -f k8s/service.yaml
      - kubectl get pods
      - kubectl get svc
EOF

cat buildspec.yml

Push to GitHub:

cd ~/mindtrack-project/Brain-Tasks-App

# Set up GitHub repo
git remote set-url origin https://vkeshwam1:YOUR_TOKEN@github.com/vkeshwam1/brain-tasks-deployment.git

# Add all files
git add .
git status
git commit -m "feat: Add Dockerfile, buildspec.yml, k8s manifests for Brain Tasks deployment"
git push -u origin main –force


Deploy to EKS now:
# Use existing EKS cluster
aws eks update-kubeconfig --name trend-eks --region us-east-1

# Give EKS permission to pull from ECR
kubectl create secret docker-registry ecr-secret \
  --docker-server=562395968623.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  2>/dev/null || echo "Secret already exists"

# Deploy
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/brain-tasks-app --timeout=300s
kubectl get pods
kubectl get svc

kubectl get pods
kubectl get svc
kubectl rollout status deployment/brain-tasks-app --timeout=300s


kubectl get svc brain-tasks-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
echo ""

# Add ECR and EKS permissions to CodeBuild role
aws iam attach-role-policy \
  --role-name codebuild-brain-tasks-app-build-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess

aws iam attach-role-policy \
  --role-name codebuild-brain-tasks-app-build-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy \
  --role-name codebuild-brain-tasks-app-build-service-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

echo "Permissions added!"

aws codebuild update-project \
  --name brain-tasks-build \
  --source '{
    "type": "GITHUB",
    "location": "https://github.com/vkeshwam1/brain-tasks-deployment",
    "buildspec": "buildspec.yml",
    "auth": {
      "type": "CODECONNECTIONS",
      "resource": "arn:aws:codeconnections:us-east-1:562395968623:connection/11b41948-d2e7-4288-adaa-4da1d8404a0f"
    }
  }' \
  --region us-east-1


Code for pipeline attach:

cat > /tmp/aws-auth-patch.yaml << 'PATCH'
data:
  mapRoles: |
    - rolearn: arn:aws:iam::562395968623:role/brain-tasks-codebuild-role
      username: codebuild
      groups:
        - system:masters
    - rolearn: arn:aws:iam::562395968623:role/trend-app-jenkins-role
      username: jenkins
      groups:
        - system:masters
    - rolearn: arn:aws:iam::562395968623:role/trend-app-eks-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    []
PATCH
