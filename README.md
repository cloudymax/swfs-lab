# Backup and Recovery with K8up

This guide will walk you through the creation, backup, and recovery processes for a local [SeaweedFS](https://github.com/seaweedfs/seaweedfs) deployment using [K8up](https://k8up.io/) and [Backblaze B2](https://www.backblaze.com/docs/cloud-storage).

K8up is a Kubernetes-native wrapper for [Restic](https://restic.readthedocs.io/en/stable/) therefore users of Restic and other Restic-based tooling like [Velero](https://velero.io/) should also find the techniques described here useful. Users of other S3 hosted services such as [Wasabi S3](https://wasabi.com/), [Cloudflare R2](https://www.cloudflare.com/developer-platform/r2/) etc... should also be able to follow along.

> For the purposes of this demo, backups are set to run very frequently and plain-text passwords are also used for convenience - do NOT do that in production.

PR ref: https://github.com/seaweedfs/seaweedfs/pull/5034

## Outline

1. [K3s Cluster creation](#k3s-cluster-creation)
2. [SeaweedFS instance setup](#seaweedfs-instance-and-user-setup)
3. [Configure scheduled backups of SeaweedFS to B2](#configure-scheduled-backups-of-seaweedfs-to-b2)
4. [Restore SeaweedFS from B2 backups](#restore-seaweedfs-from-b2-backups)


## Requirements

- [Kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [Restic](https://restic.readthedocs.io/en/latest/020_installation.html)
- S3 CLI tool of your choice, I'll be using [mc](https://github.com/minio/mc)

<h2 id="k3s-cluster-creation">K3s Cluster creation</h2>

1. Download the k3s installer

    ```bash
    curl -sfL https://get.k3s.io > k3s-install.sh
    ```

2. install k3s

    ```bash
    bash k3s-install.sh --disable=traefik
    ```

3. Wait for node to be ready

    ```console
    $ sudo k3s kubectl get node
    NAME   STATUS   ROLES                  AGE   VERSION
    vm0    Ready    control-plane,master   1m   v1.27.4+k3s1
    ```

4. Make an accessible version of the kubeconfig

    ```bash
    mkdir -p ~/.config/kube

    sudo cp /etc/rancher/k3s/k3s.yaml ~/.config/kube/config

    sudo chown $USER:$USER ~/.config/kube/config

    export KUBECONFIG=~/.config/kube/config
    ```

5. Install k8up

    ```bash
    helm repo add k8up-io https://k8up-io.github.io/k8up
    helm repo update

    kubectl apply -f https://github.com/k8up-io/k8up/releases/download/k8up-4.4.3/k8up-crd.yaml
    helm install k8up k8up-io/k8up
    ```


<h2 id="seaweedfs-instance-and-user-setup">SeaweedFS instance and user setup</h2>

1. install the MinIO client

    Docs: https://min.io/docs/minio/linux/reference/minio-mc.html

    ```bash
    mkdir -p $HOME/minio-binaries

    wget https://dl.min.io/client/mc/release/linux-amd64/mc -O $HOME/minio-binaries/mc

    chmod +x $HOME/minio-binaries/mc

    export PATH=$PATH:$HOME/minio-binaries/
    ```

2. Download SeaweedFS, unzip, then cd to the helm dir

    ```bash
    wget https://github.com/seaweedfs/seaweedfs/archive/refs/tags/3.77.zip
    unzip 3.77.zip
    cd seaweedfs-3.77/k8s/charts/seaweedfs
    ```

3. Create a minimal values file for the Seaweedfs deployment which adds annotations for K8up.

    ```bash
    /bin/cat << EOF > test-values.yaml
    master:
      enabled: true
      data:
        type: "persistentVolumeClaim"
        size: "10G"
        storageClass: "local-path"
        annotations:
          "k8up.io/backup": "true"
      livenessProbe:
        periodSeconds: 5
      readinessProbe:
        periodSeconds: 5

    volume:
      enabled: true
      readMode: proxy
      dataDirs:
        - name: data
          type: "persistentVolumeClaim"
          size: "10G"
          storageClass: "local-path"
          annotations:
            "k8up.io/backup": "true"
          maxVolumes: 0
      idx: {}
      livenessProbe:
        periodSeconds: 5
      readinessProbe:
        periodSeconds: 5

    filer:
      enabled: true
      encryptVolumeData: true
      enablePVC: true
      storage: 10Gi
      defaultReplicaPlacement: "000"
      data:
        type: "persistentVolumeClaim"
        size: "10G"
        storageClass: "local-path"
        annotations:
          "k8up.io/backup": "true"
      s3:
        enabled: true
        enableAuth: true
        port: 8333
        httpsPort: 0
        allowEmptyFolder: false
        createBuckets:
          - name: shared
            anonymousRead: false
      livenessProbe:
        periodSeconds: 5
      readinessProbe:
        periodSeconds: 5

    s3:
      enabled: false
    cosi:
      enabled: false
    EOF
    ```

4. Deploy via Helm (takes longer on slow drives)

   ```bash
   helm install seaweedfs . -f test-values.yaml --wait
   ```

5. Expose the filer service via a LoadBalancer (servicelb in k3s). This will let us view the admin UI as well as reach the S3 endpoint during the demo.

    ```bash
    /bin/cat << EOF > service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app.kubernetes.io/component: filer
        app.kubernetes.io/instance: seaweedfs
        app.kubernetes.io/name: seaweedfs
      name: seaweedfs-filer-lb
    spec:
      ports:
      - name: swfs-filer
        port: 8888
        protocol: TCP
        targetPort: 8888
      - name: swfs-filer-grpc
        port: 18888
        protocol: TCP
        targetPort: 18888
      - name: swfs-s3
        port: 8333
        protocol: TCP
        targetPort: 8333
      - name: metrics
        port: 9327
        protocol: TCP
        targetPort: 9327
      selector:
        app.kubernetes.io/component: filer
        app.kubernetes.io/name: seaweedfs
      type: LoadBalancer
    EOF
    ```

6. Export your LoadBalancer IP address as an env var

   ```bash
   export NODE_IP=""
   ```

7. Create an alias for your server using your S3 CLI tool:

    - You can find the `admin_access_key_id` and `admin_secret_access_key` values in the secret `seaweedfs-s3-secret`

      ```bash
      mc alias set seaweedfs http://$NODE_IP:8333 $admin_access_key_id $admin_secret_access_key
      ```

8. Create a bucket that will hold our demo data

      ```bash
      mc mb seaweedfs/backups
      ```

8. Add some data to the bucket

      ```bash
      mc cp ./some-file seaweedfs/backups/
      ```

9. Verify its there

      ```bash
      mc ls seaweedfs/backups
      ```

10. Open the Web UI at http://$NODE_IP:8888 in a browser to view or add more data.

 <h2 id="configure-scheduled-backups-of-seaweedf3-to-b2">Configure scheduled backups of SeaweedFS to B2</h2>

 1. Create a secret containing your external S3 credentials

    - You will need to get these from your provider (Backblaze, Wasabi etc..):

      ```bash
      export ACCESS_KEY_ID=$(echo -n "" | base64)

      export ACCESS_SECRET_KEY=$(echo -n "" |base64)
      ```

      ```bash
      /bin/cat << EOF > backblaze-secret.yaml
      apiVersion: v1
      kind: Secret
      metadata:
        name: backblaze-credentials
      type: Opaque
      data:
        "ACCESS_KEY_ID": "$ACCESS_KEY_ID"
        "ACCESS_SECRET_KEY": "$ACCESS_SECRET_KEY"
      EOF

      kubectl apply -f backblaze-secret.yaml
      ```

 2. Create a secret containing a random password for restic

    - Generate a password.

      ```bash
      export RESTIC_PASS=""
      ```

    - Create a secret manifest

      ```bash
      /bin/cat << EOF > restic.yaml
      apiVersion: v1
      kind: Secret
      metadata:
        name: restic-repo
      type: Opaque
      stringData:
        "password": "$RESTIC_PASS"
      EOF
      ```

    - Create the secret

      ```bash
      kubectl apply -f restic.yaml
      ```

3. Create a scheduled backup

    - Export your S3 address:

      ```bash
      export BACKUP_S3_URL=""
      export BACKUP_S3_BUCKET=""
      ```

    - Create a manifest for the backup

      ```bash
      /bin/cat << EOF > backup.yaml
      apiVersion: k8up.io/v1
      kind: Schedule
      metadata:
        name: schedule-backups
      spec:
        backend:
          repoPasswordSecretRef:
            name: restic-repo
            key: password
          s3:
            endpoint: "$BACKUP_S3_URL"
            bucket: "$BACKUP_S3_BUCKET"
            accessKeyIDSecretRef:
              name: backblaze-credentials
              key: ACCESS_KEY_ID
            secretAccessKeySecretRef:
              name: backblaze-credentials
              key: ACCESS_SECRET_KEY
        backup:
          schedule: '*/5 * * * *'
          keepJobs: 4
        check:
          schedule: '0 1 * * 1'
        prune:
          schedule: '0 1 * * 0'
          retention:
            keepLast: 5
            keepDaily: 14
      EOF
      ```

    - Create the backup and let it run

      ```bash
      kubectl apply -f backup.yaml
      ```

<h2 id="restore-seaweedfs-from-b2-backups">Restore SeaweedFS from B2 backups</h2>

1. Uninstall SeaweedFS, delete the PVCs, secrets, and scheduled backup

   ```bash
   kubectl delete -f backup.yaml
   helm uninstall seaweedfs
   kubectl delete pvc data-default-seaweedfs-master-0
   kubectl delete pvc data-filer-seaweedfs-filer-0
   kubectl delete pvc data-seaweedfs-volume-0
   ```

2. Create PVCs to hold our restored data

    - Create a manifest for the PVCs

      ```bash
      /bin/cat << EOF > pvc.yaml
      ---
      kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: swfs-volume-data
        annotations:
          "k8up.io/backup": "true"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
      ---
      kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: swfs-master-data
        annotations:
          "k8up.io/backup": "true"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
      ---
      kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: swfs-filer-data
        annotations:
          "k8up.io/backup": "true"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
      EOF
      ```

    - Create the PVCs

      ```bash
      kubectl apply -f pvc.yaml
      ```

3. Setup your restic credentials

    ```bash
    # the password used in your restic-repo secret
    export RESTIC_PASSWORD=""

    # Your S3 credentials
    export AWS_ACCESS_KEY_ID=""
    export AWS_SECRET_ACCESS_KEY=""
    export RESTIC_REPOSITORY="s3://$BACKUP_S3_URL/$BACKUP_S3_BUCKET"
    ```

4. Find your desired snapshot to restore

    ```console
    $ restic snapshots
    repository d91e9530 opened (version 2, compression level auto)
    created new cache in /home/friend/.cache/restic
    ID        Time                 Host        Tags        Paths
    --------------------------------------------------------------------------------------------
    4a25424a  2023-11-20 19:40:10  default                 /data/data-default-seaweedfs-master-0
    649b25c7  2023-11-20 19:40:14  default                 /data/data-filer-seaweedfs-filer-0
    99160498  2023-11-20 19:40:19  default                 /data/data-seaweedfs-volume-0
    --------------------------------------------------------------------------------------------
    3 snapshots
    ```

4. Use the K8up CLI or a declarative setup to restore data to the PVC. You will need to do this for each PVC that needs to be restored

    - Example manifest for a S3-to-PVC restore job which uses the restic snapshots shown above.

      ```bash
      /bin/cat << EOF > s3-to-pvc.yaml
      ---
      apiVersion: k8up.io/v1
      kind: Restore
      metadata:
        name: restore-volume-data
      spec:
        restoreMethod:
          folder:
            claimName: swfs-volume-data
        snapshot: "99160498"
        backend:
          repoPasswordSecretRef:
            name: restic-repo
            key: password
          s3:
            endpoint: "$BACKUP_S3_URL"
            bucket: "$BACKUP_S3_BUCKET"
            accessKeyIDSecretRef:
              name: backblaze-credentials
              key: ACCESS_KEY_ID
            secretAccessKeySecretRef:
              name: backblaze-credentials
              key: ACCESS_SECRET_KEY
      ---
      apiVersion: k8up.io/v1
      kind: Restore
      metadata:
        name: restore-master-data
      spec:
        restoreMethod:
          folder:
            claimName: swfs-master-data
        snapshot: "4a25424a"
        backend:
          repoPasswordSecretRef:
            name: restic-repo
            key: password
          s3:
            endpoint: "$BACKUP_S3_URL"
            bucket: "$BACKUP_S3_BUCKET"
            accessKeyIDSecretRef:
              name: backblaze-credentials
              key: ACCESS_KEY_ID
            secretAccessKeySecretRef:
              name: backblaze-credentials
              key: ACCESS_SECRET_KEY
      ---
      apiVersion: k8up.io/v1
      kind: Restore
      metadata:
        name: restore-filer-data
      spec:
        restoreMethod:
          folder:
            claimName: swfs-filer-data
        snapshot: "649b25c7"
        backend:
          repoPasswordSecretRef:
            name: restic-repo
            key: password
          s3:
            endpoint: "$BACKUP_S3_URL"
            bucket: "$BACKUP_S3_BUCKET"
            accessKeyIDSecretRef:
              name: backblaze-credentials
              key: ACCESS_KEY_ID
            secretAccessKeySecretRef:
              name: backblaze-credentials
              key: ACCESS_SECRET_KEY
      EOF
      ```

    - Apply manifest
      ```bash
      kubectl apply -f s3-to-pvc.yaml
      ```

5. Re-deploy Seaweedfs from the existing PVCs

   - Create a manifest that targets the PVCs we created

      ```bash
      /bin/cat << EOF > restore-values.yaml
      master:
        enabled: true
        data:
          type: "existingClaim"
          claimName: "swfs-master-data"
        livenessProbe:
          periodSeconds: 5
        readinessProbe:
          periodSeconds: 5

      volume:
        enabled: true
        readMode: proxy
        dataDirs:
          - name: data
            type: "existingClaim"
            claimName: "swfs-volume-data"
            maxVolumes: 0
        idx: {}
        livenessProbe:
          periodSeconds: 5
        readinessProbe:
          periodSeconds: 5

      filer:
        enabled: true
        encryptVolumeData: true
        enablePVC: true
        storage: 10Gi
        defaultReplicaPlacement: "000"
        data:
          type: "existingClaim"
          claimName: "swfs-filer-data"
        s3:
          enabled: true
          enableAuth: false
          port: 8333
          httpsPort: 0
          allowEmptyFolder: false
        livenessProbe:
          periodSeconds: 5
        readinessProbe:
          periodSeconds: 5

      s3:
        enabled: false
      cosi:
        enabled: false
      EOF
      ```

    - Deploy via Helm

      ```bash
      helm install seaweedfs . -f restore-values.yaml --wait
      ```

7. Update your alias for your server:

    - get the `admin_access_key_id` and `admin_secret_access_key` from the secret `seaweedfs-s3-secret`

      ```bash
      mc alias set seaweedfs http://$NODE_IP:30000 $admin_access_key_id $admin_secret_access_key
      ```

    - View for your data:

      ```bash
      mc ls seaweedfs
      ```
