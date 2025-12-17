
# [`owntube/peertube-runner`](https://github.com/OwnTube-tv/peertube-runner)

Containerized Node [`@peertube/peertube-runner`](https://www.npmjs.com/package/@peertube/peertube-runner) for
remote execution of transcoding jobs in Kubernetes.

## Container Image Variants

### Image Variant 1: `owntube/peertube-runner:v632` from PeerTube v6.3.2 with Runner v0.0.21

Build the container image:

```bash
docker buildx build --platform linux/amd64 -f Dockerfile.v632 -t owntube/peertube-runner:v632 .
```

Test running the PeerTube runner server:

```bash
docker run -it --rm -u 999 --name v632-runner-server \
  -v $PWD/dot-local:/home/peertube/.local/share/peertube-runner-nodejs \
  -v $PWD/dot-config:/home/peertube/.config/peertube-runner-nodejs \
  -v $PWD/dot-cache:/home/peertube/.cache/peertube-runner-nodejs \
  owntube/peertube-runner:v632 npx peertube-runner server
```

### Image Variant 2: `owntube/peertube-runner:v730` (`latest`) from PeerTube v7.3.0 with Runner v0.2.0

Build the container image:

```bash
docker buildx build --platform linux/amd64 -f Dockerfile.v730 -t owntube/peertube-runner:v730 .
docker tag owntube/peertube-runner:v730 owntube/peertube-runner:latest
```

Test running the PeerTube runner server:

```bash
docker run -it --rm -u 999 --name v730-runner-server \
  -v $PWD/dot-local:/home/peertube/.local/share/peertube-runner-nodejs \
  -v $PWD/dot-config:/home/peertube/.config/peertube-runner-nodejs \
  -v $PWD/dot-cache:/home/peertube/.cache/peertube-runner-nodejs \
  owntube/peertube-runner:v730 npx peertube-runner server
```

## Kubernetes Deployment

**Prerequisites:** Have a Kubernetes cluster with internet connectivity and persistent storage; the _StorageClass_
should support `.spec.accessModes[]` `ReadWriteMany` (e.g. [MicroK8s HostPath Storage](https://microk8s.io/docs/addon-hostpath-storage)),
as different container runtimes need to mount the PeerTube runner server socket as their mechanism of Inter-Process
Communication (IPC).

### Setup Step 1: Configure Persistent Storage

We need the following _PersistentVolumeClaims_ (PVCs):

1. `peertube-runner-local` for things persisted in `/home/peertube/.local/share/peertube-runner-nodejs`
2. `peertube-runner-config` for the tool internals in `/home/peertube/.config/peertube-runner-nodejs`
3. `peertube-runner-cache` for temp file storage in `/home/peertube/.cache/peertube-runner-nodejs`

If we had a namespace named `"peertube"` and a storage class named `"microk8s-hostpath"`, it could look like this:

```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: peertube-runner-local
  namespace: peertube
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: peertube-runner-config
  namespace: peertube
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: peertube-runner-cache
  namespace: peertube
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 10Gi
EOF
```

### Setup Step 2: Create a PeerTube Runner Pod

To create a _Pod_ with 2 containers in the namespace `"peertube"`, each running a PeerTube Runner server with the
_PersistentVolumes_ (PVs) from _Setup Step 1_ above, and apply a Kubernetes manifest like this one:

```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: peertube-runner-pod
  namespace: peertube
spec:
  securityContext:
    runAsUser: 999
    fsGroup: 999
  containers:
    - name: peertube-runner-1
      image: owntube/peertube-runner:v632
      command: ["npx", "peertube-runner"]
      args: ["server", "--id", "peertube-runner-1"]
      volumeMounts:
        - name: peertube-runner-local
          mountPath: /home/peertube/.local/share/peertube-runner-nodejs
        - name: peertube-runner-config
          mountPath: /home/peertube/.config/peertube-runner-nodejs
        - name: peertube-runner-cache
          mountPath: /home/peertube/.cache/peertube-runner-nodejs
    - name: peertube-runner-2
      image: owntube/peertube-runner:v730
      command: ["npx", "peertube-runner"]
      args: ["server", "--id", "peertube-runner-2"]
      volumeMounts:
        - name: peertube-runner-local
          mountPath: /home/peertube/.local/share/peertube-runner-nodejs
        - name: peertube-runner-config
          mountPath: /home/peertube/.config/peertube-runner-nodejs
        - name: peertube-runner-cache
          mountPath: /home/peertube/.cache/peertube-runner-nodejs
  volumes:
    - name: peertube-runner-local
      persistentVolumeClaim:
        claimName: peertube-runner-local
    - name: peertube-runner-config
      persistentVolumeClaim:
        claimName: peertube-runner-config
    - name: peertube-runner-cache
      persistentVolumeClaim:
        claimName: peertube-runner-cache
EOF
```

Get the pod's status and logs:

    kubectl get pods/peertube-runner-pod --namespace peertube -o wide
    kubectl logs peertube-runner-pod --namespace peertube

The logs should show no errors and indicate that the servers are up and idling.

### Setup Step 3: Register the Runners with PeerTube Instances

For illustration, let us assume that you have a PeerTube v6.3 instance that you want to connect `"peertube-runner-1"`
to, and a PeerTube v7.3 instance that you want to connect `"peertube-runner-2"` to.

Get the URLs and the _Registration Tokens_ for each of the PeerTube instances and register via `peertube-runner` CLI:

```bash
export PT_v63_RUNNER=peertube-runner-1
export PT_v63_URL=https://my-peertube63.tv
export PT_v63_TOKEN=ptrrt-e6657119-a21d-4217-75d8-1b491da3a169
kubectl exec peertube-runner-pod -n peertube -- npx peertube-runner --id $PT_v63_RUNNER \
  register --url $PT_v63_URL --registration-token $PT_v63_TOKEN \
  --runner-name my-$PT_v63_RUNNER --runner-description="OwnTube-tv/peertube-runner project"
# Verify it is registered:
kubectl exec peertube-runner-pod -n peertube -- npx peertube-runner --id $PT_v63_RUNNER \
  list-registered
'┌──────────────────────────┬──────────────────────┬────────────────────────────────────┐'
'│ instance                 │ runner name          │ runner description                 │'
'├──────────────────────────┼──────────────────────┼────────────────────────────────────┤'
'│ https://my-peertube63.tv │ my-peertube-runner-1 │ OwnTube-tv/peertube-runner project │'
'└──────────────────────────┴──────────────────────┴────────────────────────────────────┘'
```

```bash
export PT_v73_RUNNER=peertube-runner-2
export PT_v73_URL=https://my-peertube73.tv
export PT_v73_TOKEN=ptrrt-23586320-b92e-4521-21f7-3b4e1dc2b952
kubectl exec peertube-runner-pod -n peertube -- npx peertube-runner --id $PT_v73_RUNNER \
  register --url $PT_v73_URL --registration-token $PT_v73_TOKEN \
  --runner-name my-$PT_v73_RUNNER --runner-description="OwnTube-tv/peertube-runner project"
# Verify it is registered:
kubectl exec peertube-runner-pod -n peertube -- npx peertube-runner --id $PT_v73_RUNNER \
  list-registered
'┌──────────────────────────┬──────────────────────┬────────────────────────────────────┐'
'│ instance                 │ runner name          │ runner description                 │'
'├──────────────────────────┼──────────────────────┼────────────────────────────────────┤'
'│ https://my-peertube73.tv │ my-peertube-runner-2 │ OwnTube-tv/peertube-runner project │'
'└──────────────────────────┴──────────────────────┴────────────────────────────────────┘'
```

Once transcoding starts being processed, you should find that there are a few files in the persistent storage, but they
are not expected to accumulate over time in terms of volume.

Here is an illustration from my Kubernetes worker node, what it usually looks like (structurally):

```plain
/mnt/hostpath-lv/
└── microk8s-hostpath
    ├── peertube-runner-cache-pvc-3f416610-c92f-4707-94b5-4d5b25e1a803
    │   ├── peertube-runner-1
    │   │   └── transcoding
    │   │       ├── c0eb39c6-ad46-4cb5-9267-fee2760e7c93
    │   │       └── fcef49bb-66d6-40d8-a955-b5429aa42b2c
    │   └── peertube-runner-2
    │       └── transcoding
    │           └── c9faa5c0-3da0-4b73-83ca-2ff80e07f465
    ├── peertube-runner-config-pvc-00652c44-d67b-4806-8c08-2865501e4c63
    │   ├── peertube-runner-1
    │   │   └── config.toml
    │   └── peertube-runner-2
    │       └── config.toml
    └── peertube-runner-local-pvc-4946ea42-767d-4b32-be07-de231a59071f
        ├── peertube-runner-1
        │   └── peertube-runner.sock
        └── peertube-runner-2
            └── peertube-runner.sock
```

## Contributing

Do you want to contribute something? Join us on GitHub [here](https://github.com/OwnTube-tv/peertube-runner) and open
an issue, or just fork it and play around.
