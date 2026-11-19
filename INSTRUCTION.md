# ToDo Application – Kubernetes Testing Instructions

## Prerequisites

Make sure the namespace and application resources are deployed.

```bash
kubectl apply -f todoapp-pod.yml
kubectl apply -f cluster_ip.yml
kubectl apply -f node_port.yml
```

Check the status of the Pods and Services:

```bash
kubectl get pods -n todoapp
kubectl get services -n todoapp
```

Both Pods should eventually show `Running` and `Ready`.

---

## 1. Test the Application Using ClusterIP Service DNS

A ClusterIP Service is accessible only from within the Kubernetes cluster.

We can test it by creating a temporary BusyBox container.

### Step 1. Start a BusyBox container

```bash
kubectl run busybox \
  --image=busybox:1.36 \
  --namespace=todoapp \
  --restart=Never \
  -it --rm \
  -- sh
```

### Step 2. Access the ToDo application using its Service DNS

From inside the BusyBox container, execute:

```bash
wget -qO- http://todoapp-service.todoapp.svc.cluster.local/api/health
```

You can also use the shorter DNS name because BusyBox runs in the same namespace:

```bash
wget -qO- http://todoapp-service/api/health
```

Expected response:

```text
OK
```

The request is sent to the ClusterIP Service, which forwards traffic to one of the matching application Pods on port `8080`.

### Step 3. Test the readiness endpoint

```bash
wget -qO- http://todoapp-service/api/ready
```

Expected response:

```text
OK
```

Exit the BusyBox container:

```bash
exit
```

---

## 2. Test the Application Using Port Forwarding

Port forwarding allows us to access a Kubernetes Service from our local machine without exposing it externally.

### Step 1. Forward the Service port

Execute this command on your local machine:

```bash
kubectl port-forward -n todoapp service/todoapp-service 8000:80
```

Explanation:

- `8000` – local machine port.
- `80` – Service port inside Kubernetes.
- `todoapp-service` – ClusterIP Service name.

Keep this terminal running while testing.

### Step 2. Access the application

Open your browser and navigate to:

```text
http://localhost:8080/
```

You can also test the health endpoint:

```text
http://localhost:8080/api/health
```

Or use curl from another terminal:

```bash
curl http://localhost:8080/api/health
```

Expected response:

```text
OK
```

### Step 3. Stop port forwarding

Press `Ctrl+C` in the terminal running the port-forward command.

---

## 3. Access the Application Using a NodePort Service

A NodePort Service exposes the application on a specific port of each Kubernetes node.

In our configuration, the NodePort is `30007`.

### Step 1. Verify the NodePort Service

```bash
kubectl get service todoapp-nodeport-service -n todoapp
```

Expected configuration:

```text
NAME                       TYPE       CLUSTER-IP     PORT(S)
todoapp-nodeport-service   NodePort   <CLUSTER-IP>   80:30007/TCP
```

### Step 2. Find the Kubernetes Node IP

```bash
kubectl get nodes -o wide
```

Find the `INTERNAL-IP` of your Kubernetes node.

### Step 3. Access the application

Open a browser and navigate to:

```text
http://<NODE-IP>:30007/
```

For example, if your node IP is `192.168.1.100`:

```text
http://192.168.1.100:30007/
```

You can also test the health endpoint:

```bash
curl http://<NODE-IP>:30007/api/health
```

Expected response:

```text
OK
```

**Note:** The NodePort must be reachable from your machine. Make sure your network configuration and firewall allow incoming traffic on TCP port `30007`.

---

## Summary

| Method | Access mechanism | Address |
|---|---|---|
| ClusterIP | Internal cluster DNS | `http://todoapp-service/api/health` |
| Port Forward | Localhost | `http://localhost:8080/api/health` |
| NodePort | Kubernetes Node IP | `http://<NODE-IP>:30007/api/health` |

All three methods ultimately route HTTP traffic to the ToDo application Pods on container port `8000`.