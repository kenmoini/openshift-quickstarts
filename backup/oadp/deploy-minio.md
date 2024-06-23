# Deploy Minio on OpenShift

In order to test OADP and/or other workloads that need S3 storage you can quickly deploy Minio in OpenShift.

## Create a Namespace

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: minio
```

## Create a PersistentVolumeClaim

If you do not have a default StorageClass or prefer a different one you will need to specify it with the commented out `.spec.storageClassName` option below:

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: minio-pvc
  namespace: minio
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  volumeMode: Filesystem
  #storageClassName: my-storage-class
```

## Create a Secret for the root user

```yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: minio-secret
  namespace: minio
stringData:
  # change the username and password to your own values.
  # ensure that the user is at least 3 characters long and the password at least 8
  minio_root_user: minio
  minio_root_password: minio123
```

## Create the Deployment

```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: minio
  namespace: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: minio
    spec:
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-pvc
      containers:
        - resources:
            limits:
              cpu: 250m
              memory: 1Gi
            requests:
              cpu: 20m
              memory: 100Mi
          readinessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          terminationMessagePath: /dev/termination-log
          name: minio
          livenessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 30
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          env:
            - name: MINIO_ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: minio_root_user
            - name: MINIO_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: minio_root_password
          ports:
            - containerPort: 9000
              protocol: TCP
            - containerPort: 9090
              protocol: TCP
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: data
              mountPath: /data
              subPath: minio
          terminationMessagePolicy: File
          image: >-
            quay.io/minio/minio:RELEASE.2023-06-19T19-52-50Z
          args:
            - server
            - /data
            - --console-address
            - :9090
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
  strategy:
    type: Recreate
  revisionHistoryLimit: 3
  progressDeadlineSeconds: 600
```

## Create the Service and Routes

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: minio-service
  namespace: minio
spec:
  ipFamilies:
    - IPv4
  ports:
    - name: api
      protocol: TCP
      port: 9000
      targetPort: 9000
    - name: ui
      protocol: TCP
      port: 9090
      targetPort: 9090
  internalTrafficPolicy: Cluster
  type: ClusterIP
  ipFamilyPolicy: SingleStack
  sessionAffinity: None
  selector:
    app: minio
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: minio-api
  namespace: minio
spec:
  to:
    kind: Service
    name: minio-service
    weight: 100
  port:
    targetPort: api
  wildcardPolicy: None
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: minio-ui
  namespace: minio
spec:
  to:
    kind: Service
    name: minio-service
    weight: 100
  port:
    targetPort: ui
  wildcardPolicy: None
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Once those manifest objects have been created, ensure the Pod has started and access the `minio-ui` Route.  Continue by logging in as the root user set in the Secret above.

To use Minio for OADP you can follow the next set of procedures - this can easily be modified for other services, eg the Internal Image Registry.

1. Create a Bucket, eg `ocp-oadp` - make sure Versioning, Object Locking, and Quota are off.
2. Create a Policy called `allow_rw_ocp-oadp` with the following contents:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::ocp-oadp/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::ocp-oadp"
            ]
        }
    ]
}
```

If this is being used for other buckets/workloads, make sure to match the bucket name to what is defiend in the Resource sections.

3. Create a new User, eg `ocp-oadp` and assign it the `allow_rw_ocp-oadp` Policy.
4. Log out of the root user and log in as the `ocp-oadp` user - ensure you can access the bucket.
5. Create an Access Key as this user and save the details.

The Access Key ID and Access Secret will be needed for integrating into various services.