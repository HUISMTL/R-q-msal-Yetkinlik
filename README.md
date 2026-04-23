# Rəqəmsal Yetkinlik Platforması — Deploy Təlimatı

## Tələblər
- Ubuntu 22.04 VPS (minimum 2 CPU, 4GB RAM, 40GB SSD)
- Docker + Docker Compose quraşdırılmış olmalıdır
- Domain (ixtiyari, lakin SSL üçün tövsiyə olunur)

---

## 1. Serveri hazırlayın

```bash
# Docker quraşdırın
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Docker Compose yoxlayın
docker compose version
```

---

## 2. Layihəni serverə yükləyin

```bash
# Git ilə
git clone https://your-repo/reqemsal-yetkinlik.git
cd reqemsal-yetkinlik

# YA DA: faylları scp ilə köçürün
scp -r ./reqemsal-yetkinlik user@YOUR_SERVER_IP:/home/user/
```

---

## 3. Environment faylını yaradın

```bash
cp backend/.env.example .env

# .env faylını redaktə edin
nano .env
```

**.env** faylında dəyişdirin:
```env
DB_PASSWORD=OZUNUZUN_GUCLU_SIFRESI
JWT_SECRET=minimum_32_simvol_olan_guclu_acari_buraya_yazin
FRONTEND_URL=https://sizin-domain.az
ADMIN_PASSWORD=Admin@2026!
```

---

## 4. SSL Sertifikatı (Let's Encrypt)

```bash
mkdir -p nginx/ssl

# Certbot ilə (domain varsa)
sudo apt install certbot -y
sudo certbot certonly --standalone -d sizin-domain.az

# Sertifikatları kopyalayın
sudo cp /etc/letsencrypt/live/sizin-domain.az/fullchain.pem nginx/ssl/cert.pem
sudo cp /etc/letsencrypt/live/sizin-domain.az/privkey.pem  nginx/ssl/key.pem

# YA DA: self-signed (test üçün)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/key.pem \
  -out nginx/ssl/cert.pem \
  -subj "/C=AZ/ST=Baki/L=Baki/O=IDDA/CN=localhost"
```

---

## 5. Platformanı işə salın

```bash
# Bütün xidmətləri qur və başlat
docker compose up -d --build

# Logları izləyin
docker compose logs -f api

# İlk dəfə seed run edin (yalnız bir dəfə!)
docker compose exec api node prisma/seed.js
```

---

## 6. Yoxlayın

```bash
# API sağlamlığı
curl http://localhost/api/health

# Brauzerda açın
# http://YOUR_SERVER_IP  (HTTP)
# https://sizin-domain.az (HTTPS, domain varsa)
```

---

## İlkin giriş məlumatları

| İstifadəçi adı | Şifrə | Rol |
|---|---|---|
| `super.admin` | `Admin@2026!` (`.env`-dən) | Super Admin |
| `miqrasiya.user` | `Test@2026!` | Qurum (Dövlət Miqrasiya X.) |
| `ekspert.user` | `Test@2026!` | Ekspert |

> ⚠️ **İlk girişdən sonra şifrəni dəyişin!**

---

## Faydalı əmrlər

```bash
# Xidmətləri yenidən başlat
docker compose restart api

# Logları izlə
docker compose logs -f

# DB-yə qoşul
docker compose exec db psql -U reqemsal_user -d reqemsal_yetkinlik

# Yeni versiya deploy et
git pull
docker compose up -d --build api frontend

# Backup al
docker compose exec db pg_dump -U reqemsal_user reqemsal_yetkinlik > backup_$(date +%Y%m%d).sql
```

---

## Layihə Strukturu

```
reqemsal-yetkinlik/
├── backend/
│   ├── prisma/
│   │   ├── schema.prisma      # DB sxemi
│   │   └── seed.js            # İlkin data
│   ├── src/
│   │   ├── index.js           # Express app
│   │   ├── middleware/
│   │   │   ├── auth.js        # JWT + rol yoxlaması
│   │   │   └── errorHandler.js
│   │   ├── routes/
│   │   │   ├── auth.js        # Login/logout
│   │   │   ├── users.js       # İstifadəçi CRUD
│   │   │   ├── organizations.js
│   │   │   ├── assessments.js # Əsas iş məntiqi
│   │   │   ├── criteria.js
│   │   │   ├── reports.js     # Statistika
│   │   │   └── audit.js
│   │   └── controllers/
│   │       └── scoreCalculator.js  # D1–D12 bal məntiqi
│   ├── Dockerfile
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   │   ├── client.ts      # Axios instance
│   │   │   └── services.ts    # API funksiyaları
│   │   ├── hooks/
│   │   │   └── useAuth.ts     # Zustand auth store
│   │   ├── components/
│   │   │   └── Layout.tsx     # Topbar + sidebar
│   │   ├── pages/
│   │   │   ├── LoginPage.tsx
│   │   │   ├── DashboardPage.tsx
│   │   │   ├── QurumDashboard.tsx
│   │   │   ├── AssessmentPage.tsx  # Adaptiv form
│   │   │   ├── AssessmentsAdmin.tsx
│   │   │   ├── ReviewPage.tsx
│   │   │   ├── ReportsPage.tsx
│   │   │   ├── UsersPage.tsx
│   │   │   └── AuditPage.tsx
│   │   ├── App.tsx            # Router + route guards
│   │   └── main.tsx
│   ├── Dockerfile
│   └── package.json
├── nginx/
│   ├── default.conf           # Reverse proxy + SSL
│   └── ssl/                   # Sertifikatlar
├── docker-compose.yml
└── .env                       # Əsas konfiqurasiya
```

---

## API Endpoints

| Method | Endpoint | Açıqlama | Rol |
|---|---|---|---|
| POST | `/api/auth/login` | Giriş | Hamı |
| GET | `/api/auth/me` | Cari istifadəçi | Auth |
| GET | `/api/assessments` | Siyahı | Auth |
| POST | `/api/assessments` | Yarat | Auth |
| POST | `/api/assessments/:id/answers` | Cavab saxla | Qurum |
| POST | `/api/assessments/:id/submit` | Göndər | Qurum |
| POST | `/api/assessments/:id/review` | Nəzərə al | Reviewer |
| PUT | `/api/assessments/:id/scores` | Bal dəyiş | Reviewer |
| POST | `/api/assessments/:id/approve` | Təsdiq | Reviewer |
| POST | `/api/assessments/:id/reject` | Rədd et | Reviewer |
| GET | `/api/reports/summary` | Xülasə | Analyst |
| GET | `/api/reports/heatmap` | Heat map | Analyst |
| GET | `/api/users` | İstifadəçilər | Admin |
| POST | `/api/users` | Yarat | Admin |
| GET | `/api/audit` | Log | Admin |
