# Kubernetesクラスタ上でWordpressを構築する

## Mysql

### 1. Mysql用シークレットの生成
kubectl create secret generic mysql --from-literal=password=JkdJkd123@@@

### 2. pv/pvcの作成

**mysql-pv.yml**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    type: local
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /tmp/data/mysql
```

**musql-pvc.yml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: wordpress
    tier: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

上記ファイルを作成し、以下を実行

```sh
kubectl create -f mysql-pv.yml
kubectl create -f mysql-pvc.yml
kubectl get pvc,pv
```

### 3. deploymentを作成

**mysql.yml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql:5.7
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-local-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-local-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
```

上記ファイルを作成し、以下を実行

```sh
kubectl create -f mysql.yml
kubectl get pod -l app=mysql
```

### 4. serviceを作成

**mysql-service.yml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  type: ClusterIP
  ports:
    - port: 3306
  selector:
    app: mysql
```

上記ファイルを作成し、以下を実行

```sh
kubectl create -f mysql-service.yml
kubectl get service mysql
```

## Wordpress

### 1. pv/pvcの作成

**wordpress-pv.yml**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
  labels:
    type: local
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /tmp/data/wordpress
```

**wordpress-pv.yml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  labels:
    app: wordpress
    tier: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

上記ファイルを作成し、以下を実行

```sh
kubectl create -f wordpress-pv.yml
kubectl create -f wordpress-pvc.yml
kubectl get pvc,pv
```

### 2.deploymentの作成

**wordpress.yml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - image: wordpress
          name: wordpress
          env:
          - name: WORDPRESS_DB_HOST
            value: mysql:3306
          - name: WORDPRESS_DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql
                key: password
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-local-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-local-storage
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

上記ファイルを作成し、以下を実行

```sh
kubectl create -f wordpress.yml
kubectl get pod -l app=wordpress
```

### 3. serviceの作成

**wordpress-service.yml**

```yaml
kind: Service
metadata:
  labels:
    app: wordpress
  name: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: wordpress
```

上記ファイルを作成し、以下を実行

```sh
kubectl create -f wordpress-service.yml
kubectl get svc -l app=wordpress
```

## Try

### アクセスしてみる

以下コマンドを実行する。

```sh
kubectl cluster-info
```

MasterIPが確認できたので、wordpressサービスのNodePortを用いてアクセスする。  
`http://MasterIP:NodePort/wp-admin/install.php/`


### セルフヒーリングを試す

```sh
kubectl get pod
kubectl delete pod -l app=wordpress
kubectl get pod
```

### スケールアウトを試す
```sh
kubectl scale deployment wordpress --replicas 10
kubectl get pod
```
