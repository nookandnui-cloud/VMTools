# 🖥️ MFEC VMware Tools — RVTools Report

정적 HTML 보고서를 OpenShift 에 배포하는 프로젝트입니다.

---

## 📦 Architecture

```
┌─────────────────────────────────────────────────────┐
│                   OpenShift Cluster                  │
│  ┌──────────────┐    ┌──────────────┐               │
│  │ ConfigMap    │    │   Pod        │               │
│  │ rvtools-     │───▶│ rvtools-     │               │
│  │ report13-cm  │    │ report13     │               │
│  │ (index.html) │    │ busybox:1.36 │               │
│  └──────────────┘    │ httpd :8080  │               │
│                      └──────┬───────┘               │
│                             │                        │
│                      ┌──────┴───────┐               │
│                      │ Service      │               │
│                      │ rvtools-     │               │
│                      │ report13-svc │               │
│                      └──────┬───────┘               │
│                             │                        │
│              ┌──────────────┴──────────────┐         │
│              │                             │         │
│    ┌────────┴────────┐          ┌─────────┴────────┐
│    │ Route (OCP)     │          │ cloudflared      │
│    │ apps.ailab...   │          │ tunnel → svc:8080│
│    │ edge TLS        │          │ cloudflare/latest │
│    └─────────────────┘          └──────────────────┘
└─────────────────────────────────────────────────────┘
```

---

## 🚀 Deploy

### ขั้นตอนครั้งแรก

```bash
# 1. สร้าง Namespace
oc new-project mfec-vm-tools

# 2. สร้าง ConfigMap จากไฟล์ HTML
oc create configmap rvtools-report13-cm \
  --from-file=index.html=rvtools-report13.html

# 3. Deploy Pod + Service + Route + cloudflared
oc apply -f rvtools-report13-deploy.yaml
```

### ตรวจสอบ

```bash
oc get pod        # ทั้ง 2 pods Running
oc get route      # ดู URL
curl -sk https://<route-host>/   # ทดสอบ response 200
```

### URL

| ทางเข้า | URL |
|---|---|
| OpenShift Route | `https://rvtools-report13-route-mfec-vm-tools.apps.ailab.mfec.co.th/` |
| Cloudflare Tunnel | (ผ่าน domain Cloudflare) |

---

## 🔄 อัปเดต HTML

เมื่อมีการแก้ไข `rvtools-report13.html`:

```bash
cd /path/to/project

# 1. อัปเดต ConfigMap (Pod รับไฟล์ใหม่โดยอัตโนมัติ ภายใน ~1 นาที)
oc project mfec-vm-tools
oc create configmap rvtools-report13-cm \
  --from-file=index.html=rvtools-report13.html \
  --dry-run=client -o yaml | oc replace -f -

# 2. (ทางเลือก) หากต้องการให้ใช้ทันที — restart Pod
oc delete pod rvtools-report13

# 3. อัปเดต GitHub
git add rvtools-report13.html
git commit -m "อัปเดตรายงาน rvtools"
git push origin main
```

> **หมายเหตุ:** ConfigMap volume mount จะ sync ไฟล์ใหม่โดยอัตโนมัติ ไม่ต้อง restart Pod เว้นแต่ต้องการ immediate change

---

## 📁 โครงสร้างไฟล์

```
VMTools/
├── rvtools-report13.html      # รายงาน HTML (static)
└── rvtools-report13-deploy.yaml  # Kubernetes manifests
```

### `rvtools-report13-deploy.yaml` มี 4 resources:

| Resource | ชื่อ | หน้าที่ |
|---|---|---|
| `ConfigMap` | `rvtools-report13-cm` | เก็บ `index.html` |
| `Pod` | `rvtools-report13` | busybox httpd เซิร์ฟเวอร์ report |
| `Service` | `rvtools-report13-svc` | ClusterIP เข้าถึง Pod |
| `Route` | `rvtools-report13-route` | OpenShift edge TLS route |
| `Pod` | `cloudflared` | Cloudflare tunnel |

---

## 🗑️ ลบ deployment ทั้งหมด

```bash
oc delete pod rvtools-report13 cloudflared
oc delete svc rvtools-report13-svc
oc delete route rvtools-report13-route
oc delete configmap rvtools-report13-cm
```

---

## 📝 หมายเหตุ

- **Namespace:** `mfec-vm-tools` (lowercase, RFC 1123)
- **Image:** `busybox:1.36` (ไม่มี USER directive — ใช้ได้กับ CRI-O ConfigMap volume)
- **SecurityContext:** `cloudflared` pod มี `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `runAsNonRoot: true`, `seccompProfile: RuntimeDefault`
- **ConfigMap limit:** 1MB ต่อ ConfigMap — ไฟล์ report ปัจจุบัน ~217KB ใช้งานได้
