
…or create a new repository on the command line

```
echo "# k8-code-repo" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:shijithpkd/k8-code-repo.git
git push -u origin main
```
…or push an existing repository from the command line


git remote add origin git@github.com:shijithpkd/k8-code-repo.git
git branch -M main
git push -u origin main



cluster configuration 

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 139.59.87.151:6443 --token esw1fx.2uuxl99exzf3e2bz \
        --discovery-token-ca-cert-hash sha256:412ded1dfaba06344d564d6d3dd2e98c56a3b4560d877367ed563ada7d9a7f30



