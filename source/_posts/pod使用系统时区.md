---
title: pod使用系统时区
date: 2021-10-22 14:39:03
tags:
- kubernetes
---



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: '1'
  creationTimestamp: '2021-10-22T06:34:14Z'
  generation: 1
  labels:
    app: myapp
  name: myapp
  namespace: default
  resourceVersion: '7352546528'
  selfLink: /apis/apps/v1/namespaces/default/deployments/myapp
  uid: bcfd5b8b-2403-4699-af23-538d1a4b8dbf
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - image: 'nginx:latest'
          imagePullPolicy: Always
          name: myapp
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
		  startupProbe: # StartupProbe indicates that the Pod has successfully initialized.
		    httpGet:
		      path: /health
		      port: 5000
		    failureThreshold: 30
		    periodSeconds: 10
		  readinessProbe: # Periodic probe of container service readiness.
		    httpGet:
		      path: /health
		      port: 5000
		    failureThreshold: 1
		    periodSeconds: 10
		  livenessProbe:  # Periodic probe of container liveness.
		    httpGet:
		      path: /health
		      port: 5000
		    failureThreshold: 1
		    periodSeconds: 10
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /etc/localtime
              name: volume-localtime
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - hostPath:
            path: /etc/localtime
            type: ''
          name: volume-localtime
status:
  conditions:
    - lastTransitionTime: '2021-10-22T06:34:14Z'
      lastUpdateTime: '2021-10-22T06:34:14Z'
      message: Deployment does not have minimum availability.
      reason: MinimumReplicasUnavailable
      status: 'False'
      type: Available
    - lastTransitionTime: '2021-10-22T06:34:14Z'
      lastUpdateTime: '2021-10-22T06:34:14Z'
      message: ReplicaSet "myapp-59db5f9878" is progressing.
      reason: ReplicaSetUpdated
      status: 'True'
      type: Progressing
  observedGeneration: 1
  replicas: 1
  unavailableReplicas: 1
  updatedReplicas: 1
```

