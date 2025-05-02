# Zero-Trust-Authentication
A security model that assumes no entity, inside or outside the network, can be inherently trusted
Deliverables for Cloud Computing projects are: 
1. Code files.
2. Max 2 page documentation on how you did it (with code explaination). 
3. A 2 minute demonstration video.
Implementing Zero-Trust Authentication for Containerized Applications
Goal: Design a Zero-Trust architecture where each request to a containerized application is
authenticated and authorized, ensuring continuous identity verification.
Architecture:
1. Service Mesh - Use Istio or Linkerd to enforce Zero-Trust principles.
2. Identity Verification - Authenticate requests via OIDC (OpenID Connect).
3. Authorization Policy - Apply fine-grained access control using OPA (Open Policy
Agent).
Workflow:
1. Deploy Istio for mutual TLS (mTLS) between services.
2. Integrate OIDC (e.g., with Keycloak) for external identity verification.
3. Implement OPA policies to control who can access specific containers.
Tools: Istio, Kubernetes, Keycloak (OIDC), OPA (Open Policy Agent).
[User] --> [Keycloak (OIDC)] --> gets JWT  --> [Istio IngressGateway] --mTLS--> [App Pod]                                 --> [OPA Policy] --> Decision
Software Requirements
1. Kubernetes
•	Container orchestration platform
•	Install via Minikube (local)
2. Istio (Service Mesh)
•	Enables mutual TLS (mTLS), traffic control, telemetry, policy enforcement
•	Install via istioctl
3. Keycloak
•	Identity Provider (IdP) supporting OIDC
•	Used for external user authentication
4. OPA (Open Policy Agent) + Envoy Filter (or OPA-Envoy Plugin)
•	Used for defining and enforcing access control policies
5. kubectl
•	Command-line tool for managing Kubernetes
6. Helm
•	Package manager for Kubernetes
7. Docker
•	For containerizing applications
8. Optional UI Tools
•	Kiali (Istio visualization)
•	Jaeger (Tracing)
•	Prometheus & Grafana (Monitoring)

What is a "Containerized Application"?
A containerized app is just a regular app (e.g., Flask, Node.js, Spring Boot) that you package inside a Docker container and deploy on Kubernetes.
For our Zero-Trust project, this app will act as the target of secure access (i.e., what users/services are trying to access).
 
Implementing Zero-Trust Authentication for Containerized Applications
1.	Install Istio on Kubernetes Cluster.	Since Istio will act as your Service Mesh (enabling mTLS between services and policy enforcement), we first install and set up Istio properly.

(a)	Install Istio CLI.	Follow the steps from https://istio.io/latest/docs/setup/getting-started/

Download the latest Istio release. Extract it in a folder in your PC. For eg I had created a ZeroAuth folder in my D: to store all Project data
D:\ZeroTrustAuth\istio-1.25.2-win-amd64\istio-1.25.2>bin\istioctl install -f samples\bookinfo\demo-profile-no-gateways.yaml -y
        |\
        | \
        |  \
        |   \
      /||    \
     / ||     \
    /  ||      \
   /   ||       \
  /    ||        \
 /     ||         \
/______||__________\
____________________
  \__       _____/
     \_____/

✔ Istio core installed ⛵️
✔ Istiod installed 🧠
✔ Installation complete

(b)	Download and install git for windows https://git-scm.com/download/win

(c)	Update Environment Variables Path to include path of bin folder of Git

(d)	Install the Gateway API CRDs
kubectl apply -k https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.2.1

(e)	 Deploy the Bookinfo sample application 
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
(f)	Check Kubernetes svc and pods running

kubectl get services
kubectl get pods
(g)	Port-forward the productpage service (direct access)
kubectl port-forward svc/productpage 9080:9080
(h)	Now open your browser to http://localhost:9080/productpage (without mtls)
(i)	Create a mtls-strict.yaml file inside samples/bookinfo/security folder
(j)	Create a Kubernetes Gateway for the Bookinfo application
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
(k)	Change the service type to ClusterIP by annotating the gateway:
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default
(l)	To check the status of the gateway, run:
kubectl get gateway
(m)	To access the gateway, you need to use the kubectl port-forward command:
kubectl port-forward svc/bookinfo-gateway-istio 8080:80

2.	Enable OIDC (JWT) Authentication on Ingress Gateway	Our goal is to configure Keycloak (our OIDC provider) so that when a user requests http://localhost:8080/productpage, → Istio Ingress Gateway should check the JWT before forwarding to the app.

(a)	Download Helm for Windows.	Download the latest Windows .zip file from https://github.com/helm/helm/releases Extract the ZIP file.
(b)	Add folder containing helm.exe to your system PATH
(c)	Download and install keycloak
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install keycloak bitnami/keycloak -n keycloak --create-namespace

(d)	Open Keycloak.	(Quickstart) Run Keycloak in Docker
docker run -d --name keycloak -p 8081:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:22.0.5 start-dev

(e)	Configure Keycloak.		Manually fetch a token using curl

(i)	Open Keycloak admin panel → Realms → master
(ii)	Go to Clients → Create Client
(iii)	Client ID: bookinfo-client
(iv)	Root & Home URL : http://localhost:9080/productpage
(v)	Valid redirect URIs : http://localhost:9080/*
(vi)	Web origins : http://localhost:9080
(vii)	Save the client.
(viii)	Create a user in Keycloak with Username: testuser
(ix)	After creating the user, go to Credentials tab. Set Password as testpassword 
(x)	Turn OFF "Temporary"
(xi)	Now, get the Access Token using curl:
curl -X POST "http://localhost:8081/realms/master/protocol/openid-connect/token" ^
  -H "Content-Type: application/x-www-form-urlencoded" ^
  -d "grant_type=password" ^
  -d "client_id=bookinfo-client" ^
  -d "username=testuser" ^
  -d "password=testpassword"
(xii)	Copy the "access_token" — this is the JWT token to be send to Istio.
(xiii)	curl -H "Authorization: Bearer YOUR_ACCESS_TOKEN_HERE" http://localhost:8080/productpage If token is correct — it will return the page
(f)	Create Istio RequestAuthentication	We have created a folder security inside bookinfo with two files as follows:-

(i)	authz-policy.yaml	This tells Istio IngressGateway to Accept JWT issued by Keycloak (issuer), Validate using public keys (jwksUri).
(ii)	request-auth.yaml	This says: Allow only if request has a valid JWT (requestPrincipal is not empty).

(g)	Since we have installed Istio with demo profile which disabled default gateways, it didn't create an ingress gateway automatically. Hence we create a custom-ingressgateway folder inside bookinfo with two files as follows:-

(i)	ingressgateway-deployment.yaml		Deploys the ingressgateway pod that accepts external traffic.
(ii)	ingressgateway-service.yaml 	Rules for routing traffic inside Kubernetes through the Gateway.

(h)	Apply all the above files with path to bookinfo folder

kubectl apply -f custom-ingressgateway/ingressgateway-deployment.yaml
kubectl apply -f custom-ingressgateway/ingressgateway-service.yaml
kubectl apply -f security/authz-policy.yaml
kubectl apply -f security/request-auth.yaml
Steps to Activate Bookinfo Istio Keycloak post Restart
minikube start --memory=6144 --cpus=4 --driver=docker (if container does not exist) else minikube start
.\bin\istioctl.exe install --set profile=demo -y
kubectl label namespace default istio-injection=enabled (creation of 2 containers : 1 app & 1 istio proxy)
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get pods
kubectl get svc
kubectl port-forward svc/productpage 9080:9080
http://localhost:9080/productpage (direct access)
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get gateway
kubectl apply -f samples/bookinfo/security/mtls-strict.yaml
kubectl get svc istio-ingressgateway -n istio-system
http://127.0.0.1/productpage (access through istio mtls)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install keycloak bitnami/keycloak -n keycloak --create-namespace
kubectl get pod -n keycloak
kubectl get svc -n keycloak
kubectl port-forward svc/keycloak 8080:80 -n keycloak
http://localhost:8080/ (Sign in to keycloak page)
Make client, user if not pre-existing
Generate JWT



