# Section 5 — Data Model

---

## 5.1 Firebase Realtime Database — Cấu trúc tổng thể

Firebase Realtime Database là NoSQL JSON tree. Toàn bộ dữ liệu của ứng dụng được tổ chức dưới root node của project.

```json
{
  "users": {
    "<uid>": {
      "settings": { ... },
      "domains": { ... }
    }
  }
}
```

> Mọi data đều scoped theo `uid` (Firebase Auth User ID) để tránh người dùng đọc/ghi data của nhau.

---

## 5.2 Node: `/users/{uid}/settings`

Lưu cấu hình API credentials của người dùng.

```json
{
  "users": {
    "uid_abc123": {
      "settings": {
        "credentials": {
          "dpdns": {
            "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
            "verified": true,
            "verified_at": 1716192000000
          },
          "cloudflare": {
            "email": "user@example.com",
            "api_key": "c2547eb745079dac9320b638f5e225cf483cc5",
            "account_id": "01a7362d577a6c3019a474fd6f485823",
            "verified": true,
            "verified_at": 1716192000000
          }
        },
        "updated_at": 1716192000000
      }
    }
  }
}
```

### Field Definitions — Credentials

| Field | Type | Mô tả |
|-------|------|-------|
| `dpdns.token` | `string` | DigitalPlat Bearer JWT Token |
| `dpdns.verified` | `boolean` | Đã xác thực thành công qua API call? |
| `dpdns.verified_at` | `number` | Unix timestamp (ms) lần xác thực gần nhất |
| `cloudflare.email` | `string` | Cloudflare account email |
| `cloudflare.api_key` | `string` | Cloudflare Global API Key |
| `cloudflare.account_id` | `string` | Cloudflare Account ID (lấy từ API khi verify) |
| `cloudflare.verified` | `boolean` | Đã xác thực thành công? |
| `cloudflare.verified_at` | `number` | Unix timestamp (ms) |
| `settings.updated_at` | `number` | Lần cuối cập nhật settings |

---

## 5.3 Node: `/users/{uid}/domains`

Lưu danh sách domain đã đăng ký.

```json
{
  "users": {
    "uid_abc123": {
      "domains": {
        "-NxDomainKey001": {
          "name": "myapp",
          "namespace": ".dpdns.org",
          "fqdn": "myapp.dpdns.org",
          "cloudflare": {
            "zone_id": "023e105f4ecef8ad9ca31a8372d0c353",
            "nameservers": [
              "anna.ns.cloudflare.com",
              "bob.ns.cloudflare.com"
            ]
          },
          "dpdns": {
            "registered": true,
            "registration_response": "success"
          },
          "status": "active",
          "notes": "Domain cho project quản lý kho",
          "created_at": 1716192000000,
          "updated_at": 1716192000000
        }
      }
    }
  }
}
```

### Field Definitions — Domain Record

| Field | Type | Required | Mô tả |
|-------|------|----------|-------|
| `name` | `string` | ✅ | Subdomain name (không có namespace), e.g. `myapp` |
| `namespace` | `string` | ✅ | Namespace extension: `.dpdns.org`, `.us.kg`, `.qzz.io`, `.xx.kg` |
| `fqdn` | `string` | ✅ | Fully Qualified Domain Name: `myapp.dpdns.org` |
| `cloudflare.zone_id` | `string` | ✅ | Cloudflare Zone ID (32 hex chars) |
| `cloudflare.nameservers` | `string[]` | ✅ | Array 2 nameservers do Cloudflare cung cấp |
| `dpdns.registered` | `boolean` | ✅ | Đã đăng ký thành công trên DigitalPlat? |
| `dpdns.registration_response` | `string` | ❌ | Raw response hoặc status từ DPDNS API |
| `status` | `string` | ✅ | `active` \| `pending` \| `error` \| `deleted` |
| `notes` | `string` | ❌ | Ghi chú tùy chọn của người dùng |
| `created_at` | `number` | ✅ | Unix timestamp (ms) khi tạo |
| `updated_at` | `number` | ✅ | Unix timestamp (ms) lần sửa cuối |

---

## 5.4 Domain Status State Machine

```
          ┌─────────┐
          │ PENDING │  ← Domain đang trong quá trình đăng ký
          └────┬────┘
               │  Cloudflare OK + DPDNS OK
               ▼
          ┌────────┐
          │ ACTIVE │  ← Domain đã đăng ký thành công, NS đã được set
          └────┬───┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
  ┌─────────┐      ┌─────────┐
  │  ERROR  │      │ DELETED │  ← Đã xoá khỏi danh sách quản lý
  └─────────┘      └─────────┘
  ↑ (Rollback hoặc API thất bại)
```

---

## 5.5 Firebase Security Rules

```json
{
  "rules": {
    "users": {
      "$uid": {
        ".read": "$uid === auth.uid",
        ".write": "$uid === auth.uid",
        "settings": {
          "credentials": {
            ".read": "$uid === auth.uid",
            ".write": "$uid === auth.uid"
          }
        },
        "domains": {
          ".read": "$uid === auth.uid",
          ".write": "$uid === auth.uid",
          "$domainId": {
            ".validate": "newData.hasChildren(['name', 'namespace', 'fqdn', 'status', 'created_at'])"
          }
        }
      }
    }
  }
}
```

---

## 5.6 Entity Relationship Overview

```
┌──────────────┐       1     1 ┌──────────────────┐
│    User      │───────────────│    Settings      │
│  (Firebase   │               │  (credentials)   │
│   Auth UID)  │               └──────────────────┘
└──────┬───────┘
       │ 1
       │
       │ N
┌──────▼──────────────────────────────────────────┐
│                 Domain Record                    │
│  fqdn, namespace, status, created_at, notes...  │
│                                                  │
│  ┌───────────────────┐  ┌─────────────────────┐ │
│  │  Cloudflare Data  │  │    DPDNS Data        │ │
│  │  zone_id, NS[]    │  │  registered, resp    │ │
│  └───────────────────┘  └─────────────────────┘ │
└──────────────────────────────────────────────────┘
```

---

## 5.7 Indexing & Query Patterns

Firebase Realtime Database không hỗ trợ query phức tạp. Các query patterns dự kiến:

| Query | Firebase Path | Method |
|-------|--------------|--------|
| Lấy tất cả domain của user | `/users/{uid}/domains` | `onValue()` |
| Lấy credentials | `/users/{uid}/settings/credentials` | `get()` |
| Sửa 1 domain | `/users/{uid}/domains/{domainId}` | `update()` |
| Xoá 1 domain | `/users/{uid}/domains/{domainId}` | `remove()` |
| Sắp xếp theo ngày tạo | `/users/{uid}/domains` + `orderByChild('created_at')` | `query()` |

> **Index cần thêm vào Firebase rules** để sort theo `created_at`:
> ```json
> "domains": { ".indexOn": ["created_at", "status"] }
> ```
