# 🌩️ Troubleshooting Apache CloudStack 

> **লেখক**: Sumon (Cloud Engineer)  
> **ভার্সন**: Apache CloudStack 4.20+  
> **টার্গেট**: হোম ল্যাব / স্মল ডেভেলপমেন্ট এনভায়রনমেন্ট  
> **ভাষা**: বাংলা + ইংরেজি টেকনিক্যাল টার্মস  

এই গাইডটি Apache CloudStack-এ সাধারণত যেসব সমস্যা হয়ে থাকে — তার ধাপে ধাপে সমাধান নিয়ে লেখা।  
বিশেষ করে **হোম ল্যাব ইউজারদের** জন্য অত্যন্ত কার্যকর।

---

## 📌 সূচিপত্র

1. [সমস্যা #1: "No destination found for a deployment" — VM Create হয় না](#-সমস্যা-1-no-destination-found-for-a-deployment--vm-create-হয়-না)
2. [সমস্যা #2: পাওয়ার অফ/অন হলে VM অটোমেটিক স্টার্ট হয় না](#-সমস্যা-2-পাওয়ার-অফঅন-হলে-vm-অটোমেটিক-স্টার্ট-হয়-না)
3. [সমস্যা #3: Host "Disconnected" দেখাচ্ছে যদিও Agent চলছে](#-সমস্যা-3-host-disconnected-দেখাচ্ছে-যদিও-agent-চলছে)
4. [সমস্যা #4: System VM (VR/SSVM) Running হচ্ছে না](#-সমস্যা-4-system-vm-vrssvm-running-হচ্ছে-না)
5. [সমস্যা #5: Template/ISO ডাউনলোড হয় না — "Failed to install" এরর](#-সমস্যা-5-templateiso-ডাউনলোড-হয়-না--failed-to-install-এরর)
6. [সাধারণ টিপস & বেস্ট প্র্যাকটিস](#-সাধারণ-টিপস--বেস্ট-প্র্যাকটিস)

---

## ❌ সমস্যা #1: "No destination found for a deployment" — VM Create হয় না

### 🔍 লক্ষণ
- UI-তে এরর:  
  `Error: No destination found for a deployment for VM instance {"id":27,...}`  
- সবকিছু গ্রিন (Host Up, Storage Ready) — কিন্তু VM তৈরি হয় না।

### 🧠 কারণ
CloudStack-এর **Deployment Planner** কোনো উপযুক্ত Host খুঁজে পাচ্ছে না।  
সাধারণত এটি হয়:
- ভুল **Deployment Planner Order** সেট থাকায়
- Host-এ **পর্যাপ্ত RAM/CPU** না থাকায়
- **Service Offering** এবং **Host/Template Tags** মেলে না

### ✅ সমাধান (ধাপে ধাপে)

#### Step 1: `deployment.planners.order` ঠিক করুন
1. UI → **Configuration** → সার্চ করুন: `deployment.planners.order`
2. মান পরিবর্তন করুন:  
   ```
   FirstFitPlanner
   ```  
   *(অন্য সব planner মুছে দিন — বিশেষ করে `UserDispersingPlanner`)*
3. Save করুন।

#### Step 2: Management Server রিস্টার্ট করুন
```bash
sudo systemctl restart cloudstack-management
```

#### Step 3: Host Reconnect করুন
- UI → Infrastructure > Hosts > [Your Host] > **Reconnect**

#### Step 4: Tiny VM দিয়ে টেস্ট করুন
- 1 vCPU, 1 GB RAM দিয়ে নতুন Service Offering তৈরি করুন
- সেটি ব্যবহার করে VM create করুন

> ✅ **কেন কাজ করে?**  
> `FirstFitPlanner` শুধু “যেকোনো উপযুক্ত Host” খুঁজে — strategy ছাড়া।  
> হোম ল্যাবে এটিই সেরা।

---

## ⚡ সমস্যা #2: পাওয়ার অফ/অন হলে VM অটোমেটিক স্টার্ট হয় না

### 🔍 লক্ষণ
- Host/Management Server রিবুট হলে VM-গুলো **Stopped** অবস্থায় থাকে
- ম্যানুয়ালি Start করতে হয়

### 🧠 কারণ
CloudStack **ডিফল্টে VM auto-start বন্ধ রাখে** (নিরাপত্তার জন্য)।  
CloudStack 4.19+ থেকে UI-তে `vm.restart.on.reboot` অপশন আর নেই।

### ✅ সমাধান (ডাটাবেস এডিট করে)

#### Step 1: CloudStack ডাটাবেসে লগইন
```bash
mysql -u cloud -p cloud
# পাসওয়ার্ড: /etc/cloudstack/management/db.properties ফাইলে
```

#### Step 2: সেটিং চেক/Insert করুন
```sql
-- চেক করুন
SELECT name, value FROM configuration WHERE name = 'vm.restart.on.reboot';

-- যদি না থাকে, তাহলে insert করুন
INSERT INTO configuration (category, instance, component, name, value, description, default_value, is_dynamic)
VALUES ('Advanced', 'DEFAULT', 'management-server', 'vm.restart.on.reboot', 'true',
'If true, VMs that were running before host reboot will be restarted after agent reconnects.', 'false', 0);
```

#### Step 3: Management Server রিস্টার্ট
```bash
sudo systemctl restart cloudstack-management
```

#### Step 4: Host Agent Auto-Start নিশ্চিত করুন
```bash
sudo systemctl enable cloudstack-agent
sudo systemctl start cloudstack-agent
```

> ✅ **এখন থেকে**: Host reboot → VM auto-start!

---

## 🛑 সমস্যা #3: Host "Disconnected" দেখাচ্ছে যদিও Agent চলছে

### 🔍 লক্ষণ
- UI-তে Host Status = **Disconnected**
- কিন্তু Host-এ `systemctl status cloudstack-agent` = active

### ✅ সমাধান
1. **ফায়ারওয়াল চেক করুন**: Host ↔ Management Server-এর মধ্যে **পোর্ট 8250** open আছে কিনা
   ```bash
   # Host থেকে টেস্ট:
   telnet <management-server-ip> 8250
   ```
2. **Agent কনফিগ চেক করুন**:  
   `/etc/cloudstack/agent/agent.properties` → `host=` ঠিক আছে কিনা
3. **Agent রিস্টার্ট করুন**:
   ```bash
   sudo systemctl restart cloudstack-agent
   ```

---

## 🖥️ সমস্যা #4: System VM (VR/SSVM) Running হচ্ছে না

### 🔍 লক্ষণ
- VR (Virtual Router) বা SSVM (Secondary Storage VM) **Starting → Stopped** লুপে আটকে
- Console Proxy কাজ করে না

### ✅ সমাধান
1. **Secondary Storage চেক করুন**:  
   - NFS mount ঠিক আছে কিনা?  
   - `/mnt/secondary` এ **write permission** আছে কিনা?
2. **System VM Template চেক করুন**:  
   - UI → Templates → System VM Template → **Downloaded & Ready**?
3. **Agent লগ চেক করুন**:
   ```bash
   tail -f /var/log/cloudstack/agent/agent.log
   ```
4. **SSVM Destroy & Recreate**:  
   - UI → Infrastructure > System VMs → SSVM → **Destroy**  
   - CloudStack অটোমেটিক নতুনটি তৈরি করবে

---

## 📥 সমস্যা #5: Template/ISO ডাউনলোড হয় না — "Failed to install" এরর

### 🔍 লক্ষণ
- Template/ISO status = **"Downloading" → "Failed to install"**
- Secondary Storage-এ ফাইল আসে না

### ✅ সমাধান
1. **Secondary Storage URL চেক করুন**:  
   - UI → Infrastructure > Secondary Storage → **NFS path** ঠিক আছে কিনা?
2. **Management Server থেকে Secondary Storage mount টেস্ট করুন**:
   ```bash
   sudo mkdir /tmp/test
   sudo mount -t nfs <nfs-server>:/path /tmp/test
   ```
3. **ফায়ারওয়াল**: NFS (2049), RPC (111) পোর্ট open আছে কিনা
4. **Template URL চেক করুন**:  
   - HTTP/HTTPS URL accessible? (wget দিয়ে টেস্ট করুন)

---

## 💡 সাধারণ টিপস & বেস্ট প্র্যাকটিস

| টিপ | ব্যাখ্যা |
|-----|--------|
| ✅ **হোম ল্যাবে শুধু `FirstFitPlanner` ব্যবহার করুন** | অন্য planner গুলো multi-host environment-এর জন্য |
| ✅ **Host Agent সবসময় auto-start হোক** | `systemctl enable cloudstack-agent` |
| ✅ **Secondary Storage আলাদা ডিস্কে রাখুন** | I/O conflict এড়াতে |
| ✅ **লগ চেক করুন প্রথমে**: `management-server.log` | সবচেয়ে বেশি তথ্য পাবেন এখানে |
| ✅ **ছোট VM দিয়ে টেস্ট করুন** | রিসোর্স ইস্যু এড়াতে |

---

## 🌱 শেষ কথা

> CloudStack শুধু একটি প্ল্যাটফর্ম নয় — এটি আপনার ডিজিটাল নিজের হাতে গড়া আকাশ।  
> প্রতিটি এরর, প্রতিটি রিবুট — এগুলোই আপনাকে সেই আকাশের মালিক বানায়।

---

## 📚 রেফারেন্স
- [Apache CloudStack Official Docs](https://docs.cloudstack.apache.org/)
- [CloudStack Troubleshooting Guide (GitHub)](https://github.com/apache/cloudstack)
- [CloudStack 4.20 Release Notes](https://cloudstack.apache.org/releases/4.20.1.0/)

---

> 🌟 **সত্যিকারের ক্লাউড ইঞ্জিনিয়ারিং শুরু হয় তখনই, যখন আপনি এরর মেসেজগুলোকে ভয় পান না — বরং সেগুলোকে বুঝে নেন আপনার বন্ধু হিসাবে।**
