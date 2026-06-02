# 📡 إعداد بث HLS آمن بمعدل بت متكيف (ABR)

> دليل شامل لبناء خادم بث مباشر احترافي باستخدام **Nginx + RTMP + FFmpeg** على نظام **Ubuntu 20.04**

[![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Nginx](https://img.shields.io/badge/Nginx-RTMP-009639?logo=nginx&logoColor=white)](https://nginx.org/)
[![FFmpeg](https://img.shields.io/badge/FFmpeg-Transcoding-007808?logo=ffmpeg&logoColor=white)](https://ffmpeg.org/)
[![HLS](https://img.shields.io/badge/HLS-Adaptive_Bitrate-FF6B6B)](https://developer.apple.com/streaming/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 📖 جدول المحتويات

- [نظرة عامة](#-نظرة-عامة)
- [المميزات](#-المميزات)
- [المتطلبات](#-المتطلبات)
- [المعمارية](#-المعمارية)
- [خطوات الإعداد](#-خطوات-الإعداد)
  - [1. تجهيز الخادم](#1-تجهيز-الخادم)
  - [2. تثبيت Nginx وRTMP](#2-تثبيت-nginx-وrtmp)
  - [3. إنشاء المجلدات والصلاحيات](#3-إنشاء-المجلدات-والصلاحيات)
  - [4. إعداد ملفات Cross-domain](#4-إعداد-ملفات-cross-domain)
  - [5. إعداد Nginx الأساسي](#5-إعداد-nginx-الأساسي)
  - [6. إصدار شهادة SSL](#6-إصدار-شهادة-ssl)
  - [7. تهيئة nginx.conf لتفعيل RTMP](#7-تهيئة-nginxconf-لتفعيل-rtmp)
  - [8. تثبيت مشغّل Video.js](#8-تثبيت-مشغل-videojs)
  - [9. تفعيل ABR](#9-تفعيل-abr)
- [الاختبار](#-الاختبار)
- [استكشاف الأخطاء](#-استكشاف-الأخطاء)
- [المساهمة](#-المساهمة)
- [الترخيص](#-الترخيص)

---

## 🎯 نظرة عامة

هذا المشروع يشرح خطوة بخطوة كيفية إعداد خادم بث مباشر (Live Streaming) آمن باستخدام **Nginx** مع وحدة **RTMP** وأداة **FFmpeg**. سيُمكّنك هذا الإعداد من بث الفيديو بعدة جودات (**Adaptive Bitrate Streaming**) بحيث يختار المشغّل تلقائياً الجودة المناسبة لسرعة اتصال المشاهد.

---

## ✨ المميزات

- ✅ **استقبال البث عبر RTMP** من برامج مثل OBS Studio
- ✅ **تحويل تلقائي إلى 5 جودات** (480p, 720p, 960p, 1280p, المصدر)
- ✅ **توزيع البث عبر HLS و DASH** للتوافق مع جميع الأجهزة
- ✅ **شهادة SSL مجانية** من Let's Encrypt
- ✅ **مشغّل ويب جاهز** مبني على Video.js
- ✅ **تسجيل البث تلقائياً** بصيغة FLV/MP4
- ✅ **لوحة إحصائيات** لمراقبة البث المباشر
- ✅ **تشفير HLS اختياري** بـ AES-128

---

## 🛠 المتطلبات

| المتطلب | التفاصيل |
|---------|----------|
| نظام التشغيل | Ubuntu 20.04 LTS |
| المعالج | 4 أنوية فأكثر |
| الذاكرة | 4 GB RAM على الأقل |
| التخزين | 20 GB متاحة |
| النطاق | نطاق مسجّل ومُوجَّه بسجل A للخادم |
| المنافذ | 80, 443, 1935 مفتوحة |

---

## 🏗 المعمارية

```
┌─────────────┐      RTMP       ┌──────────────┐     FFmpeg      ┌──────────┐
│ OBS Studio  │ ──────────────> │  Nginx /live │ ──────────────> │ 5 جودات  │
│  (Encoder)  │   port 1935     │              │   transcoding   │  مختلفة  │
└─────────────┘                 └──────┬───────┘                 └────┬─────┘
                                       │                              │
                                       │ push                         │
                                       ▼                              ▼
                                ┌──────────────┐              ┌───────────────┐
                                │ Nginx /hls   │              │ HLS Variants  │
                                │ Nginx /dash  │              │ .m3u8 / .ts   │
                                └──────┬───────┘              └───────┬───────┘
                                       │                              │
                                       │           HTTPS              │
                                       ▼                              ▼
                                ┌─────────────────────────────────────────┐
                                │       Video.js Player (المتصفح)         │
                                └─────────────────────────────────────────┘
```

---

## 🚀 خطوات الإعداد

> **ملاحظة:** استبدل `YOURDOMAIN` بنطاقك الفعلي (مثال: `stream.example.com`) في جميع الأوامر التالية.

### 1. تجهيز الخادم

تحديث النظام:

```bash
sudo apt update && sudo apt upgrade -y
```

تثبيت الحزم والمكتبات الأساسية:

```bash
sudo apt-get install wget unzip software-properties-common dpkg-dev git \
  make gcc automake build-essential zlib1g-dev libpcre3 libpcre3-dev \
  libssl-dev libxslt1-dev libxml2-dev libgd-dev libgeoip-dev \
  libgoogle-perftools-dev libperl-dev pkg-config autotools-dev gpac \
  ffmpeg mediainfo mencoder lame libvorbisenc2 libvorbisfile3 \
  libx264-dev libvo-aacenc-dev libmp3lame-dev libopus-dev -y
```

---

### 2. تثبيت Nginx وRTMP

```bash
sudo apt install nginx -y
sudo apt install libnginx-mod-rtmp python3-certbot-nginx -y
```

---

### 3. إنشاء المجلدات والصلاحيات

```bash
# مجلد الموقع
sudo mkdir -p /var/www/yourdomain/web/js/videojs

# مجلدات البث
sudo mkdir -p /var/livestream/hls \
              /var/livestream/dash \
              /var/livestream/recordings \
              /var/livestream/keys

# روابط رمزية
sudo ln -s /var/livestream/hls  /var/www/yourdomain/web/hls
sudo ln -s /var/livestream/dash /var/www/yourdomain/web/dash

# الملكية
sudo chown -R www-data:www-data /var/livestream /var/www/yourdomain
```

جلب وحدة nginx-rtmp-module:

```bash
cd /usr/src
sudo git clone https://github.com/arut/nginx-rtmp-module

sudo cp /usr/src/nginx-rtmp-module/stat.xsl /var/www/html/stat.xsl
sudo cp /usr/src/nginx-rtmp-module/stat.xsl /var/www/yourdomain/web/stat.xsl
```

---

### 4. إعداد ملفات Cross-domain

```bash
sudo nano /var/www/html/crossdomain.xml
```

محتوى الملف:

```xml
<?xml version="1.0"?>
<!DOCTYPE cross-domain-policy SYSTEM
  "http://www.adobe.com/xml/dtds/cross-domain-policy.dtd">
<cross-domain-policy>
  <allow-access-from domain="*"/>
</cross-domain-policy>
```

نسخ الملف وإنشاء صفحة index مؤقتة:

```bash
sudo cp /var/www/html/crossdomain.xml /var/www/yourdomain/web/crossdomain.xml
sudo cp /var/www/html/index.nginx-debian.html /var/www/yourdomain/web/index.html
```

---

### 5. إعداد Nginx الأساسي

```bash
sudo nano /etc/nginx/sites-available/yourdomain.conf
```

<details>
<summary>📄 اضغط لعرض محتوى الملف</summary>

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name YOURDOMAIN;            # ← غيّر هذا
    root /var/www/YOURDOMAIN/web;      # ← غيّر هذا

    index index.html index-nginx.html index.htm index.php;
    client_max_body_size 8192M;
    add_header Strict-Transport-Security "max-age=63072000;";
    add_header X-Frame-Options "DENY";

    location / {
        add_header Cache-Control no-cache;
        add_header Access-Control-Allow-Origin *;
        try_files $uri $uri/ =404;
    }

    location /crossdomain.xml {
        root /var/www/html;
        default_type text/xml;
        expires 24h;
    }

    location /control {
        rtmp_control all;
        add_header Access-Control-Allow-Origin * always;
    }

    location /stat {
        rtmp_stat all;
        rtmp_stat_stylesheet stat.xsl;
    }

    location /stat.xsl {
        root /var/www/YOURDOMAIN/web;
    }

    location ~ /\.ht {
        deny all;
    }

    location /hls {
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
        autoindex on;
        alias /var/livestream/hls;

        expires -1;
        add_header Cache-Control no-cache;
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Expose-Headers' 'Content-Length';

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }
    }

    location /dash {
        types {
            application/dash+xml mpd;
            video/mp4 mp4;
        }
        autoindex on;
        alias /var/livestream/dash;

        expires -1;
        add_header Cache-Control no-cache;
        add_header 'Access-Control-Allow-Origin' '*' always;
    }
}
```

</details>

تفعيل الموقع واختبار الإعداد:

```bash
sudo ln -s /etc/nginx/sites-available/YOURDOMAIN.conf \
          /etc/nginx/sites-enabled/YOURDOMAIN.conf

sudo nginx -t
sudo systemctl restart nginx
```

---

### 6. إصدار شهادة SSL

```bash
sudo certbot --nginx -d YOURDOMAIN
```

> ⚠️ تأكد أن نطاقك مُوجَّه بسجل A إلى عنوان IP الخاص بالخادم قبل تنفيذ هذا الأمر.

---

### 7. تهيئة nginx.conf لتفعيل RTMP

نسخ احتياطي وتنزيل ملف جاهز:

```bash
sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx-original.conf

sudo wget -O /etc/nginx/nginx.conf \
  https://raw.githubusercontent.com/ustoopia/Nginx-config-for-livestreams-ABS-HLS-ffmpeg-transc-/main/etc/nginx/nginx.conf
```

#### 📊 جدول الجودات الناتجة بعد تفعيل ABR

| اللاحقة | الدقّة | معدل البت | الاستخدام |
|---------|--------|-----------|-----------|
| `_low` | 480p | 256 kbps | جوّال / إنترنت ضعيف |
| `_mid` | 720p | 768 kbps | اتصال متوسط |
| `_high` | 960p | 1024 kbps | اتصال جيّد |
| `_higher` | 1280p | 1920 kbps | HD - إنترنت سريع |
| `_src` | الأصلية | الأصلي | أعلى جودة ممكنة |

<details>
<summary>📄 شرح أقسام nginx.conf</summary>

**قسم http {}**: إعدادات Nginx الأساسية (workers, gzip, logs).

**تطبيق /live**: نقطة استقبال البث على المنفذ 1935.

**تطبيق /hls**: ينتج ملفات `.m3u8` و `.ts`.

**تطبيق /dash**: ينتج ملفات MPEG-DASH (`.mpd`).

**تطبيق /recorder**: لتسجيل البث تلقائياً.

</details>

---

### 8. تثبيت مشغّل Video.js

```bash
sudo wget -O /var/www/YOURDOMAIN/web/js/videojs/latest.zip \
  https://github.com/videojs/video.js/releases/download/v7.11.4/video-js-7.11.4.zip

cd /var/www/YOURDOMAIN/web/js/videojs
sudo unzip latest.zip
```

إنشاء صفحة `videoplayer.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Live Stream</title>
    <link href="https://YOURDOMAIN/js/videojs/video-js.css" rel="stylesheet">
</head>
<body>
<center>
    <video-js id="live_stream" class="vjs-default-skin" controls
              preload="auto" width="1280" height="auto">
        <source src="https://YOURDOMAIN/hls/stream/index.m3u8"
                type="application/x-mpegURL">
    </video-js>

    <script src="https://YOURDOMAIN/js/videojs/video.js"></script>
    <script src="https://YOURDOMAIN/js/videojs/videojs-http-streaming.js"></script>
    <script>
        var player = videojs('live_stream');
    </script>
</center>
</body>
</html>
```

ضبط الصلاحيات:

```bash
sudo chown -R www-data:www-data /var/www/yourdomain/web /var/www/html
```

---

### 9. تفعيل ABR

```bash
sudo nano /etc/nginx/nginx.conf
```

**الخطوات:**

1. أزل `#` من أسطر `exec ffmpeg` التي تنتج الجودات الخمس.
2. ضع `#` أمام السطر: `push rtmp://localhost/hls;`
3. أزل `#` من أسطر `hls_variant` الخمسة داخل تطبيق `hls`.
4. احفظ وأعد التشغيل:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

إنشاء صفحة `abs.html` للمشاهدة:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Adaptive Live Stream</title>
    <link href="https://YOURDOMAIN/js/videojs/video-js.css" rel="stylesheet">
</head>
<body>
<center>
    <video-js id="live_stream" class="vjs-default-skin" controls
              preload="auto" width="1280" height="auto"
              poster="https://YOURDOMAIN/poster.jpg">
        <source src="https://YOURDOMAIN/hls/stream.m3u8"
                type="application/x-mpegURL">
    </video-js>

    <script src="https://YOURDOMAIN/js/videojs/video.js"></script>
    <script src="https://YOURDOMAIN/js/videojs/videojs-http-streaming.js"></script>
    <script>
        var player = videojs('live_stream');
    </script>
</center>
</body>
</html>
```

---

## 🎬 الاختبار

### إعدادات OBS Studio

| الخيار | القيمة |
|--------|--------|
| **Service** | Custom |
| **Server** | `rtmp://YOURDOMAIN/live` |
| **Stream Key** | `stream` |

ثم افتح المتصفح على:

```
https://YOURDOMAIN/abs.html
```

### لوحة الإحصائيات

لمراقبة البث المباشر:

```
https://YOURDOMAIN/stat
```

---

## 🔧 استكشاف الأخطاء

<details>
<summary><strong>البث لا يظهر في المتصفح</strong></summary>

تحقق من فتح المنفذ 1935:

```bash
sudo ufw allow 1935/tcp
sudo ufw reload
```

</details>

<details>
<summary><strong>خطأ في إعداد Nginx</strong></summary>

راجع سجل الأخطاء:

```bash
sudo tail -f /var/log/nginx/error.log
```

</details>

<details>
<summary><strong>استهلاك CPU مرتفع</strong></summary>

قلّل عدد الجودات في `nginx.conf`، أو استخدم `-preset ultrafast` بدلاً من `veryfast`.

</details>

<details>
<summary><strong>تأخير في البث</strong></summary>

قلّل قيمة `hls_fragment` إلى `2s` و `hls_playlist_length` إلى `10s`.

</details>

---

## 🤝 المساهمة

المساهمات مرحّب بها! اتبع الخطوات:

1. **Fork** المستودع
2. أنشئ فرعاً جديداً (`git checkout -b feature/AmazingFeature`)
3. التزم بتغييراتك (`git commit -m 'Add some AmazingFeature'`)
4. ادفع الفرع (`git push origin feature/AmazingFeature`)
5. افتح **Pull Request**

---

## 📚 مراجع مفيدة

- [توثيق nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module/wiki/Directives)
- [توثيق FFmpeg](https://ffmpeg.org/documentation.html)
- [Video.js Documentation](https://videojs.com/getting-started)
- [Let's Encrypt](https://letsencrypt.org/)

---

## 📄 الترخيص

هذا المشروع مرخّص تحت رخصة **MIT** - راجع ملف [LICENSE](LICENSE) للتفاصيل.

---

<div align="center">

**⭐ إذا أعجبك هذا الدليل، لا تنسَ إعطاءه نجمة ⭐**

صُنع بـ ❤️ للمجتمع العربي

</div>
#   n g i n x - r t m p - f f m p e g - u b u n t u - s s l  
 