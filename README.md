# OpenShift Kullanıcı Grupları ve Yetkilendirme

Bu doküman, OpenShift üzerinde `oc` komutları ve YAML manifestleri ile kullanıcı grupları oluşturmayı ve yetkilendirmeyi açıklar.

## **📌 1. Kullanıcı Grupları ve Yetkileri**

| Grup Adı  | Kullanıcılar | Namespace | Yetkiler |
|-----------|------------|------------|----------------------------------------|
| **ng1**  | user1, user2 | ns1 | Full yetki (Admin) |
| **ng2**  | user3, user4 | ns2 | Full yetki (Admin) - **NetworkPolicy değiştiremez** |
| **ng3**  | user4, user5 | Tüm namespace’ler | **Read-Only**, Secret listeleyemez |
| **devops-admin**  | (isteğe bağlı) | Tüm namespace’ler | Deployment yönetebilir |
| **jenkins**  | (isteğe bağlı) | Belirtilen namespace | CI/CD işlemleri için özel yetkiler |

---

## **📌 2. OpenShift `oc` Komutları ile Grup ve Yetki Tanımlama**

### **1️⃣ Kullanıcı Gruplarını Oluştur**
```bash
oc adm groups new ng1 user1 user2
oc adm groups new ng2 user3 user4
oc adm groups new ng3 user4 user5
```

### **2️⃣ Namespace'leri Oluştur (Eğer Yoksa)**
```bash
oc new-project ns1
oc new-project ns2
```

### **3️⃣ Gruplara Rolleri Atama**

#### **🔹 `ng1` - Full Admin Yetkisi (ns1 Namespace için)**
```bash
oc create rolebinding ng1-admin --group=ng1 --clusterrole=admin -n ns1
```

#### **🔹 `ng2` - Full Admin (NetworkPolicy Hariç)**
Önce `networkpolicy` hariç admin rolü tanımla:
```bash
cat <<EOF | oc apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: ns2
  name: admin-without-networkpolicy
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["get", "list", "watch"] # update veya delete yok
EOF
```
Sonra bu rolü `ng2` grubuna bağla:
```bash
oc create rolebinding ng2-admin --group=ng2 --role=admin-without-networkpolicy -n ns2
```

#### **🔹 `ng3` - ReadOnly (Secrets Hariç)**
Önce özel readonly rolünü tanımla:
```bash
cat <<EOF | oc apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-without-secrets
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: [] # Hiçbir yetki verilmediği için erişemez
EOF
```
Sonra `ng3` grubuna bu rolü bağla:
```bash
oc create clusterrolebinding ng3-readonly --group=ng3 --clusterrole=readonly-without-secrets
```

---

## **📌 3. Ekstra Roller**

### **🔹 DevOps Admin Rolü** (Tüm Deployment işlemleri için)
```bash
oc create clusterrole devops-admin --verb=* --resource=deployments.apps
oc create clusterrolebinding devops-admin-binding --group=devops-admin --clusterrole=devops-admin
```

### **🔹 Jenkins CI/CD Kullanıcısı**
```bash
oc create sa jenkins -n ci-cd
oc adm policy add-cluster-role-to-user edit -z jenkins -n ci-cd
```

---

## **📌 4. YAML ile Tanımlamalar**

Aşağıdaki YAML dosyalarıyla tüm bu işlemleri uygulayabilirsin.

### **🔹 `rolebindings.yaml`**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ng1-admin
  namespace: ns1
subjects:
- kind: Group
  name: ng1
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ng2-admin
  namespace: ns2
subjects:
- kind: Group
  name: ng2
roleRef:
  kind: Role
  name: admin-without-networkpolicy
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ng3-readonly
subjects:
- kind: Group
  name: ng3
roleRef:
  kind: ClusterRole
  name: readonly-without-secrets
  apiGroup: rbac.authorization.k8s.io
```

### **🔹 `groups.yaml`**
```yaml
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: ng1
users:
  - user1
  - user2
---
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: ng2
users:
  - user3
  - user4
---
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: ng3
users:
  - user4
  - user5
```

---

## **📌 5. YAML Dosyalarını OpenShift’e Uygulama**
```bash
oc apply -f groups.yaml
oc apply -f rolebindings.yaml
```

Her şey hazır! 🚀 

