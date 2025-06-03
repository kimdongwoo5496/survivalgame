# 프로젝트: 맛집 추천 웹서비스 (Backend)

이 문서는 백엔드 서버에서 사용하는 **MariaDB** 데이터베이스 스키마를 설명합니다.  
세 가지 주요 테이블(`restaurant`, `menu`, `review`)의 구조 및 관계를 정리했으니, 처음 접하는 분도 쉽게 참고하실 수 있습니다.

---

## 📦 데이터베이스 스키마

### 1. 전체 구조 개요

- **데이터베이스 이름**: `ffs`  
- **사용자 및 권한**:  
  - 사용자: `ffs_user`  
  - 비밀번호: `YourSecurePassword`  
  - 권한: `ffs` 스키마 내 모든 테이블에 대해 `ALL PRIVILEGES`  
- **엔진**: InnoDB (기본), 문자셋은 `utf8mb4`, 콜레이션은 `utf8mb4_unicode_ci` 권장  
- **테이블 목록**:  
  1. `restaurant` (음식점 기본 정보)  
  2. `menu` (각 음식점 메뉴 정보)  
  3. `review` (각 음식점에 대한 사용자 리뷰)

---

### 2. `restaurant` 테이블

> 음식점의 기본 정보를 저장합니다.

```sql
CREATE TABLE IF NOT EXISTS restaurant (
  id INT PRIMARY KEY AUTO_INCREMENT,                      -- 음식점 고유 ID (자동 증가)
  name VARCHAR(100) NOT NULL,                             -- 음식점 이름
  category VARCHAR(50) NOT NULL,                          -- 음식점 카테고리 (예: 한식, 중식, 양식 등)
  description TEXT,                                       -- 음식점 설명 (필요 시 사용)
  address VARCHAR(255) NOT NULL,                          -- 도로명 주소 또는 지번 주소
  phone VARCHAR(30) NOT NULL,                             -- 전화번호
  open_time VARCHAR(100) NOT NULL,                        -- 영업시간 (예: 11:00~21:30)
  main_image_url VARCHAR(255),                            -- 대표 이미지 URL (없으면 NULL)
  kakao_id VARCHAR(50) NOT NULL UNIQUE,                   -- Kakao 고유 장소 ID (중복 방지 키)
  rating DECIMAL(2,1) DEFAULT 0.0,                        -- 평균 별점 정보 (예: 3.5)
  location_tag ENUM('정문', '후문', '기타') DEFAULT '기타', -- 음식점 위치 태그 (정문/후문/기타)
  is_new BOOLEAN DEFAULT FALSE                            -- 신규 개장 여부 (TRUE/FALSE)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

- **id**: 각 음식점을 구분하는 **자동 증가(PK)**  
- **name**: 음식점의 이름  
- **category**: 음식점 카테고리 (예: 한식, 중식, 양식, 일식 등)  
- **description**: 설명(필요하다면 사용)  
- **address**: 음식점 주소  
- **phone**: 전화번호  
- **open_time**: 영업시간 표시 문자열  
- **main_image_url**: 대표 이미지 URL (없으면 `NULL`)  
- **kakao_id**: 카카오맵 고유 장소 ID (중복 방지를 위해 `UNIQUE` 제약)  
- **rating**: 평점(소수 첫째 자리까지)  
- **location_tag**: 음식점 위치 태그 (정문 / 후문 / 기타)  
- **is_new**: 신규 개장 여부 (기본값: `FALSE`)  

---

### 3. `menu` 테이블

> 각 음식점이 제공하는 메뉴 정보를 저장합니다.  
> `restaurant` 테이블의 `id`를 외래키(FK)로 참조하여, 해당 음식점에 속하는 메뉴를 관리합니다.

```sql
CREATE TABLE IF NOT EXISTS menu (
  id INT PRIMARY KEY AUTO_INCREMENT,              -- 메뉴 고유 ID (자동 증가)
  restaurant_id INT NOT NULL,                     -- 참조할 음식점 ID (FK)
  name VARCHAR(100) NOT NULL,                     -- 메뉴 이름
  price INT NOT NULL,                              -- 가격 (원 단위)
  description TEXT,                                -- 메뉴 설명 (옵션)
  image_url VARCHAR(255),                          -- 메뉴 이미지 URL (옵션)
  FOREIGN KEY (restaurant_id)
    REFERENCES restaurant(id)
    ON DELETE CASCADE                              -- 음식점 삭제 시 해당 메뉴도 함께 삭제
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

- **id**: 메뉴 고유 ID (자동 증가)  
- **restaurant_id**: `restaurant(id)`를 참조하는 **외래키**  
- **name**: 메뉴 이름  
- **price**: 메뉴 가격 (원 단위)  
- **description**: 메뉴에 대한 추가 설명  
- **image_url**: 메뉴 이미지 URL (옵션)  
- 테이블 삭제 or 레코드 삭제 시, 관련 참조 무결성을 위해 `ON DELETE CASCADE` 옵션 적용  

---

### 4. `review` 테이블

> 사용자 리뷰(별점 + 코멘트)를 저장합니다.  
> 각 리뷰는 `restaurant` 테이블의 `id`를 외래키로 참조하며,  
> 음식점이 삭제되면 해당 음식점의 모든 리뷰도 함께 삭제됩니다.

```sql
CREATE TABLE IF NOT EXISTS review (
  id INT PRIMARY KEY AUTO_INCREMENT,                -- 리뷰 고유 ID (자동 증가)
  restaurant_id INT NOT NULL,                       -- 참조할 음식점 ID (FK)
  rating DECIMAL(2,1) DEFAULT 0.0,                  -- 별점 (1.0 ~ 5.0), 소수 첫째 자리까지
  comment TEXT NOT NULL,                            -- 리뷰 내용
  created_at DATE NOT NULL,                         -- 리뷰 작성일 (YYYY-MM-DD)
  FOREIGN KEY (restaurant_id)
    REFERENCES restaurant(id)
    ON DELETE CASCADE,                              -- 음식점 삭제 시 리뷰도 함께 삭제
  UNIQUE KEY uk_review (restaurant_id, comment, created_at)
    -- 같은 음식점에 동일한 날짜, 동일한 내용(comment)이 중복 저장되는 것을 방지
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

- **id**: 리뷰 고유 ID (자동 증가)  
- **restaurant_id**: `restaurant(id)`를 참조하는 외래키  
- **rating**: 사용자 평점 (소수 첫째 자리까지)  
- **comment**: 리뷰 텍스트(필수)  
- **created_at**: 리뷰 작성일 (`DATE` 타입)  
- **UNIQUE KEY**: `(restaurant_id, comment, created_at)` 조합으로 중복 리뷰 방지  
- `ON DELETE CASCADE`를 걸어 놓아, 음식점이 삭제되면 해당 음식점의 리뷰도 자동 삭제  

---

## 🔗 테이블 관계 다이어그램 (ERD)

```
┌──────────────────────────┐          ┌──────────────────────────┐
│       restaurant        │1        *│          menu            │
│──────────────────────────│─────────▶│──────────────────────────│
│ PK  id   (INT, AI)      │          │ PK  id           (INT, AI)│
│     name (VARCHAR(100)) │          │     restaurant_id (INT)   │
│     category (VARCHAR50)│          │     name (VARCHAR(100))   │
│     description (TEXT)  │          │     price (INT)           │
│     address (VARCHAR255)│          │     description (TEXT)    │
│     phone (VARCHAR30)   │          │     image_url (VARCHAR255)│
│     open_time (VARCHAR100)│        │   FK→restaurant(id)       │
│     main_image_url(TEXT)│          └──────────────────────────┘
│     kakao_id (VARCHAR50)│
│     rating (DECIMAL)    │          ┌──────────────────────────┐
│     location_tag (ENUM) │ 1      * │         review           │
│     is_new (BOOLEAN)    │─────────▶│──────────────────────────│
└──────────────────────────┘          │ PK  id           (INT, AI)│
                                       │     restaurant_id (INT)   │
                                       │     rating       (DECIMAL)│
                                       │     comment      (TEXT)   │
                                       │     created_at   (DATE)   │
                                       │ FK→restaurant(id)         │
                                       │ UNIQUE (restaurant_id,     │
                                       │        comment, created_at)│
                                       └──────────────────────────┘
```

- **1 : N 관계 (restaurant ↔ menu)**  
  - 한 개의 음식점(`restaurant`)에는 여러 개의 메뉴(`menu`)가 속함  
- **1 : N 관계 (restaurant ↔ review)**  
  - 한 개의 음식점에는 여러 개의 리뷰(`review`)가 속함  

---

## 🚀 스키마 적용 및 초기 데이터 삽입

1. 위에 작성된 세 개의 `CREATE TABLE` 문을 **순서대로** 실행하세요.  
   (참고: `restaurant` → `menu` → `review` 순서)  
2. 참고용 초기 샘플 데이터를 삽입하고 싶다면, 예시 SQL을 추가로 작성합니다.  
   ```sql
   INSERT INTO restaurant
     (name, category, address, phone, open_time, kakao_id, rating, location_tag, is_new)
   VALUES
     ('세종대 스테이크하우스', '양식', '세종로 123', '02-1234-5678', '11:00~22:00', 'kakao12345', 4.7, '정문', TRUE);

   INSERT INTO menu
     (restaurant_id, name, price, description, image_url)
   VALUES
     (1, '안심 스테이크', 25000, '부드러운 레어 스테이크', 'https://example.com/steak.jpg'),
     (1, '갈릭 새우 파스타', 15000, '마늘과 새우가 어우러진 파스타', NULL);

   INSERT INTO review
     (restaurant_id, rating, comment, created_at)
   VALUES
     (1, 5.0, '최고의 스테이크였습니다!', '2025-06-01'),
     (1, 4.5, '분위기도 좋고, 맛있었어요.', '2025-06-02');
   ```

---

## 📝 요약

- **`restaurant`**: 음식점 기본 정보  
- **`menu`**: 음식점별 메뉴 정보 (`restaurant.id` → `menu.restaurant_id`, `ON DELETE CASCADE`)  
- **`review`**: 음식점별 리뷰 정보 (`restaurant.id` → `review.restaurant_id`, `ON DELETE CASCADE`)  
- **Character Set**: `utf8mb4` (한글, 이모지 지원)  
- **참고**: `UNIQUE` 제약과 `FOREIGN KEY ... ON DELETE CASCADE` 옵션으로  
  데이터 무결성을 유지하며, 중복 및 참조 오류를 방지합니다.  

이 스키마를 기준으로 백엔드 API 코드를 작성하면, 데이터 저장 및 조회가 원활하게 이루어집니다.  
필요에 따라 컬럼을 추가/수정하거나, 인덱스를 추가하여 성능을 최적화해 보세요!  
