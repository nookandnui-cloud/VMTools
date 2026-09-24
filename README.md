# 🖥️ MFEC VMware Tools — RVTools Report

รายงาน HTML แบบ static deploy บน OpenShift บน MFEC Cloud

---

## 📐 Architecture

```
                   OpenShift Cluster (mfec-vm-tools)
┌─────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐    ┌──────────────────┐               │
│  │ ConfigMap        │    │ Pod              │               │
│  │ rvtools-report-  │───▶│ rvtools-report   │               │
│  │ cm (index.html)  │    │ busybox:1.36     │               │
│  └──────────────────┘    │ httpd :8080      │               │
│                          └──────┬───────────┘               │
│                                 │                            │
│                          ┌──────┴───────────┐               │
│                          │ Service         │               │
│                          │ rvtools-report  │               │
│                          │ -svc :8080      │               │
│                          └──────┬───────────┘               │
│          ┌─────────────────────┼─────────────────────┐       │
│          │                     │                     │       │
│  ┌───────┴───────────┐   ┌─────┴──────────────┐     │       │
│  │ Route (OCP)       │   │ cloudflared        │     │       │
│  │ apps.ailab...     │   │ tunnel → svc:8080  │     │       │
│  │ edge TLS          │   │ cloudflare:latest  │     │       │
│  └───────────────────┘   └────────────────────┘     │       │
└──────────────────────────────────────────────────────┘       │
                                                              │
                                          Cloudflare Tunnel ──┘
                                          (เข้าถึงผ่าน domain CF)
```

---

## 🚀 Deploy

### ขั้นตอนครั้งแรก

```bash
# 1. สร้าง Namespace
oc new-project mfec-vm-tools

# 2. สร้าง ConfigMap จากไฟล์ HTML
oc create configmap rvtools-report-cm \
  --from-file=index.html=rvtools-report.html

# 3. Deploy Pod + Service + Route + cloudflared
oc apply -f rvtools-report-deploy.yaml
```

### ตรวจสอบ

```bash
oc get pod        # ทั้ง 2 pods Running
oc get route      # ดู URL
curl -sk https://<route-host>/   # ทดสอบ response 200
```

### เข้าถึงแอป

| ทางเข้า | URL / วิธี |
|---|---|
| **Cloudflare Tunnel** | เข้าผ่าน domain Cloudflare ที่เชื่อมกับ tunnel |
| OpenShift Route | `https://rvtools-report-route-mfec-vm-tools.apps.ailab.mfec.co.th/` |

> Cloudflare tunnel คือทางเข้าหลัก เข้าถึงผ่าน domain ที่ตั้งค่าใน Cloudflare dashboard

---

## 🔄 อัปเดต HTML

เมื่อมีการแก้ไข `rvtools-report.html`:

```bash
cd /path/to/project

# 1. อัปเดต ConfigMap (Pod รับไฟล์ใหม่โดยอัตโนมัติ ภายใน ~1 นาที)
oc project mfec-vm-tools
oc create configmap rvtools-report-cm \
  --from-file=index.html=rvtools-report.html \
  --dry-run=client -o yaml | oc replace -f -

# 2. (ทางเลือก) หากต้องการให้ใช้ทันที — restart Pod
oc delete pod rvtools-report

# 3. อัปเดต GitHub
git add rvtools-report.html
git commit -m "อัปเดตรายงาน"
git push origin main
```

---

## 📁 ไฟล์ในโปรเจกต์

```
VMTools/
├── rvtools-report.html          # รายงาน HTML (static)
├── rvtools-report-deploy.yaml   # Kubernetes manifests
└── README.md                    # เอกสารนี้
```

### `rvtools-report-deploy.yaml` มี 4 resources:

| Resource | ชื่อ | หน้าที่ |
|---|---|---|
| `ConfigMap` | `rvtools-report-cm` | เก็บ `index.html` |
| `Pod` | `rvtools-report` | busybox httpd เซิร์ฟเวอร์ report |
| `Service` | `rvtools-report-svc` | ClusterIP เข้าถึง Pod |
| `Route` | `rvtools-report-route` | OpenShift edge TLS route |
| `Pod` | `cloudflared` | Cloudflare tunnel → svc:8080 |

---

## 🗑️ ลบ deployment ทั้งหมด

```bash
oc delete pod rvtools-report cloudflared
oc delete svc rvtools-report-svc
oc delete route rvtools-report-route
oc delete configmap rvtools-report-cm
```

---

## 📝 หมายเหตุ

- **Namespace:** `mfec-vm-tools`
- **Image:** `busybox:1.36` (ไม่มี USER directive — ใช้ได้กับ CRI-O + ConfigMap volume)
- **SecurityContext (cloudflared):** `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `runAsNonRoot: true`, `seccompProfile: RuntimeDefault`
- **ConfigMap limit:** 1MB ต่อ ConfigMap — ไฟล์ report ปัจจุบัน ~217KB ใช้งานได้
- **URL หลัก:** เข้าถึงผ่าน Cloudflare tunnel domain (ตั้งค่าใน CF dashboard)
