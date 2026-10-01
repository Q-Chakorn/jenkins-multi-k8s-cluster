# Jenkins บน Kubernetes

ชุด manifest นี้ใช้สำหรับติดตั้ง Jenkins master บน Kubernetes พร้อมพื้นที่เก็บข้อมูลถาวร, Jenkins web UI, ช่องทางเชื่อมต่อ agent แบบ JNLP และ Service สำหรับชี้ไปยัง Jenkins slave ที่อยู่นอก cluster

## ภาพรวมการทำงาน

- สร้าง namespace `jenkins-q` สำหรับทรัพยากรของ Jenkins ที่เป็น namespaced resource ส่วน PV เป็นทรัพยากรระดับ cluster
- รัน Jenkins จาก image `jenkins/jenkins:2.479.3-lts-jdk17` โดยมี `initContainer` สำหรับติดตั้ง plugins จาก `plugins.txt`
- โหลด `init-config.groovy` เพื่อสร้างผู้ใช้ `admin`, ปิด anonymous read, เปิด CSRF และกำหนด JNLP port `50000`
- เก็บ Jenkins home ที่ `/jenkins-data/jenkins-Q-home` บน node ผ่าน `hostPath` โดย PV/PVC ขอ capacity `1Gi` แต่ `hostPath` ไม่ได้จำกัดพื้นที่จริงของ directory โดยอัตโนมัติ
- เปิด Jenkins web UI ผ่าน Service `jenkins-web` ที่ port `80` และส่งต่อไปยัง container port `8080`
- เปิด JNLP agent ผ่าน NodePort `30001` และส่งต่อไปยัง port `50000`
- เปิด web UI ผ่าน NGINX Ingress ที่ path `/chakorn/jenkins`
- สร้าง Service `jenkins-slave-agent` ที่ port `6443` เพื่อชี้ไปยัง external endpoints ของ slave cluster

โครงสร้าง manifest หลัก:

```text
manifests/
├── jenkins-master-ns.yaml
├── jenkins-master-pv.yaml
├── jenkins-master-pvc.yaml
├── jenkins-master-init-configmap.yaml
├── jenkins-master-svc.yaml
├── jenkins-master-deploy.yaml
├── jenkins-master-ingress.yaml
├── jenkins-master-network-policy.yaml
└── slave-cluster/
    ├── jenkins-slave-ext-svc.yaml
    └── jenkins-slave-ext-endpoint.yaml
```

## สิ่งที่ต้องมี

- Kubernetes cluster และ `kubectl` ที่เชื่อมต่อ cluster เป้าหมายแล้ว
- NGINX Ingress Controller หากต้องการใช้งานผ่าน Ingress
- สร้าง directory `/jenkins-data/jenkins-Q-home` บน node ที่จะรัน Jenkins และให้ kubelet/container UID/GID `1000` เข้าถึงได้
- ตรวจสอบว่า NodePort `30001` และ port `6443` เปิดให้เครือข่ายที่เกี่ยวข้องเข้าถึง
- แก้ IP ของ external Jenkins slave ให้เป็นค่าจริงใน `manifests/slave-cluster/jenkins-slave-ext-endpoint.yaml`
- หากใช้ NetworkPolicy ต้องกำหนด namespace หรือ CIDR ของ agent ที่ได้รับอนุญาตให้เชื่อมต่อ JNLP

## ตรวจสอบก่อนติดตั้ง

Manifest ถูกจัดรูปแบบให้ใช้ namespace `jenkins-q`, PVC `jenkins-home-pvc`, Ingress API รุ่นใหม่ และ replica เดียวแล้ว ก่อนติดตั้งต้องตรวจสอบค่าที่ขึ้นกับสภาพแวดล้อมจริง:

### 1. กำหนด IP ของ external Jenkins slave

แทนค่าข้อความ `${JENKINS_SLAVE_IP}` และ `${JENKINS_SLAVE_IP2}` ใน `manifests/slave-cluster/jenkins-slave-ext-endpoint.yaml` ด้วย IP จริงก่อน apply ตัวอย่างด้านล่างใช้เพื่ออธิบายรูปแบบเท่านั้น:

```yaml
metadata:
  name: jenkins-slave-agent
  namespace: jenkins-q
subsets:
  - addresses:
      - ip: <JENKINS_SLAVE_IP>
      - ip: <JENKINS_SLAVE_IP2>
    ports:
      - port: 6443
```

`kubectl apply` ไม่แทนค่า `${...}` ให้เอง หากใช้ template ต้อง render ด้วยเครื่องมือที่เหมาะสม เช่น `envsubst` และควรตรวจว่าไฟล์ที่ render แล้วไม่มี placeholder เหลืออยู่

ตรวจสอบไฟล์หลังแก้ด้วย:

```bash
kubectl apply --dry-run=client -f manifests/slave-cluster/jenkins-slave-ext-endpoint.yaml
```

### 2. ตรวจสอบ storage และสิทธิ์ของ directory

สร้าง `/jenkins-data/jenkins-Q-home` บน node ที่จะรัน Pod และตรวจสอบให้ UID/GID `1000` เขียนได้ เนื่องจาก Jenkins ทำงานแบบ non-root แล้ว ปัจจุบัน Deployment ยังไม่มี node affinity จึงควรเพิ่ม affinity/nodeSelector หรือใช้ storage ที่ย้ายข้าม node ได้ หากย้าย node ต้องย้ายข้อมูลและกำหนด storage ให้สอดคล้องกัน

### 3. ตรวจสอบ Ingress และ NetworkPolicy

ต้องติดตั้ง NGINX Ingress Controller และตรวจสอบว่า namespace ของ controller มี label `kubernetes.io/metadata.name=ingress-nginx` หากใช้ชื่อ namespace อื่นให้แก้ใน NetworkPolicy

NetworkPolicy ปัจจุบันอนุญาต JNLP เฉพาะจาก namespace ที่มี label `jenkins-agent-access: "true"` เท่านั้น ดังนั้น external agent จะยังเชื่อมต่อไม่ได้จนกว่าจะเพิ่ม `ipBlock` ที่เป็น CIDR จริงของ agent ใน `jenkins-master-network-policy.yaml` และจำกัด firewall ของ NodePort `30001` ให้ตรงกับ CIDR เดียวกัน

PV ขอ capacity `1Gi` ใช้ `hostPath` และ `ReadWriteOnce` จึงตั้ง Deployment เป็น `replicas: 1` เพื่อป้องกันการ mount พร้อมกันและข้อมูลเสียหาย ทั้งนี้ `hostPath` ไม่ได้บังคับ quota ของ directory จริง

Ingress ปัจจุบันไม่มี hostname และ TLS อีกทั้งตั้ง `ssl-redirect: "false"` จึงให้บริการผ่าน HTTP ที่ไม่เข้ารหัส เหมาะสำหรับการทดสอบเท่านั้น ควรกำหนด hostname, TLS Secret และเปิด HTTPS redirect ก่อนใช้งานจริง

External slave ใช้ resource ชนิด `Endpoints` แบบ legacy ซึ่งอาจมีข้อจำกัดใน Kubernetes รุ่นใหม่ ควรตรวจสอบ compatibility ของ cluster และพิจารณาใช้ EndpointSlice หรือกลไกที่รองรับในสภาพแวดล้อมเป้าหมาย

## การติดตั้ง

ตรวจสอบ context ก่อนเริ่ม:

```bash
kubectl config current-context
kubectl cluster-info
```

ติดตั้งตามลำดับ dependency:

```bash
kubectl apply -f manifests/jenkins-master-ns.yaml
kubectl apply -f manifests/jenkins-master-pv.yaml
kubectl apply -f manifests/jenkins-master-pvc.yaml
kubectl apply -f manifests/jenkins-master-init-configmap.yaml
kubectl apply -f manifests/slave-cluster/jenkins-slave-ext-svc.yaml
kubectl apply -f manifests/slave-cluster/jenkins-slave-ext-endpoint.yaml
kubectl apply -f manifests/jenkins-master-svc.yaml
kubectl apply -f manifests/jenkins-master-deploy.yaml
kubectl apply -f manifests/jenkins-master-ingress.yaml
kubectl apply -f manifests/jenkins-master-network-policy.yaml
```

หรือใช้คำสั่งเดียวหลังจากแก้ไข manifest และตรวจสอบเรียบร้อยแล้ว:

```bash
kubectl apply -f manifests/jenkins-master-ns.yaml \
  -f manifests/jenkins-master-pv.yaml \
  -f manifests/jenkins-master-pvc.yaml \
  -f manifests/jenkins-master-init-configmap.yaml \
  -f manifests/slave-cluster/jenkins-slave-ext-svc.yaml \
  -f manifests/slave-cluster/jenkins-slave-ext-endpoint.yaml \
  -f manifests/jenkins-master-svc.yaml \
  -f manifests/jenkins-master-deploy.yaml \
  -f manifests/jenkins-master-ingress.yaml \
  -f manifests/jenkins-master-network-policy.yaml
```

## การเข้าใช้งาน

### Jenkins web UI

Ingress ใช้ path prefix:

```text
http://<INGRESS-IP>/chakorn/jenkins
```

ตรวจสอบ address ของ Ingress ด้วย:

```bash
kubectl get ingress jenkins-master -n jenkins-q
```

หากยังไม่มี Ingress หรืออยู่ระหว่างทดสอบ สามารถทำ port-forward ได้:

```bash
kubectl port-forward -n jenkins-q svc/jenkins-web 8080:80
```

จากนั้นเปิด `http://localhost:8080/chakorn/jenkins`

Groovy script สร้างผู้ใช้ `admin` โดยปัจจุบันมีรหัสผ่าน hard-code อยู่ใน ConfigMap ที่ [jenkins-master-init-configmap.yaml](manifests/jenkins-master-init-configmap.yaml) ควรย้ายรหัสผ่านไปใช้ Secret และ rotate รหัสผ่านก่อนใช้งานจริง ห้ามเผยแพร่ credential ใน README หรือ Git

### Jenkins agent ผ่าน JNLP

ค้นหา IP ของ node และเชื่อมต่อไปยัง NodePort:

```bash
kubectl get nodes -o wide
kubectl get svc jenkins-jnlp -n jenkins-q
```

ปลายทางสำหรับ agent คือ:

```text
<NODE-IP>:30001
```

## การตรวจสอบสถานะและแก้ปัญหา

```bash
kubectl get pv jenkins-home-pv
kubectl get pvc -n jenkins-q
kubectl get deploy,pod -n jenkins-q -o wide
kubectl get svc,ingress,endpoints -n jenkins-q
kubectl get events -n jenkins-q --sort-by=.lastTimestamp
```

ดู log ของ Jenkins และ init container:

```bash
kubectl logs deployment/jenkins -n jenkins-q -c jenkins
kubectl logs deployment/jenkins -n jenkins-q -c init-jenkins
```

อาการที่พบบ่อย:

- Pod ค้าง `Pending`: ตรวจสอบชื่อ PVC, สถานะ PV/PVC และ directory ของ `hostPath`
- init container ติดตั้ง plugins ไม่สำเร็จ: ตรวจสอบ network, ค่า `JENKINS_UC` และ version ของ plugins
- เข้า web UI ไม่ได้: ตรวจสอบ NGINX Ingress Controller, path `/chakorn/jenkins` และ service endpoints
- agent เชื่อมต่อไม่ได้: ตรวจสอบ NodePort `30001`, firewall และ JNLP port `50000`
- external slave ใช้งานไม่ได้: ตรวจสอบ IP, port `6443` และ Endpoints ด้วย `kubectl get endpoints jenkins-slave-agent -n jenkins-q`

## การลบทรัพยากร

ลบทรัพยากรใน namespace ก่อน แล้วจึงลบ storage และ namespace:

```bash
kubectl delete -f manifests/jenkins-master-ingress.yaml
kubectl delete -f manifests/jenkins-master-deploy.yaml
kubectl delete -f manifests/jenkins-master-svc.yaml
kubectl delete -f manifests/slave-cluster/jenkins-slave-ext-endpoint.yaml
kubectl delete -f manifests/slave-cluster/jenkins-slave-ext-svc.yaml
kubectl delete -f manifests/jenkins-master-init-configmap.yaml
kubectl delete -f manifests/jenkins-master-network-policy.yaml
kubectl delete -f manifests/jenkins-master-pvc.yaml
kubectl delete -f manifests/jenkins-master-pv.yaml
kubectl delete -f manifests/jenkins-master-ns.yaml
```

การลบ PV object โดยทั่วไปไม่ได้ลบไฟล์ใน `hostPath` แต่การลบหรือเขียนทับ directory `/jenkins-data/jenkins-Q-home` บน node จะทำให้ข้อมูล Jenkins หาย ควรสำรองข้อมูลก่อนดำเนินการ

## ข้อควรระวังด้านความปลอดภัยและการใช้งานจริง

- เปลี่ยนรหัสผ่านเริ่มต้นและย้าย credential ไปใช้ Kubernetes Secret หรือระบบจัดการ secret
- ระบุ Jenkins image และ plugin version ให้เหมาะกับการควบคุมการเปลี่ยนแปลง
- ใช้ TLS กับ Ingress และกำหนด hostname ที่ชัดเจน
- เปลี่ยน `hostPath` เป็น storage class หรือ persistent storage ที่เหมาะกับ cluster
- ทดสอบ compatibility ของ plugin versions กับ Jenkins image ก่อนอัปเกรด