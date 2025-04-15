#!/bin/bash

# Config
REGION="eu-north-1"
ACCOUNT_ID="598871140224"
REPO_NAME="container-4"
ECR_REPO="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$REPO_NAME"
CLUSTER_NAME="MyCluster"
SERVICE_NAME="service-1"
IMAGE="$ECR_REPO:latest"
TS=$(date +%Y%m%d%H%M%S)

echo "🔧 Building image..."
docker build -t $IMAGE .

echo "📤 Pushing image to ECR..."
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
docker push $IMAGE

echo "📄 Fetching current task definition..."
TD_ARN=$(aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME --query "services[0].taskDefinition" --output text)
TD_DEF=$(aws ecs describe-task-definition --task-definition $TD_ARN)

echo "🛠️ Creating new task definition..."
NEW_DEF=$(echo $TD_DEF | jq --arg IMAGE "$IMAGE" --arg TS "$TS" '
.taskDefinition |
{
  family: "container-3-task",  # manually set
  containerDefinitions: (.containerDefinitions | map(
    .image = $IMAGE |
    .environment += [{"name":"REDEPLOY_TRIGGER","value":$TS}]
  )),
  requiresCompatibilities: .requiresCompatibilities,
  networkMode: .networkMode,
  cpu: .cpu,
  memory: .memory,
  executionRoleArn: .executionRoleArn,
  taskRoleArn: .taskRoleArn,
  volumes: (.volumes // [])
}' | jq -c)  # compact JSON

echo "📦 Registering new task definition revision..."
NEW_TD_ARN=$(aws ecs register-task-definition \
  --cli-input-json "$NEW_DEF" \
  --query "taskDefinition.taskDefinitionArn" --output text)

echo "🚀 Updating ECS service to use new task def..."
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --task-definition "$NEW_TD_ARN"

echo "✅ Deployment complete!"
