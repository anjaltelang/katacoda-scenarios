Welcome!
This TUTORIAL will give you a glimpse into the simplicity of Installing Pinniped
#Step summary:
Here's what we will be doing in the lab:
1. Install K8s cluster - we will use kind
2. Install Docker
3. Install Kubectl
4. Create kind cluster
5. Install local Authenticator
6. Create a user in the authenticator
7. Install Pinniped Conceirge
8. Configure Conceirge to authenticate using local authenticator
9. Install Pinniped cli
10. Create RBAC rules for the cluster
11. Use Kubeconfig created to access cluster with Kubectl commands

##STEP 1
**Install Kind cluster**
Run the following in the adjacent terminal

`curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.10.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind`{{execute}}

##STEP 2
**Install Docker**
Run the following command to install docker

`sudo apt-get install docker-ce docker-ce-cli containerd.io`{{execute}}

##STEP 3
**Install Kubectl**
Download Repo

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`{{execute}}

Install Kubectl

`sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`{{execute}}

Verify it works

`kubectl version --client`{{execute}}

##STEP 4
**Create kind cluster**
You can create a cluster with a --name <clustername> if you like.

`kind create cluster`{{execute}}

##STEP 5
**Deploy Authenticator**
This is a demo authenticator. In production, you would configure an authenticator that works with your real identity provider, and therefore would not need to deploy or configure local-user-authenticator.

`kubectl apply -f https://get.pinniped.dev/latest/install-local-user-authenticator.yaml`{{execute}}

##STEP 6
**Create a user**
Create a test user named *pinny-the-seal* in the local-user-authenticator namespace

`kubectl create secret generic pinny-the-seal \
  --namespace local-user-authenticator \
  --from-literal=groups=group1,group2 \
  --from-literal=passwordHash=$(htpasswd -nbBC 10 x password123 | sed -e "s/^x://")`{{execute}}

##STEP 7
**Install Pinniped Conceirge**

`kubectl apply -f https://get.pinniped.dev/latest/install-pinniped-concierge.yaml`{{execute}}


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
**Install Pinniped cli **
Run the following in the linux terminal

`curl -Lso pinniped https://get.pinniped.dev/v0.4.1/pinniped-cli-linux-amd64 \
  && chmod +x pinniped \
  && sudo mv pinniped /usr/local/bin/pinniped`{{execute}}

Great! Now verify you have the right Pinniped cli version installed

`Pinniped version`{{execute}}

##STEP 10
**Generate Kubeconfig**

Generate a kubeconfig for the current cluster. Use --static-token to include a token which should allow you to authenticate as the user that you created previously.

`pinniped get kubeconfig \
  --static-token "pinny-the-seal:password123" \
  --concierge-authenticator-type webhook \
  --concierge-authenticator-name local-user-authenticator \ > /tmp/pinniped-kubeconfig`{{execute}}

##STEP 11
**Create RBAC Rules for the user**
`kubectl create clusterrolebinding pinny-can-read \
  --clusterrole view \
  --user pinny-the-seal`{{execute}}

##STEP 12
**Use kubectl commands with the generated kubeconfig**
`kubectl create clusterrolebinding pinny-can-read \
  --clusterrole view \
  --user pinny-the-seal`{{execute}}
