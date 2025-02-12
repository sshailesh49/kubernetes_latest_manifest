### **🚀 AWS RDS को Kubernetes में ExternalName Service से एक्सेस करना**  

अगर आप अपने **Kubernetes क्लस्टर** के अंदर से **AWS RDS** (MySQL/PostgreSQL) से कनेक्ट करना चाहते हैं, तो आप **ExternalName Service** का उपयोग कर सकते हैं।  
इससे Kubernetes **DNS Resolution** को AWS RDS के **Endpoint** पर रीडायरेक्ट करेगा, जिससे बिना किसी लोड बैलेंसर या Ingress के RDS से कनेक्ट किया जा सकता है।

---

## **🔹 Step 1: AWS RDS Endpoint प्राप्त करें**  
सबसे पहले **AWS RDS Console** में जाएं और अपने **Database के Endpoint** को नोट करें।  
👉 Example: `mydb.abc123xyz456.us-east-1.rds.amazonaws.com`

---

## **🔹 Step 2: ExternalName Service YAML File बनाएं**  
ExternalName Service बनाने के लिए एक YAML फ़ाइल बनाएँ, जो AWS RDS के DNS नाम को पॉइंट करेगा।

### **📄 `external-rds-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-rds
spec:
  type: ExternalName
  externalName: mydb.abc123xyz456.us-east-1.rds.amazonaws.com
```

---

## **🔹 Step 3: Service को Apply करें**  
अब इस **YAML फ़ाइल** को Kubernetes क्लस्टर में Apply करें:

```sh
kubectl apply -f external-rds-service.yaml
```

👉 **Expected Output:**  
```
service/external-rds created
```

---

## **🔹 Step 4: Kubernetes से RDS को Test करें**  
अब चेक करें कि **ExternalName Service** सही से DNS Resolve कर रहा है या नहीं।

### **1️⃣ nslookup से DNS Resolution चेक करें**  
```sh
kubectl run test-pod --image=busybox -it --rm --restart=Never -- nslookup external-rds.default.svc.cluster.local
```
👉 **Expected Output:**  
```
Server:    10.96.0.10
Address:   10.96.0.10#53

Name:   external-rds.default.svc.cluster.local
Address:  mydb.abc123xyz456.us-east-1.rds.amazonaws.com
```
🔹 इसका मतलब यह हुआ कि `external-rds` सर्विस सही से AWS RDS को पॉइंट कर रही है।

---

## **🔹 Step 5: Kubernetes Pod से AWS RDS से कनेक्ट करना**  

### **1️⃣ MySQL Client से RDS कनेक्ट करें**  
अगर आपका RDS **MySQL** पर है:
```sh
kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- \
    mysql -h external-rds.default.svc.cluster.local -u admin -p
```
👉 अब MySQL पासवर्ड डालें और अगर सब कुछ सही है तो MySQL Shell में लॉगिन हो जाना चाहिए।

---

### **2️⃣ PostgreSQL Client से RDS कनेक्ट करें**  
अगर आपका RDS **PostgreSQL** पर है:
```sh
kubectl run psql-client --image=postgres -it --rm --restart=Never -- \
    psql -h external-rds.default.svc.cluster.local -U admin -d mydatabase
```
👉 PostgreSQL पासवर्ड डालें और कनेक्शन सफल होना चाहिए।

---

## **🔹 Bonus: Kubernetes Deployment से AWS RDS को कनेक्ट करना**  
अब हम Kubernetes **Deployment** में ExternalName Service का उपयोग करके **AWS RDS** को कनेक्ट करेंगे।

### **📄 `deployment-with-rds.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          env:
            - name: DB_HOST
              value: "external-rds.default.svc.cluster.local"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: rds-secret
                  key: username
            - name: DB_PASS
              valueFrom:
                secretKeyRef:
                  name: rds-secret
                  key: password
          ports:
            - containerPort: 8080
```
👉 यह **Deployment** `external-rds` सर्विस का उपयोग करके **AWS RDS** से कनेक्ट करेगा।

---

## **🔹 Step 6: Secrets के साथ AWS RDS Credentials को सुरक्षित रखना**  
RDS के **Username और Password** को सुरक्षित रखने के लिए Kubernetes **Secrets** का उपयोग करें।

```sh
kubectl create secret generic rds-secret --from-literal=username=admin --from-literal=password=MySecurePassword123
```

👉 अब आपका **MySQL/PostgreSQL Credentials** `rds-secret` में स्टोर हो जाएगा और **Deployment YAML** इसे उपयोग कर सकता है।

---

## **🔹 Step 7: Kubernetes Deployment को Apply करें**  
अब इस **Deployment** को Kubernetes में Apply करें:

```sh
kubectl apply -f deployment-with-rds.yaml
```

👉 अब आपका **Application** AWS RDS के साथ कनेक्ट हो जाएगा।

---

## **✅ निष्कर्ष**  
- **ExternalName Service** के जरिए **AWS RDS को Kubernetes में एक्सेस** करना आसान है।  
- कोई **LoadBalancer, Ingress, या Proxy** की जरूरत नहीं होती।  
- **DNS Resolution** के जरिए AWS RDS को **internal Kubernetes DNS** से कनेक्ट कर सकते हैं।  
- **Secrets का उपयोग** करके RDS Credentials को सुरक्षित रख सकते हैं।  
- यह **Microservices, Web Applications, और API Backends** के लिए उपयोगी है।

---

🚀 **अब आप Kubernetes से AWS RDS को आसानी से कनेक्ट कर सकते हैं!** 🎉  
**कोई और दिक्कत हो तो बताइए!** 😊
