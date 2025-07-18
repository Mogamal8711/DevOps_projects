**Step1: Deploy the secrts**

kubectl apply -f Secrts.yaml*

**Step2: Deploy the ConfigMaps**

kubectl apply -f ConfigMap.yaml

**Step3: App Deployment**

kubectl apply -f deplyment.yaml

**Step4: App Service Deployment**

Kubectl apply -f service.yaml

**Step5: Create ConfigMap for SQL database tables Creation**

kubectl create configmap mysql-init-config --from-file=init.sql

**Step6: Create the PersistentVolume**

kubectl apply -f todo-app-pv.yaml

**Step7: Create The PersistentVolumeClaim**

kubectl apply -f todo-app-pvc.yaml



