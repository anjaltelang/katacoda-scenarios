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
  --from-literal=passwordHash=$2y$10$wntWabvqI93j7zPE3yIKreuIUOk0.kQdZR7o0k8mqXxT36d0FuUPu`{{execute}}

##STEP6
**Fetch the generated cert**
`kubectl get secret local-user-authenticator-tls-serving-certificate --namespace local-user-authenticator \
  -o jsonpath={.data.caCertificate} \
  | tee /tmp/local-user-authenticator-ca-base64-encoded`{{execute}}

**Alternatively try**
`while ! kubectl get secret local-user-authenticator-tls-serving-certificate --namespace local-user-authenticator -o jsonpath={.data.caCertificate} 1>/tmp/local-user-authenticator-ca-base64-encoded 2>/dev/null; do echo "Waiting for local-user-authenticator-tls-serving-certificate Secret to be created..."; sleep 3; done`{{execute}}

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
**Install Pinniped cli**
Run the following in the linux terminal

`curl -Lso pinniped https://get.pinniped.dev/latest/pinniped-cli-linux-amd64 \
  && chmod +x pinniped \
  && sudo mv pinniped /usr/local/bin/pinniped`{{execute}}

Great! Now verify you have the right Pinniped cli version installed

`pinniped version`{{execute}}

##STEP 10
**Generate Kubeconfig**

Generate a kubeconfig for the current cluster. Use --static-token to include a token which should allow you to authenticate as the user that you created previously.

**Copy paste the text in quotes below into the terminal**

**Note**: I have broken the command down into two parts but it is a **Single** command. Copy the first part and add the second part with a space, then hit enter.  

`pinniped get kubeconfig \
  --static-token "pinny-the-seal:password123" \
  --concierge-authenticator-type webhook \
  --concierge-authenticator-name local-user-authenticator \`

  `> /tmp/pinniped-kubeconfig`


##STEP 11
**Create RBAC Rules for the user**

`kubectl create clusterrolebinding pinny-can-read \
  --clusterrole view \
  --user pinny-the-seal`{{execute}}

##STEP 12
**Use kubectl commands with the generated kubeconfig**

`kubectl --kubeconfig /tmp/pinniped-kubeconfig \
  get pods -n pinniped-concierge`{{execute}}

**DEBUG**

**Check get pods without Kubeconfig**

`kubectl get pods -n pinniped-concierge`{{execute}}

**Check logs for the pods**

"kubectl -f "pod name from above" -n pinniped-concierge"
