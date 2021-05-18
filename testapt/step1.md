Welcome!
This TUTORIAL will give you a glimpse into the simplicity of Installing Pinniped
#Step summary:
Here's what we will be doing in the lab:
1. Install K8s cluster - we will use kind
2. Install Kubectl
3. Create kind cluster
4. Install local Authenticator
5. Create a user in the authenticator
6. Install Pinniped Conceirge
7. Configure Conceirge to authenticate using local authenticator
8. Install Pinniped cli
9. Generate kubeconfig
10. Create RBAC rules for the cluster
11. Use Kubeconfig created to access cluster with Kubectl commands

##STEP 1
**Install Kind cluster**
Run the following in the adjacent terminal

`curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.10.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/bin/kind`{{execute}}

##STEP 2
**Install Kubectl**
Download Repo

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`{{execute}}

Install Kubectl

`sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`{{execute}}

Verify it works

`kubectl version --client`{{execute}}

##STEP 3
**Create kind cluster**
You can create a cluster with a --name <clustername> if you like.

`kind create cluster`{{execute}}

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
  --from-literal=passwordHash=$(htpasswd -nbBC 10 x password123 | sed -e "s/^x://")`{{execute}}

##STEP 6
**Install Pinniped Conceirge**

`kubectl apply -f https://get.pinniped.dev/latest/install-pinniped-concierge.yaml`{{execute}}


##STEP 7
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


##STEP 8
**Install Pinniped cli**
Run the following in the linux terminal

`curl -Lso pinniped https://get.pinniped.dev/v0.4.1/pinniped-cli-linux-amd64 \
  && chmod +x pinniped \
  && sudo mv pinniped /usr/local/bin/pinniped`{{execute}}

Great! Now verify you have the right Pinniped cli version installed

`Pinniped version`{{execute}}

##STEP 9
**Generate Kubeconfig**

Generate a kubeconfig for the current cluster. Use --static-token to include a token which should allow you to authenticate as the user that you created previously.

`pinniped get kubeconfig \
  --static-token "pinny-the-seal:password123" \
  --concierge-authenticator-type webhook \
  --concierge-authenticator-name local-user-authenticator \ > /tmp/pinniped-kubeconfig`{{execute}}

##STEP 10
**Create RBAC Rules for the user**

`kubectl create clusterrolebinding pinny-can-read \
  --clusterrole view \
  --user pinny-the-seal`{{execute}}

##STEP 12
**Use kubectl commands with the generated kubeconfig**

`kubectl --kubeconfig /tmp/pinniped-kubeconfig \
  get pods -n pinniped-concierge`{{execute}}
