# kaniko 起步文档

简介：kaniko是一个构建镜像的工具，可以用于替代docker cli工具，集成到云平台中，可以实现进行镜像的无缝整合。

## 一. 准备条件

1. dokerhub账号
2. k8s集群

## 二. 操作步骤

第一步：部署pv卷

volume.yaml

```yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: dockerfile
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /data/kaniko ##目录提前创建好
```

第二步：部署pvc

volume-claim.yaml

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: dockerfile-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual
```

第三步： 创建secret

```sh
kubectl create secret docker-registry regcred \
	--docker-server=https://index.docker.io/v1/ \
       	--docker-username=**** \
	--docker-password=**** \
	--docker-email=***
```

第四步：部署pod

pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    imagePullPolicy: IfNotPresent
    args: ["--dockerfile=/workspace/dockerfile",
            "--context=dir://workspace",
            "--destination=linjinbao66/ubuntu:0.0.2"]
            #"--tarPath=/workspace/ubuntu-network-utils-docker-image.tar"]
    volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker
      - name: dockerfile-storage
        mountPath: /workspace
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
  restartPolicy: Never
  volumes:
    - name: kaniko-secret
      secret:
        secretName: regcred
        items:
          - key: .dockerconfigjson
            path: config.json
    - name: dockerfile-storage
      persistentVolumeClaim:
        claimName: dockerfile-claim
```

备注：国内由于GFW污染原因，需要挂载DNS为谷歌的8.8.8.8；`--tarPath=/workspace/ubuntu-network-utils-docker-image.tar`参数如果定义则会保存到本地。

第五步：查看日志

`kubectl logs kaniko`：

```bash
E0108 07:45:19.973167       1 aws_credentials.go:77] while getting AWS credentials NoCredentialProviders: no valid providers in chain. Deprecated.
	For verbose messaging see aws.Config.CredentialsChainVerboseErrors
INFO[0011] Retrieving image manifest ubuntu             
INFO[0011] Retrieving image ubuntu                      
INFO[0031] Retrieving image manifest ubuntu             
INFO[0031] Retrieving image ubuntu                      
INFO[0037] Built cross stage deps: map[]                
INFO[0037] Retrieving image manifest ubuntu             
INFO[0037] Retrieving image ubuntu                      
INFO[0043] Retrieving image manifest ubuntu             
INFO[0043] Retrieving image ubuntu                      
INFO[0053] Executing 0 build triggers                   
INFO[0053] Skipping unpacking as no commands require it. 
INFO[0053] ENTRYPOINT ["/bin/bash", "-c", "echo hello"] 
```