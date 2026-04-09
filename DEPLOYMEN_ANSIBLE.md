# Ansible Deployment Automation for Kubernetes Application
Here is an Ansible playbook to automate the deployment process for your application, including cloning from Git, building and distributing Docker images, and deploying to Kubernetes.

```yaml
---
- name: Deploy Login Application to Kubernetes
  hosts: localhost
  become: false
  vars:
    git_repo: https://github.com/Widhi-yahya/kubernetes_installation_docker.git
    git_branch: main
    app_dir: "{{ playbook_dir }}"
    worker_nodes:
      - name: k8s-control
        user: widhi
        ip: 10.34.7.5
    master_node_ip: 10.34.7.115

  tasks:
    - name: Create app directory if it doesn't exist
      file:
        path: "{{ app_dir }}"
        state: directory

    - name: Clone git repository
      git:
        repo: "{{ git_repo }}"
        dest: "{{ app_dir }}"
        version: "{{ git_branch }}"
        clone: yes
        update: yes

    - name: Create data directory on worker nodes
      delegate_to: "{{ item.ip }}"
      become: true
      file:
        path: /mnt/data
        state: directory
        mode: '0777'
      with_items: "{{ worker_nodes }}"

    - name: Build Docker image
      shell: |
        cd {{ app_dir }}/app
        docker build -t login-app:latest .
      args:
        executable: /bin/bash

    - name: Save Docker image
      shell: docker save login-app:latest > {{ app_dir }}/app/login-app.tar
      args:
        executable: /bin/bash

    - name: Copy Docker image to worker nodes
      copy:
        src: "{{ app_dir }}/app/login-app.tar"
        dest: "/tmp/login-app.tar"
      delegate_to: "{{ item.ip }}"
      with_items: "{{ worker_nodes }}"

    - name: Load Docker image on worker nodes
      delegate_to: "{{ item.ip }}"
      shell: docker load < /tmp/login-app.tar
      with_items: "{{ worker_nodes }}"

    # Adding server identification code for load balancer 
    - name: Create server-patch.js file
      copy:
        content: |
          const os = require('os');
          const serverInfo = {
            hostname: os.hostname(),
            podName: process.env.POD_NAME || 'unknown',
            nodeName: process.env.NODE_NAME || 'unknown'
          };

          // Add this after the health route
          app.get('/server-info', (req, res) => {
            res.json(serverInfo);
          });

          // Insert this line before app.listen
          app.use((req, res, next) => {
            res.setHeader('X-Served-By', serverInfo.podName);
            next();
          });
        dest: "{{ app_dir }}/app/server-patch.js"

    - name: Apply server patch for load balancer
      shell: |
        cd {{ app_dir }}/app
        cat server-patch.js >> server.js
        docker build -t login-app:latest .
        docker save login-app:latest > login-app.tar
      args:
        executable: /bin/bash

    - name: Copy updated Docker image to worker nodes
      copy:
        src: "{{ app_dir }}/app/login-app.tar"
        dest: "/tmp/login-app.tar"
      delegate_to: "{{ item.ip }}"
      with_items: "{{ worker_nodes }}"

    - name: Load updated Docker image on worker nodes
      delegate_to: "{{ item.ip }}"
      shell: docker load < /tmp/login-app.tar
      with_items: "{{ worker_nodes }}"

    - name: Clean up old deployments 
      shell: |
        kubectl delete deployment login-app mysql --ignore-not-found
        kubectl delete service login-app mysql --ignore-not-found
        kubectl delete pvc mysql-pvc --ignore-not-found
        kubectl delete pv mysql-pv --ignore-not-found
        kubectl delete secret mysql-secret --ignore-not-found
      args:
        executable: /bin/bash
      ignore_errors: true

    - name: Deploy MySQL
      shell: |
        cd {{ app_dir }}
        kubectl apply -f k8s/mysql-secret.yaml
        kubectl apply -f k8s/mysql-pv.yaml
        kubectl apply -f k8s/mysql-pvc.yaml
        kubectl apply -f k8s/mysql-deployment.yaml
        kubectl apply -f k8s/mysql-service.yaml
      args:
        executable: /bin/bash

    - name: Wait for MySQL to be ready
      shell: |
        kubectl wait --for=condition=ready pod -l app=mysql --timeout=180s
      args:
        executable: /bin/bash
      ignore_errors: true

    - name: Deploy Standard Web Application
      shell: |
        cd {{ app_dir }}
        kubectl apply -f k8s/web-deployment.yaml
        kubectl apply -f k8s/web-service.yaml
      args:
        executable: /bin/bash

    - name: Deploy Load Balancer Configuration
      shell: |
        cd {{ app_dir }}
        kubectl apply -f k8s/web-deployment-lb.yaml
        kubectl create namespace ingress-nginx --dry-run=client -o yaml | kubectl apply -f -
        
        # Add Helm repo if not already added
        helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx || true
        helm repo update
        
        # Get worker node name
        WORKER_NODE=$(kubectl get nodes | grep -v master | grep -v control | head -1 | awk '{print $1}')
        
        # Install Nginx Ingress Controller
        helm install ingress-nginx ingress-nginx/ingress-nginx \
          --namespace ingress-nginx \
          --set controller.nodeSelector."kubernetes\\.io/hostname"=$WORKER_NODE \
          --set controller.service.type=NodePort \
          --set controller.service.nodePorts.http=30081
        
        kubectl apply -f k8s/login-app-ingress.yaml
        kubectl apply -f k8s/web-service-lb.yaml
      args:
        executable: /bin/bash
      ignore_errors: true

    - name: Configure Calico networking for correct IP detection
      shell: |
        # Fix Calico IP detection to use the correct network interface
        kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=can-reach={{ master_node_ip }}
        
        # Restart calico-node pods to apply changes
        kubectl delete pod -n calico-system -l k8s-app=calico-node
        
        # Wait for calico-node pods to be ready
        kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=180s
      args:
        executable: /bin/bash
      ignore_errors: true

    - name: Restart login-app deployment for DNS fix
      shell: |
        # Restart deployment to ensure proper DNS resolution
        kubectl rollout restart deployment login-app
        
        # Wait for rollout to complete
        kubectl rollout status deployment login-app --timeout=180s
      args:
        executable: /bin/bash

    - name: Display access information
      debug:
        msg: |
          Deployment completed!
          
          Standard application access:
          http://{{ master_node_ip }}:30080
          
          Load balanced application access:
          http://{{ master_node_ip }}:30081
          
          Login credentials:
          Username: admin
          Password: admin123

    - name: Test load balancer
      shell: |
        for i in {1..5}; do 
          curl -s http://{{ master_node_ip }}:30081/server-info
          echo ""
          sleep 1
        done
      register: lb_test
      ignore_errors: true

    - name: Display load balancer test results
      debug:
        var: lb_test.stdout_lines
      when: lb_test is defined
```

## Instructions for Using the Ansible Playbook

1. Save the above playbook as `ansible-deploy.yml`

2. The variables are already set for your current cluster:
   - `git_repo`: https://github.com/Widhi-yahya/kubernetes_installation_docker.git
   - `worker_nodes`: k8s-control (10.34.7.5) with user widhi
   - `master_node_ip`: 10.34.7.115

3. Install required Ansible modules:
   ```bash
   ansible-galaxy collection install kubernetes.core
   ansible-galaxy collection install community.general
   ```

4. Make sure you have Ansible installed:
   ```bash
   pip install ansible
   ```

5. Set up SSH key-based authentication to worker node:
   ```bash
   ssh-copy-id widhi@10.34.7.5
   ```

6. Run the playbook:
   ```bash
   ansible-playbook ansible-deploy.yml
   ```

## Prerequisites

1. Ansible installed on the control machine
2. SSH access to worker nodes with passwordless authentication (SSH keys)
3. kubectl configured on the control plane (kube-master)
4. Helm installed for the load balancer deployment
5. Docker installed on all machines
6. Calico CNI installed and configured

## Customizing the Deployment

1. If you have more worker nodes, add them to the `worker_nodes` list
2. If your Git repository requires authentication, add credentials to the Git task
3. Modify the Docker build process if you need additional steps
4. Adjust the kubernetes wait timeouts if needed

## Access Information

After successful deployment:

### Login Application
- **Standard access**: http://10.34.7.115:30080 or http://10.34.7.5:30080
- **Load balanced access**: http://10.34.7.115:30081 or http://10.34.7.5:30081
- **Credentials**: 
  - Username: `admin`
  - Password: `admin123`

### Kubernetes Dashboard
- **URL**: https://10.34.7.115:30119 or https://10.34.7.5:30119
- **Access Token**: Generate with:
  ```bash
  kubectl create token widhi -n kube-system --duration=24h
  ```

## Troubleshooting

### Check deployment status:
```bash
# Check all pods
kubectl get pods

# Check login-app logs
kubectl logs -l app=login-app

# Check MySQL logs
kubectl logs -l app=mysql

# Check Calico networking
kubectl get pods -n calico-system
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.projectcalico\.org/IPv4Address}{"\n"}{end}'
```

### If database connection fails:
```bash
# Restart login-app deployment
kubectl rollout restart deployment login-app

# Check DNS resolution
kubectl exec -it $(kubectl get pods -l app=login-app -o name | head -1) -- nslookup mysql
```

### If networking issues persist:
```bash
# Verify Calico IP detection
kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=can-reach=10.34.7.115
kubectl delete pod -n calico-system -l k8s-app=calico-node
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=180s
```

This Ansible playbook automates the entire deployment process, including building and distributing Docker images, setting up the database, configuring networking, and setting up the load balancer.