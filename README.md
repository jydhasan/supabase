# Django + Supabase (PostgreSQL + Storage) প্রজেক্ট

Django প্রজেক্ট, যেখানে Supabase এর দুটো সার্ভিস ব্যবহার করা হয়েছে:

1. **Database**: Supabase PostgreSQL, **Session pooler** এর মাধ্যমে (ফ্রি প্ল্যানেও IPv4 তে কাজ করে)।
2. **File Storage**: Supabase Storage এর **Public bucket**, S3-compatible endpoint ও `django-storages` দিয়ে।

## সূচিপত্র

- [প্রয়োজনীয় জিনিস](#প্রয়োজনীয়-জিনিস)
- [দ্রুত সেটআপ (নতুন কম্পিউটারে)](#দ্রুত-সেটআপ-নতুন-কম্পিউটারে)
- [কনফিগারেশন ১: Database](#কনফিগারেশন-১-database-supabase-postgresql)
- [কনফিগারেশন ২: File Storage](#কনফিগারেশন-২-file-storage-supabase-bucket)
- [`.env` ফাইলের পূর্ণ নমুনা](#env-ফাইলের-পূর্ণ-নমুনা)
- [Troubleshooting](#troubleshooting)
- [Production এ deploy করার আগে](#production-এ-deploy-করার-আগে)
- [প্রজেক্টের গঠন](#প্রজেক্টের-গঠন)

## প্রয়োজনীয় জিনিস

- Python 3.12
- একটি Supabase প্রজেক্ট (Free plan হলেই চলবে)
- Supabase এ তৈরি করা একটি Public bucket (যেমন `media`)

## দ্রুত সেটআপ (নতুন কম্পিউটারে)

### ১. রিপোজিটরি ক্লোন

```
git clone <repo-url>
cd Supabase
```

### ২. Virtual environment (প্রস্তাবিত)

```
py -m venv venv
venv\Scripts\activate
```

### ৩. Package install

```
pip install -r requirements.txt
```

`requirements.txt` না থাকলে:

```
pip install django dj-database-url python-dotenv psycopg2-binary django-storages boto3
```

### ৪. `.env` ফাইল তৈরি

`manage.py` এর পাশে `.env` নামে ফাইল বানান। `.env.example` কপি করে নিতে পারেন:

```
copy .env.example .env
```

তারপর আসল মানগুলো বসান ([পূর্ণ নমুনা নিচে](#env-ফাইলের-পূর্ণ-নমুনা))।

### ৫. Migration ও admin user

```
py manage.py migrate
py manage.py createsuperuser
```

### ৬. সার্ভার চালু

```
py manage.py runserver
```

ব্রাউজারে `http://127.0.0.1:8000/` ও `http://127.0.0.1:8000/admin/` খুলুন।

---

## কনফিগারেশন ১: Database (Supabase PostgreSQL)

### Connection string কোথায় পাবেন

1. Supabase dashboard এ প্রজেক্ট খুলুন।
2. উপরে **Connect** বাটনে ক্লিক করুন।
3. Method থেকে **Session pooler** (Shared pooler) বেছে নিন।
4. Connection string কপি করুন। `[YOUR-PASSWORD]` এর জায়গায় আসল database password বসান।

| অংশ | মান |
|---|---|
| host | `aws-1-ap-southeast-2.pooler.supabase.com` (dashboard এ যা দেখাবে) |
| port | `5432` |
| database | `postgres` |
| user | `postgres.<PROJECT-REF>` |

### `.env` এ

```
DATABASE_URL=postgresql://postgres.<PROJECT-REF>:<PASSWORD>@<POOLER-HOST>:5432/postgres
```

### `settings.py`

ফাইলের একদম উপরে:

```python
import os
from pathlib import Path

import dj_database_url
import dotenv

BASE_DIR = Path(__file__).resolve().parent.parent
dotenv.load_dotenv(BASE_DIR / '.env')
```

Database ব্লক:

```python
DATABASES = {
    'default': dj_database_url.config(
        default=os.environ['DATABASE_URL'],
        conn_max_age=600,
        ssl_require=True,
    )
}
```

`os.environ['DATABASE_URL']` ব্যবহার করা হয়েছে যাতে `.env` ঠিক না থাকলে সাথে সাথে `KeyError` দেখায়, চুপচাপ ব্যর্থ না হয়।

### নিয়ম ও সতর্কতা (Database)

- Password এ `@`, `#`, `/`, `:`, `%` থাকলে percent-encode করতে হবে (যেমন `@` হবে `%40`)। সহজ উপায়: password এ শুধু ইংরেজি অক্ষর ও সংখ্যা রাখুন।
- `.env` এর ফরম্যাট `নাম=মান`। শুধু URL লিখলে কাজ করবে না, শুরুতে `DATABASE_URL=` লাগবে। `=` এর দুই পাশে স্পেস বা কোটেশন দেবেন না।
- Username এ অবশ্যই project ref থাকবে: `postgres.<PROJECT-REF>`।
- **"Upgrade to Pro" বা "Enable IPv4 add-on" এ ক্লিক করার দরকার নেই।** Session pooler ফ্রি প্ল্যানেই IPv4 তে চলে।
- Direct connection (`db.<ref>.supabase.co`) IPv6-only। আপনার নেটওয়ার্কে IPv6 না থাকলে `could not translate host name` error আসবে।
- Password কোথাও পেস্ট করে ফেললে (চ্যাট, issue, screenshot) সাথে সাথে **Reset database password** দিয়ে বদলে ফেলুন।

### Transaction pooler (port 6543) ব্যবহার করলে

`migrate` এর জন্য Session pooler (5432) ই ভালো। Transaction pooler ব্যবহার করলে:

```python
DATABASES = {
    'default': dj_database_url.config(
        default=os.environ['DATABASE_URL'],
        conn_max_age=0,
        ssl_require=True,
    )
}
DATABASES['default']['DISABLE_SERVER_SIDE_CURSORS'] = True
```

---

## কনফিগারেশন ২: File Storage (Supabase bucket)

Supabase Storage এর S3-compatible endpoint ব্যবহার করা হয়েছে, যাতে Django এর `FileField`/`ImageField` সরাসরি bucket এ আপলোড করে।

### ধাপ ১: Bucket তৈরি

1. Supabase dashboard এ **Storage** এ যান।
2. **New bucket** ক্লিক করুন, নাম দিন (এই প্রজেক্টে `media`)।
3. **Public bucket** চালু করুন।

### ধাপ ২: S3 access key তৈরি

1. **Storage** এর **S3 Configuration** পেজে যান।
2. **S3 protocol connection** টগল চালু করুন।
3. **Endpoint** ও **Region** কপি করুন:
   - Endpoint: `https://<PROJECT-REF>.storage.supabase.co/storage/v1/s3`
   - Region: `ap-southeast-2`
4. **New access key** ক্লিক করে **Access Key ID** ও **Secret Access Key** কপি করুন। **Secret শুধু একবার দেখায়**, তাই সাথে সাথে `.env` এ রাখুন।

### ধাপ ৩: Package

```
pip install django-storages boto3
```

### ধাপ ৪: `.env` এ

```
SUPABASE_S3_ENDPOINT=https://<PROJECT-REF>.storage.supabase.co/storage/v1/s3
SUPABASE_S3_REGION=ap-southeast-2
SUPABASE_S3_ACCESS_KEY=<ACCESS_KEY_ID>
SUPABASE_S3_SECRET_KEY=<SECRET_ACCESS_KEY>
SUPABASE_BUCKET=media
SUPABASE_PROJECT_REF=<PROJECT-REF>
```

### ধাপ ৫: `settings.py`

`INSTALLED_APPS` এ `'storages'` যোগ করুন:

```python
INSTALLED_APPS = [
    # ... আগের গুলো
    'storages',
]
```

ফাইলের নিচে:

```python
# Supabase Storage (S3-compatible, Public bucket)
SUPABASE_PROJECT_REF = os.environ['SUPABASE_PROJECT_REF']
SUPABASE_BUCKET = os.environ['SUPABASE_BUCKET']

STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3.S3Storage",
        "OPTIONS": {
            "access_key": os.environ['SUPABASE_S3_ACCESS_KEY'],
            "secret_key": os.environ['SUPABASE_S3_SECRET_KEY'],
            "bucket_name": SUPABASE_BUCKET,
            "endpoint_url": os.environ['SUPABASE_S3_ENDPOINT'],
            "region_name": os.environ['SUPABASE_S3_REGION'],
            "addressing_style": "path",
            "signature_version": "s3v4",
            "default_acl": None,
            "file_overwrite": False,      # একই নামের ফাইল ওভাররাইট হবে না
            "querystring_auth": False,    # Public bucket: signed URL লাগে না
            "custom_domain": f"{SUPABASE_PROJECT_REF}.supabase.co/storage/v1/object/public/{SUPABASE_BUCKET}",
        },
    },
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
    },
}
```

### ধাপ ৬: পরীক্ষা

```
py manage.py shell
```

```python
from django.core.files.base import ContentFile
from django.core.files.storage import default_storage

path = default_storage.save('test/hello.txt', ContentFile(b'hello supabase'))
print(default_storage.url(path))
```

প্রত্যাশিত আউটপুট:

```
https://<PROJECT-REF>.supabase.co/storage/v1/object/public/media/test/hello.txt
```

এই URL ব্রাউজারে খুললে `hello supabase` দেখা যাবে, আর Supabase এ bucket এর ভেতর `test/hello.txt` ফাইলটাও থাকবে। পরীক্ষা শেষে ফাইলটা dashboard থেকে মুছে দিতে পারেন।

### Model এ ব্যবহার

```python
from django.db import models

class Document(models.Model):
    title = models.CharField(max_length=200)
    file = models.FileField(upload_to='documents/')
```

`ImageField` ব্যবহার করলে `pip install Pillow` লাগবে। এরপর:

```
py manage.py makemigrations
py manage.py migrate
```

Admin বা ফর্ম থেকে ফাইল আপলোড করলে সরাসরি bucket এ যাবে। Template এ `{{ obj.file.url }}` দিলে public URL পাওয়া যাবে।

### নিয়ম ও সতর্কতা (Storage)

- **S3 access key পুরো প্রজেক্টের সব bucket এ অ্যাক্সেস দেয় এবং Storage এর RLS policy পাশ কাটিয়ে যায়।** এটা শুধু সার্ভারের `.env` এ থাকবে। ব্রাউজার, ফ্রন্টএন্ড কোড, git বা চ্যাটে কখনো নয়।
- **Public bucket এর যেকোনো ফাইলের URL যে কেউ খুলতে পারবে।** NID, রিসিপ্ট, ব্যক্তিগত ডকুমেন্ট এখানে রাখবেন না। সেগুলোর জন্য আলাদা **Private bucket** বানান।
- Private bucket ব্যবহার করতে চাইলে `STORAGES` এ `querystring_auth` ও `custom_domain` লাইন দুটো মুছে দিন। তখন `.url` কল করলে সময়সীমাওয়ালা signed URL পাওয়া যাবে (ডিফল্ট ১ ঘণ্টা)।
- Endpoint এ শেষের `/storage/v1/s3` অংশ রাখতে হবে, আর endpoint এ bucket এর নাম জুড়বেন না।
- ব্যবহারকারী সরাসরি ব্রাউজার থেকে Supabase এ আপলোড করবে, এমন ক্ষেত্রে এই পদ্ধতি নয়। তখন Supabase এর signed upload URL ও RLS policy ব্যবহার করতে হয়।
- Secret key চ্যাটে বা কোথাও প্রকাশ হয়ে গেলে dashboard থেকে সেই key মুছে নতুন key বানান।

---

## `.env` ফাইলের পূর্ণ নমুনা

এটা `.env.example` নামে git এ commit করা যাবে (আসল মান ছাড়া):

```
# Database
DATABASE_URL=postgresql://postgres.<PROJECT-REF>:<PASSWORD>@<POOLER-HOST>:5432/postgres

# Storage
SUPABASE_S3_ENDPOINT=https://<PROJECT-REF>.storage.supabase.co/storage/v1/s3
SUPABASE_S3_REGION=ap-southeast-2
SUPABASE_S3_ACCESS_KEY=<ACCESS_KEY_ID>
SUPABASE_S3_SECRET_KEY=<SECRET_ACCESS_KEY>
SUPABASE_BUCKET=media
SUPABASE_PROJECT_REF=<PROJECT-REF>
```

আসল `.env` ফাইল **কখনো git এ commit করবেন না**। `.gitignore` এ থাকতে হবে:

```
.env
venv/
__pycache__/
*.pyc
```

---

## Troubleshooting

### Database

| Error | কারণ | সমাধান |
|---|---|---|
| `could not translate host name "db.xxx.supabase.co"` | Direct connection IPv6-only, নেটওয়ার্কে IPv6 নেই | Session pooler string ব্যবহার করুন |
| `KeyError: 'DATABASE_URL'` | `.env` পড়া যাচ্ছে না, বা ফাইলে `DATABASE_URL=` লেখা নেই | `.env` `manage.py` এর পাশে আছে কিনা এবং `DATABASE_URL=...` ফরম্যাটে আছে কিনা দেখুন |
| `No DATABASE_URL environment variable set` | একই কারণ | একই সমাধান |
| `password authentication failed` | Password ভুল, বা special character encode করা হয়নি | Password reset করুন, encode করুন |
| `Tenant or user not found` | Username এ project ref বাদ পড়েছে | Username হবে `postgres.<PROJECT-REF>` |
| ফাইল আছে কিন্তু `.env` পড়ছে না | নাম আসলে `.env.txt`, অথবা ফাইলে BOM আছে | `dir /a .env*` দিয়ে নাম দেখুন; VS Code এ UTF-8 (BOM ছাড়া) হিসেবে save করুন |

`.env` ঠিকমতো পড়া হচ্ছে কিনা দেখতে (আউটপুট পেস্ট করার আগে secret মুছে দিন):

```
py -c "import dotenv; print(dotenv.dotenv_values('.env'))"
```

### Storage

| Error | কারণ | সমাধান |
|---|---|---|
| `KeyError: 'SUPABASE_...'` | `.env` এ কোনো variable বাদ পড়েছে | উপরের পূর্ণ নমুনার সাথে মিলিয়ে দেখুন |
| `NoSuchBucket` বা `404` | Bucket এর নাম ভুল (বড়-ছোট হাতের অক্ষরসহ হুবহু মিলতে হবে) | `SUPABASE_BUCKET` ঠিক করুন |
| `SignatureDoesNotMatch` বা `403` | Access Key ID বা Secret ভুল, বা কপির সময় বাড়তি স্পেস ঢুকেছে | Key আবার কপি করুন, দরকারে নতুন key বানান |
| আপলোড হয় কিন্তু URL ব্রাউজারে খোলে না | Bucket Public নয়, অথবা `custom_domain` এ ভুল | Bucket Public আছে কিনা এবং `custom_domain` ঠিক আছে কিনা দেখুন |
| Connection error / timeout | Endpoint ভুল বা শেষের `/storage/v1/s3` বাদ পড়েছে | dashboard থেকে endpoint আবার কপি করুন |
| `ModuleNotFoundError: storages` | `django-storages` install হয়নি, বা `INSTALLED_APPS` এ `'storages'` নেই | `pip install django-storages boto3` চালান ও `INSTALLED_APPS` দেখুন |

---

## Production এ deploy করার আগে

- `SECRET_KEY` কে `.env` এ সরান এবং নতুন লম্বা random key বানান:

  ```
  python -c "from django.core.management.utils import get_random_secret_key as g; print(g())"
  ```

- `DEBUG = False` করুন।
- `ALLOWED_HOSTS` এ আপনার domain দিন।
- Hosting প্ল্যাটফর্মের environment variables এ `.env` এর সব মান বসান (`.env` ফাইল সার্ভারে আপলোড না করাই ভালো)।
- Email এর জন্য `settings.py` তে `EMAIL_BACKEND` ব্যবহার করুন (Django তে `MAILERS` নামে কোনো setting নেই):

  ```python
  EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
  ```

---

## প্রজেক্টের গঠন

```
Supabase/
├── config/            # Django settings, urls, wsgi
├── manage.py
├── requirements.txt
├── .env               # গোপন, git এ যাবে না
├── .env.example       # নমুনা, git এ যাবে
├── .gitignore
└── README.md
```
