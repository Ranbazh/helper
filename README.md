# helper

*********************
minikube...
*********************

minikube start -p keptn
minikube dashboard -p keptn
minikube tunnel -p keptn


*********************
    keptn
*********************

curl -sL https://get.keptn.sh | KEPTN_VERSION=0.11.4 bash

helm install keptn keptn --version 0.18.1 -n keptn --repo=https://charts.keptn.sh --create-namespace --wait --set=continuousDelivery.enabled=true,apiGatewayNginx.type=LoadBalancer

helm install jmeter-service https://github.com/keptn/keptn/releases/download/0.18.1/jmeter-service-0.18.1.tgz -n keptn --create-namespace --wait


*********************
ISTIO 
*********************

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.12.1 sh -

./istio-1.12.1/bin/istioctl install

Config

curl -o configure-istio.sh https://raw.githubusercontent.com/keptn/examples/0.11.0/istio-configuration/configure-istio.sh

chmod +x configure-istio.sh

*********************
    KEPTN SET 
*********************

KEPTN_ENDPOINT=http://$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')/api
KEPTN_API_TOKEN=$(kubectl get secret keptn-api-token -n keptn -ojsonpath='{.data.keptn-api-token}' | base64 --decode)
KEPTN_BRIDGE_URL=http://$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')/bridge

keptn auth --endpoint=$KEPTN_ENDPOINT --api-token=$KEPTN_API_TOKEN

***************************
    KEPTN CREATE PROJECT
***************************

keptn create project sockshop --shipyard=./shipyard.yaml --git-user=ranbazh --git-token=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 --git-remote-url=https://github.com/Ranbazh/sockshop.git

kubectl get secret -n keptn bridge-credentials -o jsonpath="{.data.BASIC_AUTH_USERNAME}" | base64 –decode

kubectl get secret -n keptn bridge-credentials -o jsonpath="{.data.BASIC_AUTH_PASSWORD}" | base64 --decode

***************************
    KEPTN CREATE SERVICES
***************************

keptn create service carts --project=sockshop
keptn add-resource --project=sockshop --service=carts --all-stages --resource=./carts.tgz --resourceUri=helm/carts.tgz

keptn create service carts-db --project=sockshop
keptn add-resource --project=sockshop --service=carts-db --all-stages --resource=./carts-db.tgz --resourceUri=helm/carts-db.tgz

***************************
CREATE FUNCITON AND LOAD VALIDATION
***************************

Keptn add-resource --project=sockshop --stage=dev --service=carts --resource=jmeter/basiccheck.jmx --resourceUri=jmeter/basiccheck.jmx
keptn add-resource --project=sockshop --stage=staging --service=carts --resource=jmeter/load.jmx --resourceUri=jmeter/load.jmx

***************************
TRIGGER DEPLOYMENT LOCAL DOCKER
***************************

keptn trigger delivery --project=sockshop --service=carts-db --image=http://localhost:5000/mongo:4.2.2 --sequence=delivery-direct
curl -X GET http://192.168.1.254:5000/v2/_catalog

keptn trigger delivery --project=sockshop --service=carts-db --image=192.168.1.254:5000/mongo:4.2.2 --sequence=delivery-direct
keptn trigger delivery --project=sockshop --service=carts --image=keptnexamples/carts:0.13.1

0.13.2 - slow version

***************************
PROMETHEUS MONITORING
***************************

kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --namespace monitoring --wait

Histogram_quantile(0.9, sum by(le) (rate(http_response_time_milliseconds_bucket{job="carts-sockshop-production-primary"}[3m])))

***************************
PROMETHEUS SERVICE
***************************
0.8.2
helm install -n keptn prometheus-service https://github.com/keptn-contrib/prometheus-service/releases/download/0.7.2/prometheus-service-0.7.2.tgz --wait

kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/prometheus-service/0.7.2/deploy/role.yaml -n monitoring

kubectl -n keptn get events

kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/prometheus-service/0.7.2/deploy/role.yaml -n monitoring

kubectl port-forward svc/prometheus-server 8080:80 -n monitoring

keptn configure monitoring prometheus --project=sockshop --service=carts

***************************
CART SERVICE
***************************
echo http://carts.sockshop-dev.$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')
echo http://carts.sockshop-staging.$(kubectl -n keptn get ingress api-keptn-ingress -ojsonpath='{.spec.rules[0].host}')
echo http://carts.sockshop-production.$(kubectl -n keptn get ingress api-keptn-ingress

***************************
DOCKER 
***************************

 docker run -d -p 5000:5000 --restart=always --name localregistry registry:latest
 docker rm $(docker ps --filter status=exited -q)
 docker run -d -p 5000:5000 --restart=always --name registry -v $(pwd)/docker-registry:/var/lib/registry registry:latest

***************************
LOAD GEN namespace
***************************
kubectl apply -f deploy/cartsloadgen-base.yaml
kubectl apply -f cartsloadgen-faulty.yaml


********************************
ADD SLI & SLO 
********************************

keptn add-resource --project=sockshop --stage=staging --service=carts --resource=sli-config-prometheus-bg.yaml --resourceUri=prometheus/sli.yaml 
keptn add-resource --project=sockshop --stage=staging --service=carts --resource=slo-quality-gates.yaml --resourceUri=slo.yaml


*******************************
SELF HEALING 
******************************

keptn add-resource --project=sockshop --stage=production --service=carts --resource=sli-config-prometheus-selfhealing.yaml --resourceUri=prometheus/sli.yaml 
keptn add-resource --project=sockshop --stage=production --service=carts --resource=slo-self-healing-prometheus.yaml --resourceUri=slo.yaml
keptn add-resource --project=sockshop --stage=production --service=carts --resource=remediation.yaml --resourceUri=remediation.yaml

echo -e "{\"type\": \"sh.keptn.event.production.remediation.triggered\",\"specversion\":\"1.0\",\"source\":\"https:\/\/github.com\/keptn\/keptn\/prometheus-service\",\"id\": \"f2b878d3-03c0-4e8f-bc3f-454bc1b3d79d\",  \"time\": \"2019-06-07T07:02:15.64489Z\",  \"contenttype\": \"application\/json\", \"data\": {\"project\": \"sockshop\",\"stage\": \"production\",\"service\": \"carts\",\"problem\": { \"problemTitle\": \"response_time_p90\",\"rootCause\": \"Response time degradation\"}}}" > remediation_trigger.json | keptn send event -f remediation_trigger.json
