Welcome!
This TUTORIAL will give you a glimpse into the simplicity of Installing and using Pinniped for K8s auth.

**What is PINNIPED**

Pinniped is the easy, secure way to login to your Kubernetes cluster.
Pinniped allows cluster administrators to easily plug in external identity providers (IDPs) into Kubernetes clusters. This is achieved via a uniform install procedure across all types and origins of Kubernetes clusters, declarative configuration via Kubernetes APIs, enterprise-grade integrations with IDPs, and distribution-specific integration strategies.
More information https://pinniped.dev

#Summary:
In this lab, we will learn to use the Pinniped Concierge with Kind clusters.
Overview of steps:
1. Install K8s cluster - we will use kind
2. Install Kubectl
3. Create kind cluster
4. Install local Authenticator
5. Create a user in the authenticator
6. Fetch the generated cert
7. Install Pinniped Conceirge
8. Configure Conceirge to authenticate using local authenticator
9. Install Pinniped cli
10. Generate kubeconfig
11. Create RBAC rules for the cluster
12. Use Kubeconfig created to access cluster with Kubectl commands

##STEP 1
**Install Kind cluster**
Run the following in the adjacent terminal

`curl --insecure -Lo ./kind  https://github.com/kubernetes-sigs/kind/releases/download/v0.11.1/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/bin/kind`{{execute}}

##STEP 2
**Install Kubectl**

<!---
`sudo apt-get update`{{execute}}

`sudo apt-get install -y apt-transport-https ca-certificates curl`{{execute}}

`sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg`{{execute}}

`echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`{{execute}}

--->
`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`{{execute}}

`sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`{{execute}}

Verify it works

`kubectl version --client`{{execute}}

##STEP 3
**Create kind cluster**
You can create a cluster with a --name <clustername> if you like.

`kind create cluster`{{execute}}

Kind cluster creation takes about **two minutes**, Breathe, Have a sip of coffee and check the status!!  

##STEP 4
**Deploy Authenticator**

This is a demo authenticator. In production, you would configure an authenticator that works with your real identity provider, and therefore would not need to deploy or configure local-user-authenticator.

`kubectl apply -f https://get.pinniped.dev/latest/install-local-user-authenticator.yaml`{{execute}}

##STEP 5
**Create a user**

Create a test user named *pinny-the-seal* in the local-user-authenticator namespace

`kubectl create secret generic pinny-the-seal \
  --namespace local-user-authenticator \
  --from-literal=groups=group1,group2 \
  --from-literal=passwordHash='$2y$10$wntWabvqI93j7zPE3yIKreuIUOk0.kQdZR7o0k8mqXxT36d0FuUPu'`{{execute}}

##STEP6
**Fetch the generated cert**
`while ! kubectl get secret local-user-authenticator-tls-serving-certificate --namespace local-user-authenticator -o jsonpath={.data.caCertificate} 1>/tmp/local-user-authenticator-ca-base64-encoded 2>/dev/null; do echo "Waiting for local-user-authenticator-tls-serving-certificate Secret to be created..."; sleep 3; done`{{execute}}

##STEP 7
**Install Pinniped Conceirge**

`kubectl apply -f https://get.pinniped.dev/v0.9.2/install-pinniped-concierge-crds.yaml`{{execute}}
`kubectl apply -f https://get.pinniped.dev/v0.9.2/install-pinniped-concierge.yaml`{{execute}}


##STEP 8
**Create a WebhookAuthenticator object**
 Configure the Pinniped Concierge to authenticate using local-user-authenticator.

`cat <<EOF | kubectl create -f -
apiVersion: authentication.concierge.pinniped.dev/v1alpha1
kind: WebhookAuthenticator
metadata:
  name: local-user-authenticator
spec:
  endpoint: https://local-user-authenticator.local-user-authenticator.svc/authenticate
  tls:
    certificateAuthorityData: $(cat /tmp/local-user-authenticator-ca-base64-encoded)
EOF`{{execute}}


##STEP 9
**Install Pinniped cli**
Run the following in the linux terminal

`curl -Lso pinniped https://get.pinniped.dev/latest/pinniped-cli-linux-amd64 \
  && chmod +x pinniped \
  && sudo mv pinniped /usr/local/bin/pinniped`{{execute}}

Great! Now verify you have the right Pinniped cli version installed

`pinniped version`{{execute}}

##STEP 10
**Generate Kubeconfig**

Generate a kubeconfig for the current cluster.  

`pinniped get kubeconfig \
  --static-token "pinny-the-seal:password123" \
  --concierge-authenticator-type webhook \
  --concierge-authenticator-name local-user-authenticator>/tmp/pinniped-kubeconfig`{{execute}}
##STEP 11
**Create RBAC Rules for the user**

`kubectl create clusterrolebinding pinny-can-read \
  --clusterrole view \
  --user pinny-the-seal`{{execute}}

##STEP 12
**Use kubectl commands with the generated kubeconfig**

`kubectl --kubeconfig /tmp/pinniped-kubeconfig \
  get pods -n pinniped-concierge`{{execute}}

**DEBUG IN CASE OF FAILURE**

**Check the secret created for the user**

`kubectl get secret pinny-the-seal --namespace local-user-authenticator -o yaml`{{execute}}

**Check get pods without Kubeconfig**

`kubectl get pods -n pinniped-concierge`{{execute}}

**Check logs for the pods**

"kubectl -f "pod name from above" -n pinniped-concierge"
